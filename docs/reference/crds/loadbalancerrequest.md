---
title: LoadBalancerRequest
sidebar_position: 7
---

A LoadBalancerRequest provisions a cloud-native L4 load balancer for a management cluster's control plane endpoint.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Namespaced

## Short Name

`lbr`

## Description

LoadBalancerRequest is used during cloud bootstrap to create a load balancer that fronts the control plane nodes. The bootstrap controller creates a LoadBalancerRequest, and the appropriate cloud provider controller (GCP, AWS, or Azure) provisions the cloud-native load balancer and reports its endpoint. That endpoint is then baked into Talos machine configs as the control plane address before nodes boot.

On-prem deployments do not use LoadBalancerRequest. They use kube-vip for control plane HA instead.

## Specification

### Full Example

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: LoadBalancerRequest
metadata:
  name: butler-mgmt-lb
  namespace: butler-system
spec:
  # Name of the cluster this LB serves. Used as a prefix for cloud resource names.
  clusterName: butler-mgmt

  # Reference to ProviderConfig with cloud credentials.
  providerConfigRef:
    name: gcp-prod
    namespace: butler-system

  # Backend port (default: 6443)
  port: 6443

  # Health check port (defaults to port value)
  healthCheckPort: 6443

  # Backend instances, updated incrementally as CP nodes come online.
  targets:
    - ip: "10.128.0.10"
      instanceName: "butler-mgmt-cp-0"
    - ip: "10.128.0.11"
      instanceName: "butler-mgmt-cp-1"
    - ip: "10.128.0.12"
      instanceName: "butler-mgmt-cp-2"
```

### Spec Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `clusterName` | string | Yes | -- | DNS-safe name (1-63 chars, `^[a-z0-9]([-a-z0-9]*[a-z0-9])?$`). Immutable after creation. |
| `providerConfigRef` | ProviderReference | Yes | -- | References the ProviderConfig with cloud credentials. Immutable after creation. |
| `port` | int32 | No | 6443 | Backend port on control plane nodes. Range: 1-65535. |
| `healthCheckPort` | int32 | No | Same as `port` | Port for LB health checks. Range: 1-65535. |
| `targets` | []LoadBalancerTarget | No | -- | Backend instances to register. Updated incrementally by the bootstrap controller. |

### LoadBalancerTarget

| Field | Type | Description |
|-------|------|-------------|
| `ip` | string | Target IP address. |
| `instanceID` | string | Cloud instance identifier. Used by AWS for instance-based NLB target groups. |
| `instanceName` | string | Cloud instance name. Used by GCP for target pool registration. |

Which target fields are populated depends on the cloud provider:
- **GCP**: `ip` and `instanceName`
- **AWS**: `ip` and `instanceID`
- **Azure**: `ip` only (NICs are added to the backend pool directly)

## Status

```yaml
status:
  phase: Ready
  endpoint: "35.194.52.218"
  resourceID: "butler-mgmt-lb-fwd"
  registeredTargets: 3
  conditions:
    - type: Provisioned
      status: "True"
      lastTransitionTime: "2026-03-10T12:00:00Z"
      reason: "LBReady"
      message: "Load balancer resources created"
    - type: TargetsSynced
      status: "True"
      lastTransitionTime: "2026-03-10T12:05:00Z"
      reason: "AllTargetsRegistered"
      message: "3 targets registered"
  lastUpdated: "2026-03-10T12:05:00Z"
  observedGeneration: 2
