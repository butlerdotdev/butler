# Getting Started with Butler

This guide walks you through installing Butler and creating your first management cluster.

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
| Docker | For running the temporary KIND bootstrap cluster |
| kubectl | Kubernetes CLI (1.28+) |
| Infrastructure | Harvester, Nutanix, AWS, GCP, or Azure account |
| Network | IP addresses for control plane VIP and LoadBalancer pool (on-prem only) |

### Infrastructure-Specific

**For On-Prem (Harvester, Nutanix):**
- Infrastructure API access (Harvester kubeconfig or Prism Central credentials)
- VM network configured with DHCP or static IPs
- Talos Linux image uploaded to the infrastructure
- A VIP address reserved for the control plane endpoint
- An IP range for MetalLB LoadBalancer services

**For Cloud (AWS, GCP, Azure):**
- Cloud account with compute and networking permissions
- VPC/VNet with a subnet and firewall/security group rules
- Talos Linux image available in the target region (AMI, GCE image, or Azure gallery image)

See [Provider Guides](../providers/) for detailed per-provider setup.

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
choco install butler-cli
```

### Windows (Direct Download)

```powershell
$arch = if ($env:PROCESSOR_ARCHITECTURE -eq "ARM64") { "arm64" } else { "amd64" }
Invoke-WebRequest -Uri "https://github.com/butlerdotdev/butler-cli/releases/latest/download/butler_windows_${arch}.tar.gz" -OutFile butler.tar.gz
tar xzf butler.tar.gz
Move-Item butleradm.exe, butlerctl.exe -Destination "$env:LOCALAPPDATA\Microsoft\WindowsApps\"
```

### Verify Installation

```bash
butleradm version
butlerctl version
```

---

## Bootstrap Management Cluster

### 1. Create Bootstrap Configuration

Create a YAML config file for `butleradm`. This example uses Harvester as the infrastructure provider. See [Provider Guides](../providers/) for AWS, GCP, Azure, and Nutanix configs.

```yaml
provider: harvester

cluster:
  name: butler-mgmt
  topology: ha              # "ha" (3 CP + workers) or "single-node" (1 node)
  controlPlane:
    replicas: 3
    cpu: 4
    memoryMB: 8192
    diskGB: 50
  workers:
    replicas: 2
    cpu: 4
    memoryMB: 8192
    diskGB: 50
    extraDisks:
      - sizeGB: 50           # Additional disk for Longhorn storage

network:
  podCIDR: 10.244.0.0/16
  serviceCIDR: 10.96.0.0/12
  vip: 10.40.0.200           # Control plane VIP (on-prem only)
  loadBalancerPool:           # MetalLB IP range (on-prem only)
    start: 10.40.0.210
    end: 10.40.0.220

talos:
  version: v1.12.1
  schematic: dc7b152cb3ea99b821fcb7340ce7168313ce393d663740b791c36f6e95fc8586

addons:
  cni:
    type: cilium
  storage:
    type: longhorn
  loadBalancer:
    type: metallb
  console:
    enabled: true
    ingress:
      enabled: true
      className: traefik

providerConfig:
  harvester:
    kubeconfigPath: ~/.butler/harvester-kubeconfig
    namespace: default
    networkName: default/workloads
    imageName: default/talos-image
```

Save this as `~/.butler/bootstrap.yaml`.

For a full list of every config field and its defaults, see the [Bootstrap Config Reference](../reference/bootstrap-config.md).

### 2. Run Bootstrap

```bash
butleradm bootstrap harvester --config ~/.butler/bootstrap.yaml
```

Available flags:

| Flag | Description |
|------|-------------|
| `--config <path>` | Path to the bootstrap config YAML (required) |
| `--no-tui` | Disable interactive TUI, use line-by-line log output |
| `--skip-cleanup` | Keep the KIND cluster after bootstrap for debugging |
| `--local` | Build controller images from local source (development) |
| `--provider-image <image>` | Override the provider controller container image |

This will:
1. Create a temporary KIND cluster on your machine
2. Deploy Butler CRDs, bootstrap controller, and provider controller into KIND
3. Provision VMs on your infrastructure
4. Generate and apply Talos Linux configs to each node
5. Bootstrap Kubernetes on the first control plane node
6. Install addons (Cilium, cert-manager, Longhorn, MetalLB, Steward, Butler, Console)
7. Save kubeconfig and talosconfig to `~/.butler/`
8. Delete the KIND cluster (unless `--skip-cleanup`)

**Expected duration:** 15-30 minutes depending on infrastructure and network speed.

### 3. Verify Installation

```bash
# Set kubeconfig to the new management cluster
export KUBECONFIG=~/.butler/butler-mgmt-kubeconfig

# Check nodes
kubectl get nodes

# Check Butler components
kubectl get pods -n butler-system

# Check platform addons
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
kubectl get pods -n longhorn-system
kubectl get pods -n cert-manager
kubectl get pods -n steward-system
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
- [Bootstrap Config Reference](../reference/bootstrap-config.md) - Every config field documented

---

## Troubleshooting

### Bootstrap Fails

```bash
# Use --skip-cleanup to keep KIND cluster for debugging
butleradm bootstrap harvester --config bootstrap.yaml --skip-cleanup

# Then inspect the bootstrap state from the KIND context:
kubectl --context kind-butler-bootstrap get clusterbootstrap -n butler-system
kubectl --context kind-butler-bootstrap get machinerequest -n butler-system
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
