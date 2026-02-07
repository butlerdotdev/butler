# Nutanix Provider Guide

This guide covers setting up Butler with [Nutanix AHV](https://www.nutanix.com/products/ahv) as the infrastructure provider.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Nutanix Setup](#nutanix-setup)
- [Butler Configuration](#butler-configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

Butler uses [CAPX (Cluster API Provider Nutanix)](https://github.com/nutanix-cloud-native/cluster-api-provider-nutanix) to provision VMs on Nutanix AHV via Prism Central.

```mermaid
flowchart LR
    Butler["Butler Controller"] --> CAPI["Cluster API"]
    CAPI --> CAPX["CAPX Provider"]
    CAPX --> PC["Prism Central"]
    PC --> PE["Prism Element"]
    PE --> VMs["Virtual Machines"]
```

### Key Components

| Component | Purpose |
|-----------|---------|
| butler-provider-nutanix | Translates MachineRequests to Nutanix VMs |
| CAPX | Creates NutanixMachine resources |
| Prism Central | Nutanix management plane |
| Prism Element | Per-cluster hypervisor management |

---

## Prerequisites

### Nutanix Environment

- Nutanix AOS 5.20+ or 6.x
- Prism Central 2023.x or later
- Admin credentials for Prism Central
- Network connectivity from Butler to Prism Central (port 9440)

### Required Resources

| Resource | Purpose |
|----------|---------|
| Subnet | Network for cluster nodes |
| VM Image | OS image (uploaded to Prism Central) |
| Cluster | Target Nutanix cluster for VMs |
| Categories (optional) | For VM organization |

---

## Nutanix Setup

### 1. Upload VM Image

#### Via Prism Central UI

1. Navigate to Compute & Storage -> Images
2. Click "Add Image"
3. Select image source:
   - URL: Direct download link
   - Upload: Local file

#### Talos Linux Image

```bash
# Download Talos Nutanix image
curl -LO https://github.com/siderolabs/talos/releases/download/v1.9.0/nutanix-amd64.qcow2
```

Upload with name: `talos-1.9`

#### Rocky Linux Image

Download Rocky Linux cloud image:
```bash
curl -LO https://dl.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud.latest.x86_64.qcow2
```

Upload with name: `rocky-9-cloud`

### 2. Identify Subnet

In Prism Central -> Network & Security -> Subnets

Note the subnet name and UUID for your workload network.

### 3. Identify Cluster

In Prism Central -> Compute & Storage -> Clusters

Note the cluster name and UUID where VMs will be created.

### 4. Create Service Account (Recommended)

For production, create a dedicated service account:

1. Prism Central -> Administration -> Users
2. Create local user with appropriate roles
3. Or configure AD/LDAP integration

Required roles:
- Cluster Admin (for target clusters)
- Or custom role with VM create/delete permissions

---

## Butler Configuration

### 1. Create Credentials Secret

```bash
kubectl create secret generic nutanix-credentials \
  --from-literal=NUTANIX_USER=admin \
  --from-literal=NUTANIX_PASSWORD='your-password' \
  --from-literal=NUTANIX_ENDPOINT='prism-central.example.com' \
  --from-literal=NUTANIX_PORT='9440' \
  --from-literal=NUTANIX_INSECURE='false' \
  -n butler-system
```

### 2. Create ProviderConfig Resource

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: nutanix-prod
  namespace: butler-system
spec:
  type: nutanix
  nutanix:
    credentialsRef:
      name: nutanix-credentials
    prismCentral:
      address: prism-central.example.com
      port: 9440
      insecure: false
    cluster:
      name: cluster-01
      # Or use UUID:
      # uuid: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    subnet:
      name: workload-subnet
      # Or use UUID:
      # uuid: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    image:
      name: talos-1.9
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
    name: nutanix-prod
```

---

## Network Configuration

### VIP for Control Plane

Reserve a static IP for the control plane VIP:

```yaml
# In ClusterBootstrap for management cluster
spec:
  cluster:
    controlPlaneEndpoint: 10.127.14.29
```

kube-vip uses the `ens3` interface for Nutanix VMs.

### MetalLB Pool

Allocate IPs for LoadBalancer services:

```yaml
# ButlerConfig or per-cluster
spec:
  platform:
    metalLBPool: "10.127.14.30-10.127.14.50"
```

---

## Resource Specifications

### Management Cluster Recommendations

| Role | vCPUs | Memory | Disk | Count |
|------|-------|--------|------|-------|
| Control Plane | 4 | 16GB | 100GB | 3 (HA) |
| Worker | 8 | 32GB | 200GB | 0-3 |

### Tenant Cluster Defaults

| Role | vCPUs | Memory | Disk |
|------|-------|--------|------|
| Worker | 4 | 8GB | 40GB |

Override in TenantCluster spec:

```yaml
spec:
  workers:
    replicas: 3
    machineTemplate:
      cpu: 8
      memory: 16Gi
      disk: 100Gi
```

---

## Troubleshooting

### Authentication Failures

**Verify credentials:**
```bash
# Test Prism Central API
curl -k -u admin:password https://prism-central.example.com:9440/api/nutanix/v3/clusters/list \
  -H "Content-Type: application/json" \
  -d '{"length": 10}'
```

**Check secret:**
```bash
kubectl get secret nutanix-credentials -n butler-system -o yaml
```

### VM Not Creating

**Check CAPX controller logs:**
```bash
kubectl logs -n capx-system deploy/capx-controller-manager -f
```

**Check NutanixMachine status:**
```bash
kubectl get nutanixmachine -A
kubectl describe nutanixmachine <n> -n <namespace>
```

### Network Issues

**Verify subnet exists:**
```bash
# Via API
curl -k -u admin:password https://prism-central.example.com:9440/api/nutanix/v3/subnets/list \
  -H "Content-Type: application/json" \
  -d '{"length": 100}'
```

**Check VM network config:**
- Verify DHCP is configured on subnet
- Or use static IP assignment in machine config

### Image Issues

**Verify image exists:**
```bash
# Via API
curl -k -u admin:password https://prism-central.example.com:9440/api/nutanix/v3/images/list \
  -H "Content-Type: application/json" \
  -d '{"length": 100}'
```

**Check image type:**
- Must be DISK image, not ISO
- qcow2 or raw format

### Certificate Issues

If using self-signed certificates:

```yaml
spec:
  nutanix:
    prismCentral:
      insecure: true  # For testing only
```

For production, add CA certificate to trust store.

---

## CAPX Version Compatibility

| Butler Version | CAPX Version | Nutanix AOS |
|----------------|--------------|-------------|
| 0.1.x | 1.4.x | 5.20+, 6.x |

---

## See Also

- [Getting Started](../getting-started/) - Bootstrap guide
- [Architecture](../architecture/) - System design
- [Harvester Provider](harvester.md) - Alternative provider
- [CAPX Documentation](https://opendocs.nutanix.com/capx/latest/)
