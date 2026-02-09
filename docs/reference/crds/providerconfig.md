---
title: ProviderConfig
sidebar_position: 3
---

A ProviderConfig defines connection credentials and settings for an infrastructure provider.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Namespaced

## Short Name

`pc`

## Description

ProviderConfig stores the credentials and configuration needed to connect to an infrastructure provider (Harvester, Nutanix, or Proxmox). Each provider type has its own configuration section with provider-specific fields.

## Spec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider` | string | Yes | Provider type: `harvester`, `nutanix`, or `proxmox` |
| `credentialsRef` | SecretReference | Yes | Reference to Secret containing provider credentials |
| `harvester` | HarvesterConfig | No | Harvester-specific configuration |
| `nutanix` | NutanixConfig | No | Nutanix-specific configuration |
| `proxmox` | ProxmoxConfig | No | Proxmox-specific configuration |

### HarvesterConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `namespace` | string | Yes | Harvester namespace for VMs |
| `networkName` | string | Yes | Network name for VM NICs |
| `imageName` | string | Yes | VM image name |

### NutanixConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `endpoint` | string | Yes | Prism Central endpoint URL |
| `port` | int32 | No | API port (default: 9440) |
| `insecure` | bool | No | Skip TLS verification |
| `clusterUUID` | string | Yes | Target AHV cluster UUID |
| `subnetUUID` | string | Yes | Network subnet UUID |
| `imageUUID` | string | No | VM image UUID |

## Example

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: harvester-prod
  namespace: butler-system
spec:
  provider: harvester
  credentialsRef:
    name: harvester-kubeconfig
    namespace: butler-system
  harvester:
    namespace: default
    networkName: default/vlan40-workloads
    imageName: default/talos-1.8.0
```

## Status

| Field | Type | Description |
|-------|------|-------------|
| `ready` | bool | Whether the provider is reachable |
| `lastValidated` | Time | Last successful connection test |
| `message` | string | Status message or error |

## See Also

- [Provider Guides](../../providers/) - Provider-specific documentation
- [TenantCluster](./tenantcluster.md) - Uses ProviderConfig for infrastructure
