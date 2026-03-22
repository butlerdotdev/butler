# Getting Started with Butler

Butler is a Kubernetes-as-a-Service platform. It provisions and manages tenant Kubernetes clusters across on-prem and cloud infrastructure from a single management cluster.

This page covers installing the CLI, bootstrapping a management cluster, and creating your first tenant cluster.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Install CLI](#install-cli)
- [Bootstrap a Management Cluster](#bootstrap-a-management-cluster)
- [Create Your First Tenant Cluster](#create-your-first-tenant-cluster)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Requirement | Description |
|-------------|-------------|
| Docker | Runs the temporary KIND bootstrap cluster |
| kubectl | Kubernetes CLI (1.28+) |
| Infrastructure | One of: Harvester, Nutanix, AWS, GCP, or Azure |

Each infrastructure provider has additional prerequisites (credentials, networking, VM images). Follow the provider guide for your infrastructure **before** running bootstrap:

| Provider | Guide |
|----------|-------|
| Harvester | [Harvester Provider Guide](../providers/harvester.md) |
| Nutanix | [Nutanix Provider Guide](../providers/nutanix.md) |
| AWS | [AWS Provider Guide](../providers/aws.md) |
| GCP | [GCP Provider Guide](../providers/gcp.md) |
| Azure | [Azure Provider Guide](../providers/azure.md) |

Each provider guide walks through prerequisites, infrastructure setup, config creation, running bootstrap, and validation. Once your management cluster is running, come back here to create your first tenant cluster.

---

## Install CLI

Butler ships two CLI binaries:
- `butleradm` -- platform administration (bootstrap, provider management)
- `butlerctl` -- tenant operations (create clusters, get kubeconfigs)

### macOS / Linux (Homebrew)

```bash
brew install butlerdotdev/tap/butler
```

### macOS / Linux (Direct Download)

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
[[ "$ARCH" == "x86_64" ]] && ARCH="amd64"
[[ "$ARCH" == "aarch64" ]] && ARCH="arm64"

curl -sLO "https://github.com/butlerdotdev/butler-cli/releases/latest/download/butler_${OS}_${ARCH}.tar.gz"
tar xzf butler_${OS}_${ARCH}.tar.gz
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

## Bootstrap a Management Cluster

Bootstrap creates a management cluster on your infrastructure. The process takes 15-30 minutes and produces:

- A Kubernetes cluster running Talos Linux
- Cilium CNI, Longhorn storage, cert-manager
- Steward for hosted tenant control planes
- Butler CRDs, controller, and console
- On-prem: kube-vip (control plane HA) + MetalLB + Traefik (ingress)
- Cloud: Cloud Controller Manager + cloud load balancers

### 1. Follow Your Provider Guide

Each provider guide includes the full config file, infrastructure setup steps, and the bootstrap command. Pick your provider:

- [Harvester](../providers/harvester.md) -- on-prem, Harvester HCI
- [Nutanix](../providers/nutanix.md) -- on-prem, Nutanix AHV
- [AWS](../providers/aws.md) -- cloud, Amazon Web Services
- [GCP](../providers/gcp.md) -- cloud, Google Cloud Platform
- [Azure](../providers/azure.md) -- cloud, Microsoft Azure

### 2. What Bootstrap Does

```
butleradm bootstrap <provider> --config ~/.butler/bootstrap-<provider>.yaml
```

1. Creates a temporary KIND cluster on your machine
2. Deploys Butler bootstrap controller and provider controller into KIND
3. Provisions VMs on your infrastructure
4. Generates and applies Talos Linux configs to each node
5. Bootstraps Kubernetes on the first control plane node
6. Installs platform addons (Cilium, cert-manager, Longhorn, Steward, Butler, Console)
7. Saves kubeconfig to `~/.butler/<cluster-name>-kubeconfig`
8. Deletes the KIND cluster

Available flags:

| Flag | Description |
|------|-------------|
| `--config <path>` | Path to bootstrap config YAML (required) |
| `--no-tui` | Disable interactive TUI, use line-by-line log output |
| `--skip-cleanup` | Keep the KIND cluster after bootstrap for debugging |
| `--local` | Build controller images from local source (development) |
| `--provider-image <image>` | Override the provider controller container image |

### 3. After Bootstrap

The kubeconfig is saved as `~/.butler/<cluster-name>-kubeconfig`. The cluster name comes from `cluster.name` in your config file.

```bash
# Example: if cluster.name is "butler-mgmt"
export KUBECONFIG=~/.butler/butler-mgmt-kubeconfig

kubectl get nodes
kubectl get pods -n butler-system
```

The provider guide for your infrastructure includes a full validation checklist. Complete that before continuing.

---

## Create Your First Tenant Cluster

After your management cluster is running and validated, create a tenant cluster.

### 1. Create a Team

Teams are the multi-tenancy boundary in Butler. Each team gets its own namespace.

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

Using the CLI:

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

- [Architecture](../architecture/) -- how Butler works internally
- [Operations Guide](../operations/) -- day-2 operations
- [Bootstrap Config Reference](../reference/bootstrap-config.md) -- every config field documented
- [Bootstrap Flow](../architecture/bootstrap-flow.md) -- detailed bootstrap sequence

---

## Troubleshooting

### Bootstrap Fails

```bash
# Re-run with --skip-cleanup to keep KIND cluster for debugging
butleradm bootstrap <provider> --config <path> --skip-cleanup

# Inspect bootstrap state from the KIND context:
kubectl --context kind-butler-bootstrap get clusterbootstrap -n butler-system
kubectl --context kind-butler-bootstrap get machinerequest -n butler-system
kubectl --context kind-butler-bootstrap logs -n butler-system deploy/butler-bootstrap-controller
```

### Cluster Stuck in Provisioning

```bash
kubectl describe tenantcluster my-first-cluster -n dev-team
kubectl get cluster,machinedeployment,machine -A
kubectl get tenantcontrolplane -A
```

### Worker Nodes Not Joining

```bash
kubectl describe machinedeployment my-first-cluster-workers -n dev-team
kubectl get machine -l cluster.x-k8s.io/cluster-name=my-first-cluster -A
```

### Get Help

- [GitHub Issues](https://github.com/butlerdotdev/butler/issues)
- [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions)