```

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | LoadBalancerPhase | Current lifecycle phase. |
| `endpoint` | string | LB IP address or DNS name. Populated when phase reaches Ready. |
| `resourceID` | string | Cloud resource identifier used for cleanup. |
| `failureReason` | string | Machine-readable failure reason. |
| `failureMessage` | string | Human-readable failure details. |
| `registeredTargets` | int32 | Number of backends registered with the LB. |
| `conditions` | []Condition | Standard Kubernetes conditions (see below). |
| `lastUpdated` | Time | Timestamp of the last status update. |
| `observedGeneration` | int64 | Last observed spec generation. |

### Phases

| Phase | Description |
|-------|-------------|
| `Pending` | Request received, not yet processed by the provider controller. |
| `Creating` | Cloud LB resources are being provisioned. |
| `Ready` | LB is provisioned and has an endpoint. Control plane traffic can flow. |
| `Failed` | Provisioning failed. Check `failureReason` and `failureMessage`. |
| `Deleting` | LB resources are being torn down. |

### Conditions

| Type | Description |
|------|-------------|
| `Provisioned` | True when all cloud LB resources (IP, health check, forwarding rule) exist. |
| `TargetsSynced` | True when all `spec.targets` are registered with the LB backend. |

## Cloud Resources by Provider

Each provider controller creates different cloud resources to implement the L4 passthrough load balancer:

| Provider | Resources Created |
|----------|-------------------|
| GCP | Regional static IP, legacy HTTP health check, target pool, forwarding rule |
| AWS | Network Load Balancer (NLB), target group (TCP), listener on port 6443 |
| Azure | Public IP, Standard Load Balancer, health probe, load balancing rule, backend pool |

All providers use TCP passthrough (not TLS termination). The kube-apiserver handles TLS directly.

### Loopback Interface Patch

Cloud passthrough load balancers deliver packets with the original destination IP (the LB's IP) unchanged. For the kube-apiserver to accept these connections, each control plane node must recognize the LB IP as a local address. The bootstrap controller handles this by adding a Talos config patch that assigns the LB IP to the loopback interface on each control plane node.

## Finalizer

| Finalizer | Purpose |
|-----------|---------|
| `butler.butlerlabs.dev/loadbalancerrequest` | Ensures cloud LB resources are deleted before the CR is removed. |

## kubectl Output

```
$ kubectl get lbr -n butler-system
NAME              CLUSTER        PHASE   ENDPOINT         TARGETS   AGE
butler-mgmt-lb   butler-mgmt    Ready   35.194.52.218    3         45m
```

## Examples

### GCP

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: LoadBalancerRequest
metadata:
  name: mgmt-lb
  namespace: butler-system
spec:
  clusterName: mgmt
  providerConfigRef:
    name: gcp-config
    namespace: butler-system
  targets:
    - ip: "10.128.0.10"
      instanceName: "mgmt-cp-0"
    - ip: "10.128.0.11"
      instanceName: "mgmt-cp-1"
    - ip: "10.128.0.12"
      instanceName: "mgmt-cp-2"
```

### AWS

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: LoadBalancerRequest
metadata:
  name: mgmt-lb
  namespace: butler-system
spec:
  clusterName: mgmt
  providerConfigRef:
    name: aws-config
    namespace: butler-system
  targets:
    - ip: "10.0.1.10"
      instanceID: "i-0abcd1234efgh5678"
    - ip: "10.0.1.11"
      instanceID: "i-0abcd1234efgh9012"
    - ip: "10.0.1.12"
      instanceID: "i-0abcd1234efgh3456"
```

### Azure

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: LoadBalancerRequest
metadata:
  name: mgmt-lb
  namespace: butler-system
spec:
  clusterName: mgmt
  providerConfigRef:
    name: azure-config
    namespace: butler-system
  targets:
    - ip: "10.0.0.4"
    - ip: "10.0.0.5"
    - ip: "10.0.0.6"
```

## See Also

- [ClusterBootstrap](./clusterbootstrap.md) -- creates LoadBalancerRequest during cloud bootstrap
- [ProviderConfig](./providerconfig.md) -- cloud credentials referenced by `providerConfigRef`
- [Bootstrap Flow](../../architecture/bootstrap-flow.md) -- end-to-end bootstrap sequence
- [GCP Provider Guide](../../providers/gcp.md)
- [AWS Provider Guide](../../providers/aws.md)
- [Azure Provider Guide](../../providers/azure.md)
