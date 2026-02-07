# Harvester Provider Guide

This guide covers setting up Butler with [Harvester HCI](https://harvesterhci.io/) as the infrastructure provider.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Harvester Setup](#harvester-setup)
- [Butler Configuration](#butler-configuration)
- [Network Configuration](#network-configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

Butler uses the [CAPI KubeVirt provider](https://github.com/kubernetes-sigs/cluster-api-provider-kubevirt) (capk) to provision VMs on Harvester, which runs KubeVirt under the hood.

```mermaid
flowchart LR
    Butler["Butler Controller"] --> CAPI["Cluster API"]
    CAPI --> CAPK["KubeVirt Provider"]
    CAPK --> Harvester["Harvester HCI"]
    Harvester --> VMs["Virtual Machines"]
```

### Key Components

| Component | Purpose |
|-----------|---------|
| butler-provider-harvester | Translates MachineRequests to Harvester VMs |
| CAPI KubeVirt Provider | Creates KubevirtMachine resources |
| Harvester | Runs KubeVirt and manages VM lifecycle |

---

## Prerequisites

### Harvester Cluster

- Harvester version 1.3.0 or later
- Admin access to Harvester dashboard
- Network connectivity from Butler management cluster to Harvester API

### Required Resources

| Resource | Purpose |
|----------|---------|
| VM Network | Network for cluster nodes (VLAN-backed recommended) |
| VM Image | OS image (Talos Linux for management, Rocky Linux for workers) |
| SSH Key | For node access (Rocky Linux workers) |
| Kubeconfig | Harvester API access |

---

## Harvester Setup

### 1. Create VM Network

In Harvester Dashboard -> Networks -> Create:

```yaml
# Example VLAN network
Name: workloads
Namespace: default
VLAN ID: 40
Cluster Network: mgmt
```

**Result:** `default/workloads` network name

### 2. Upload VM Images

#### Talos Linux (Management Cluster)

Download the KubeVirt image from [Talos releases](https://github.com/siderolabs/talos/releases):

```bash
# Download Talos KubeVirt image
curl -LO https://github.com/siderolabs/talos/releases/download/v1.9.0/nocloud-amd64.raw.xz
xz -d nocloud-amd64.raw.xz
```

In Harvester Dashboard -> Images -> Create:
- Name: `talos-1.9`
- URL or upload the raw file
- Namespace: `default`

**Result:** `default/talos-1.9` image name

#### Rocky Linux (Tenant Workers)

Download Rocky Linux cloud image:

```bash
curl -LO https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud.latest.x86_64.qcow2
```

Upload to Harvester:
- Name: `rocky-9-cloud`
- Namespace: `default`

**Result:** `default/rocky-9-cloud` image name

### 3. Create SSH Key

In Harvester Dashboard -> SSH Keys -> Create:

```bash
# Generate key pair (if needed)
ssh-keygen -t ed25519 -f ~/.ssh/butler-ssh -N ""
```

Upload public key:
- Name: `butler-ssh`
- Namespace: `default`

**Result:** `default/butler-ssh` SSH key name

### 4. Export Kubeconfig

In Harvester Dashboard -> Support -> Download Kubeconfig

Or via CLI:
```bash
# From Harvester node
cat /etc/rancher/rke2/rke2.yaml
```

Save as `harvester-kubeconfig.yaml` and update the server URL to the external address.

---

## Butler Configuration

### 1. Create ProviderConfig Secret

```bash
kubectl create secret generic harvester-kubeconfig \
  --from-file=kubeconfig=harvester-kubeconfig.yaml \
  -n butler-system
```

### 2. Create ProviderConfig Resource

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: harvester-prod
  namespace: butler-system
spec:
  type: harvester
  harvester:
    credentialsRef:
      name: harvester-kubeconfig
      key: kubeconfig
    namespace: default
    networkName: default/workloads
    imageName: default/talos-1.9
    storageClassName: harvester-longhorn
```

### 3. Set as Default Provider

Update ButlerConfig:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ButlerConfig
metadata:
  name: butler
spec:
  defaultProviderConfigRef:
    name: harvester-prod
```

---

## Network Configuration

### Recommended VLAN Layout

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 20 | Management | 10.20.0.0/16 | Harvester management, infrastructure |
| 30 | Storage | 10.30.0.0/16 | Storage traffic (iSCSI, NFS) |
| 40 | Workloads | 10.40.0.0/16 | Kubernetes clusters |
| 50 | External | 10.50.0.0/16 | External-facing services |

### MetalLB Integration

For tenant clusters to get LoadBalancer IPs, allocate pools from the workload VLAN:

```yaml
# In TenantCluster spec
spec:
  addons:
    loadBalancer:
      provider: metallb
      addressPool: "10.40.1.0-10.40.1.50"
```

### Control Plane Endpoint

For management cluster:
- Reserve a VIP (e.g., `10.40.0.200`) 
- kube-vip provides HA for control plane
- MetalLB provides LoadBalancer for services

For tenant clusters:
- Steward creates LoadBalancer service per cluster
- MetalLB assigns IPs from management cluster pool

---

## Resource Specifications

### Management Cluster Recommendations

| Role | CPU | Memory | Disk | Count |
|------|-----|--------|------|-------|
| Control Plane | 4 | 16Gi | 100Gi | 3 (HA) or 1 (single-node) |
| Worker | 8 | 32Gi | 200Gi | 0-3 |

### Tenant Cluster Defaults

| Role | CPU | Memory | Disk |
|------|-----|--------|------|
| Worker | 4 | 8Gi | 40Gi |

Override in TenantCluster spec:

```yaml
spec:
  workers:
    replicas: 3
    machineTemplate:
      cpu: 8
      memory: 16Gi
      diskSize: 100Gi
```

---

## Troubleshooting

### VM Not Starting

**Check Harvester logs:**
```bash
kubectl logs -n harvester-system deploy/harvester-harvester -f
```

**Check VM status:**
```bash
kubectl get virtualmachine -A
kubectl describe virtualmachine <name> -n <namespace>
```

### Network Connectivity Issues

**Verify VLAN configuration:**
- Ensure VLAN is created in Harvester
- Check physical switch VLAN trunking
- Verify cluster network attachment

**Test from within VM:**
```bash
# SSH to Rocky Linux worker
ssh -i ~/.ssh/butler-ssh rocky@<vm-ip>
ping <gateway-ip>
ping <api-server-ip>
```

### Image Issues

**Verify image exists:**
```bash
kubectl get virtualmachineimage -n default
```

**Check image import status:**
```bash
kubectl describe virtualmachineimage <name> -n default
```

### Kubeconfig Issues

**Test Harvester API access:**
```bash
kubectl --kubeconfig=harvester-kubeconfig.yaml get nodes
```

**Common issues:**
- Wrong server URL (use external address)
- Certificate validation (may need `insecure-skip-tls-verify: true` initially)
- Firewall blocking port 6443

### MachineRequest Stuck

**Check provider controller logs:**
```bash
kubectl logs -n butler-system deploy/butler-provider-harvester -f
```

**Check MachineRequest status:**
```bash
kubectl describe machinerequest <name> -n <namespace>
```

---

## See Also

- [Getting Started](../getting-started/) - Bootstrap guide
- [Architecture](../architecture/) - System design
- [Nutanix Provider](nutanix.md) - Alternative provider
