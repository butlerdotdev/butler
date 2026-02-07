# Getting Started with Butler

This guide walks you through installing Butler and creating your first tenant cluster.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Install CLI](#install-cli)
- [Bootstrap Management Cluster](#bootstrap-management-cluster)
- [Create Your First Cluster](#create-your-first-cluster)
- [Next Steps](#next-steps)

---

## Prerequisites

### Required

| Requirement | Description |
|-------------|-------------|
| Docker | For running the bootstrap process |
| kubectl | Kubernetes CLI (1.28+) |
| Infrastructure | Harvester, Nutanix, or other supported provider |
| Network | IP addresses for control plane and LoadBalancer |

### Infrastructure-Specific

**For Harvester:**
- Harvester cluster with API access
- Harvester kubeconfig file
- VM network configured
- Talos Linux image uploaded

**For Nutanix:**
- Prism Central access
- Subnet and cluster configuration
- Cloud image uploaded

See [Provider Guides](../providers/) for detailed setup.

---

## Install CLI

### macOS / Linux (Homebrew)

```bash
brew install butlerdotdev/tap/butler
```

### macOS / Linux (Direct Download)

```bash
# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
[[ "$ARCH" == "x86_64" ]] && ARCH="amd64"
[[ "$ARCH" == "aarch64" ]] && ARCH="arm64"

# Download latest release
curl -sLO "https://github.com/butlerdotdev/butler-cli/releases/latest/download/butler_${OS}_${ARCH}.tar.gz"
tar xzf butler_${OS}_${ARCH}.tar.gz

# Install
sudo mv butleradm butlerctl /usr/local/bin/
```

### Windows (Chocolatey)

```powershell
choco install butler
```

### Verify Installation

```bash
butleradm version
butlerctl version
```

---

## Bootstrap Management Cluster

### 1. Create Bootstrap Configuration

Create a file named `bootstrap.yaml`:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: butler-mgmt
  namespace: butler-system
spec:
  provider: harvester  # or nutanix
  
  cluster:
    name: butler-mgmt
    controlPlaneEndpoint: 10.40.0.200  # Your VIP
    kubernetesVersion: "v1.31.0"
    topology: ha  # ha or single-node
    
    controlPlane:
      replicas: 3
      machineTemplate:
        cpu: 4
        memory: 16Gi
        diskSize: 100Gi

    workers:
      replicas: 3
      machineTemplate:
        cpu: 8
        memory: 32Gi
        diskSize: 200Gi

  networking:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    
  platform:
    cni: cilium
    storage: longhorn
    loadBalancer: metallb
    metalLBPool: 10.40.0.200-10.40.0.250
    
  providerConfig:
    harvester:
      credentialsRef:
        name: harvester-kubeconfig
      namespace: default
      networkName: default/workloads
      imageName: default/talos-1.9
```

### 2. Prepare Provider Credentials

**For Harvester:**

```bash
# Create credentials secret file
kubectl create secret generic harvester-kubeconfig \
  --from-file=kubeconfig=/path/to/harvester-kubeconfig.yaml \
  --dry-run=client -o yaml > harvester-secret.yaml
```

### 3. Run Bootstrap

```bash
butleradm bootstrap harvester \
  --config bootstrap.yaml \
  --credentials harvester-secret.yaml
```

This will:
1. Create a temporary KIND cluster
2. Deploy Butler controllers
3. Provision management cluster VMs
4. Install platform components
5. Save kubeconfig and clean up

**Expected duration:** 15-30 minutes depending on infrastructure.

### 4. Verify Installation

```bash
# Set kubeconfig
export KUBECONFIG=~/.butler/butler-mgmt-kubeconfig

# Check nodes
kubectl get nodes

# Check Butler components
kubectl get pods -n butler-system
```

---

## Create Your First Cluster

### 1. Create a Team (Optional)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: dev-team
spec:
  displayName: "Development Team"
  access:
    users:
      - name: you@example.com
        role: admin
EOF
```

### 2. Create a Tenant Cluster

Using CLI:

```bash
butlerctl cluster create my-first-cluster \
  --workers 2 \
  --k8s-version v1.30.0 \
  --namespace dev-team
```

Or using YAML:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: my-first-cluster
  namespace: dev-team
spec:
  kubernetesVersion: "v1.30.0"
  controlPlane:
    type: hosted
  workers:
    replicas: 2
  providerConfigRef:
    name: default
  addons:
    cni:
      provider: cilium
    loadBalancer:
      provider: metallb
EOF
```

### 3. Watch Progress

```bash
butlerctl cluster get my-first-cluster -w
```

Or:

```bash
kubectl get tenantcluster my-first-cluster -n dev-team -w
```

### 4. Get Kubeconfig

```bash
butlerctl cluster kubeconfig my-first-cluster > my-first-cluster.yaml
```

### 5. Use Your Cluster

```bash
export KUBECONFIG=my-first-cluster.yaml

kubectl get nodes
kubectl get pods -A
```

---

## Next Steps

### Install Console

Access Butler via a web UI:

```bash
helm install butler-console oci://ghcr.io/butlerdotdev/charts/butler-console \
  -n butler-system \
  --set ingress.enabled=true \
  --set ingress.host=butler.example.com
```

### Enable GitOps

Manage clusters declaratively via Git:

```bash
# Install Flux (if not already installed)
flux bootstrap github \
  --owner=myorg \
  --repository=butler-clusters \
  --path=clusters/butler-mgmt
```

### Add Monitoring

Install observability stack:

```bash
butlerctl addon install prometheus --cluster my-first-cluster
```

### Read More

- [Architecture](../architecture/) - Understand how Butler works
- [Operations Guide](../operations/) - Day-2 operations
- [Provider Guides](../providers/) - Infrastructure-specific setup

---

## Troubleshooting

### Bootstrap Fails

```bash
# Check bootstrap logs
butleradm bootstrap harvester --config bootstrap.yaml --verbose

# Keep KIND cluster for debugging
butleradm bootstrap harvester --config bootstrap.yaml --keep-kind
kubectl --context kind-butler-bootstrap logs -n butler-system deploy/butler-bootstrap-controller
```

### Cluster Stuck in Provisioning

```bash
# Check TenantCluster status
kubectl describe tenantcluster my-first-cluster -n dev-team

# Check CAPI resources
kubectl get cluster,machinedeployment,machine -A

# Check Steward control plane
kubectl get tenantcontrolplane -A
```

### Worker Nodes Not Joining

```bash
# Check MachineDeployment
kubectl describe machinedeployment my-first-cluster-workers -n dev-team

# Check Machine status
kubectl get machine -l cluster.x-k8s.io/cluster-name=my-first-cluster -A
```

### Get Help

- [GitHub Issues](https://github.com/butlerdotdev/butler/issues)
- [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions)
