---
title: MachineRequest
sidebar_position: 9
---

A MachineRequest represents a request to provision a virtual machine on an infrastructure provider.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Namespaced

## Short Name

`mr`

## Description

MachineRequest is the interface contract between Butler's orchestration controllers and infrastructure provider controllers. The bootstrap controller creates MachineRequest resources for management cluster nodes. The butler-controller creates them for tenant cluster workers. Provider controllers (butler-provider-harvester, butler-provider-nutanix, etc.) watch for MachineRequest resources, provision the requested VM, and update the status with the assigned IP address.

The MachineRequest spec is provider-agnostic. All provider-specific details (endpoint, network, image defaults) come from the referenced ProviderConfig.

## Specification

### Full Example

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: MachineRequest
metadata:
  name: butler-mgmt-cp-0
  namespace: butler-system
spec:
  providerRef:
    name: harvester-prod
    namespace: butler-system
  machineName: butler-mgmt-cp-0
  role: control-plane
  cpu: 4
  memoryMB: 16384
  diskGB: 100
  extraDisks:
    - sizeGB: 500
      storageClass: longhorn
  image: "default/talos-v1.9.2"
  userData: |
    #cloud-config
    ...
  labels:
    butler.butlerlabs.dev/cluster: butler-mgmt
    butler.butlerlabs.dev/role: control-plane
```

### Spec Fields

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `providerRef` | ProviderReference | Yes | -- | References the ProviderConfig with infrastructure credentials. |
| `machineName` | string | Yes | 1-63 chars, DNS-safe | Desired VM name. Must be unique within the provider's namespace or project. |
| `role` | MachineRole | Yes | Enum | Machine role: `control-plane` or `worker`. |
| `cpu` | int32 | Yes | 1-128 | Virtual CPU cores. |
| `memoryMB` | int32 | Yes | min 1024 | Memory in megabytes. |
| `diskGB` | int32 | Yes | min 10 | Root disk size in gigabytes. |
| `extraDisks` | []DiskSpec | No | -- | Additional disks to attach. |
| `image` | string | No | -- | OS image override. Format is provider-specific (see below). Defaults to the image configured in ProviderConfig. |
| `userData` | string | No | -- | Cloud-init user data. Typically contains the Talos machine configuration. |
| `networkData` | string | No | -- | Cloud-init network configuration. |
| `labels` | map[string]string | No | -- | Key-value pairs applied to the VM in the provider. |

### DiskSpec

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `sizeGB` | int32 | Yes | min 1 | Disk size in gigabytes. |
| `storageClass` | string | No | -- | Provider-specific storage class or tier. |

### Image Format by Provider

| Provider | Format | Example |
|----------|--------|---------|
| Harvester | `namespace/image-name` | `default/talos-v1.9.2` |
| Nutanix | UUID | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| Proxmox | Template ID or image name | `9000` |
| GCP | Image family or self link | `projects/my-project/global/images/talos-v1-9-2` |
| AWS | AMI ID | `ami-0abcdef1234567890` |
| Azure | Image reference | `/subscriptions/.../images/talos-v1-9-2` |

## Status

```yaml
status:
  phase: Running
  providerID: "abc123-def456"
  ipAddress: "10.40.0.10"
  ipAddresses:
    - "10.40.0.10"
  macAddress: "52:54:00:12:34:56"
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2026-03-10T12:00:00Z"
      reason: "VMRunning"
      message: "Virtual machine is running"
  lastUpdated: "2026-03-10T12:00:00Z"
  observedGeneration: 1
```

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | MachinePhase | Current lifecycle phase (see Phases below). |
| `providerID` | string | Provider-specific VM identifier (Harvester VM UID, Nutanix VM UUID, etc.). |
| `ipAddress` | string | Primary IP address. Set when phase reaches Running. |
| `ipAddresses` | []string | All IP addresses assigned to the machine. |
| `macAddress` | string | Primary MAC address. |
| `failureReason` | string | Machine-readable failure reason. |
| `failureMessage` | string | Human-readable failure details. |
| `conditions` | []Condition | Standard Kubernetes conditions. |
| `lastUpdated` | Time | Timestamp of the last status update. |
| `observedGeneration` | int64 | Last observed spec generation. |

## Phases

| Phase | Description |
|-------|-------------|
| `Pending` | Request received, not yet processed by the provider controller. |
| `Creating` | Provider controller is creating the VM. |
| `Running` | VM is running and has an IP address. |
| `Failed` | VM creation failed. Check `failureReason` and `failureMessage`. |
| `Deleting` | VM is being deleted. |
| `Deleted` | VM has been deleted. |
| `Unknown` | VM state cannot be determined. |

## Provider Controller Contract

Every provider controller must:

1. Watch MachineRequest resources filtered by its provider type
2. Read credentials from the ProviderConfig referenced by `providerRef`
3. Create a VM matching the requested resources (CPU, memory, disk)
4. Update `status.phase` through the lifecycle: Pending, Creating, Running
5. Set `status.ipAddress` when the VM obtains an IP
6. Add a finalizer to ensure VM cleanup on deletion
7. Delete the VM and remove the finalizer during CR deletion
8. Record Kubernetes events for audit trail

## kubectl Output

```
$ kubectl get mr -n butler-system
NAME                 MACHINE              ROLE            PHASE     IP           AGE
butler-mgmt-cp-0    butler-mgmt-cp-0     control-plane   Running   10.40.0.10   30m
butler-mgmt-cp-1    butler-mgmt-cp-1     control-plane   Running   10.40.0.11   30m
butler-mgmt-cp-2    butler-mgmt-cp-2     control-plane   Running   10.40.0.12   30m
butler-mgmt-w-0     butler-mgmt-w-0      worker          Running   10.40.0.20   28m
```

## See Also

- [ClusterBootstrap](./clusterbootstrap.md) - creates MachineRequests during management cluster bootstrap
- [ProviderConfig](./providerconfig.md) - credentials referenced by `providerRef`
- [Provider Guides](../../getting-started/) -- Per-provider setup instructions
