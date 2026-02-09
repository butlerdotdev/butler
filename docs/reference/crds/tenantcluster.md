---
title: TenantCluster
sidebar_position: 2
---

A TenantCluster represents a Kubernetes cluster managed by Butler for running tenant workloads.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Namespaced

## Short Name

`tc`

## Description

TenantCluster is the primary resource users interact with to provision Kubernetes clusters. When a TenantCluster is created, Butler:

1. Creates a hosted control plane via Steward (TenantControlPlane)
2. Provisions worker VMs via the configured infrastructure provider
3. Bootstraps workers to join the cluster
4. Installs platform addons (CNI, LoadBalancer, etc.)

## Specification

### Full Example

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: my-cluster
  namespace: team-backend
  labels:
    butler.butlerlabs.dev/team: backend-team
spec:
  # Kubernetes version (must start with 'v')
  kubernetesVersion: "v1.30.0"

  # Team reference (optional, inferred from namespace)
  teamRef:
    name: backend-team

  # Control plane configuration
  controlPlane:
    replicas: 1

  # Worker node configuration
  workers:
    replicas: 3
    machineTemplate:
      cpu: 4
      memory: 8Gi
      disk: 40Gi

  # Infrastructure provider reference
  providerConfigRef:
    name: harvester-prod

  # Network configuration
  networking:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12

  # Addons to install
  addons:
    cni:
      provider: cilium
    loadBalancer:
      provider: metallb
      addressPool: 10.40.1.100-10.40.1.120
    storage:
      provider: longhorn
    certManager:
      enabled: true
```

### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kubernetesVersion` | string | Yes | Kubernetes version in format vX.Y.Z (e.g., "v1.30.0") |
| `teamRef` | object | No | Reference to the Team resource |
| `controlPlane` | object | No | Control plane configuration |
| `workers` | object | Yes | Worker node configuration |
| `providerConfigRef` | object | No | Reference to ProviderConfig |
| `networking` | object | No | Network CIDR configuration |
| `addons` | object | No | Addon configuration |
| `managementPolicy` | object | No | How Butler manages this cluster |

### controlPlane

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `replicas` | integer | No | API server replicas (1-3, default: 1) |
| `dataStoreRef` | object | No | Reference to Steward DataStore |
| `serviceType` | string | No | LoadBalancer, NodePort, or ClusterIP |
| `certSANs` | array | No | Additional SANs for API server cert |

### workers

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `replicas` | integer | Yes | Number of worker nodes |
| `machineTemplate` | object | No | Machine specifications |

### workers.machineTemplate

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cpu` | integer | No | CPU cores per worker (default: 4) |
| `memory` | string | No | Memory per worker (default: 8Gi) |
| `disk` | string | No | Disk size per worker (default: 40Gi) |

### providerConfigRef

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the ProviderConfig |

### networking

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `podCIDR` | string | No | Pod network CIDR (default: 10.244.0.0/16) |
| `serviceCIDR` | string | No | Service network CIDR (default: 10.96.0.0/12) |

### addons

Addons are configured as structured fields, not an array:

| Field | Type | Description |
|-------|------|-------------|
| `cni` | object | CNI configuration (Cilium) |
| `loadBalancer` | object | Load balancer (MetalLB) |
| `certManager` | object | Certificate management |
| `storage` | object | Persistent storage (Longhorn) |
| `ingress` | object | Ingress controller |
| `gitops` | object | GitOps (Flux or ArgoCD) |

#### addons.cni

```yaml
addons:
  cni:
    provider: cilium
    version: "1.17.0"  # optional
```

#### addons.loadBalancer

```yaml
addons:
  loadBalancer:
    provider: metallb
    addressPool: 10.40.1.100-10.40.1.120
```

#### addons.gitops

```yaml
addons:
  gitops:
    provider: flux  # or argocd
    repository: https://github.com/org/clusters
    path: clusters/my-cluster
```

## Status

The status subresource tracks the current state of the cluster.

```yaml
status:
  phase: Running
  controlPlaneEndpoint: "10.40.0.201:6443"
  workerNodesReady: 3
  workerNodesDesired: 3
  tenantNamespace: "tenant-my-cluster"
  credentialsRef:
    name: my-cluster-kubeconfig
  observedState:
    kubernetesVersion: "v1.30.0"
    workers:
      desired: 3
      ready: 3
      nodes:
        - my-cluster-worker-0
        - my-cluster-worker-1
        - my-cluster-worker-2
    addons:
      - name: cilium
        version: "1.17.0"
        status: Healthy
        managedBy: butler
```

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Current lifecycle phase |
| `controlPlaneEndpoint` | string | API server endpoint |
| `workerNodesReady` | integer | Number of ready worker nodes |
| `workerNodesDesired` | integer | Desired number of workers |
| `tenantNamespace` | string | Namespace for tenant control plane pods |
| `credentialsRef` | object | Reference to kubeconfig Secret |
| `observedState` | object | Observed cluster state |

### Phases

| Phase | Description |
|-------|-------------|
| `Pending` | CR created, awaiting reconciliation |
| `Provisioning` | Creating control plane and workers |
| `Installing` | Installing platform addons |
| `Ready` | Cluster fully operational |
| `Updating` | Processing spec changes |
| `Deleting` | Cleaning up resources |
| `Failed` | Error state (check conditions) |

### Conditions

| Condition | Description |
|-----------|-------------|
| `InfrastructureReady` | CAPI resources are ready |
| `ControlPlaneReady` | Control plane pods are running |
| `WorkersReady` | All worker nodes have joined |
| `AddonsReady` | Platform addons are healthy |
| `Ready` | Overall cluster readiness |

## Labels

Butler automatically applies these labels:

| Label | Description |
|-------|-------------|
| `butler.butlerlabs.dev/team` | Team name |
| `butler.butlerlabs.dev/cluster` | Cluster name |
| `app.kubernetes.io/managed-by` | Always "butler" |

## Finalizers

| Finalizer | Description |
|-----------|-------------|
| `butler.butlerlabs.dev/tenantcluster` | Ensures cleanup of child resources |

## Examples

### Minimal Cluster

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: dev-cluster
  namespace: butler-tenants
spec:
  kubernetesVersion: "v1.30.0"
  workers:
    replicas: 2
  addons:
    cni:
      provider: cilium
    loadBalancer:
      provider: metallb
      addressPool: 10.40.1.100-10.40.1.110
```

### Production Cluster

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: prod-api
  namespace: team-backend
spec:
  kubernetesVersion: "v1.30.0"
  workers:
    replicas: 5
    machineTemplate:
      cpu: 8
      memory: 32Gi
      disk: 200Gi
  providerConfigRef:
    name: harvester-prod
  addons:
    cni:
      provider: cilium
    loadBalancer:
      provider: metallb
      addressPool: 10.40.2.100-10.40.2.150
    storage:
      provider: longhorn
    certManager:
      enabled: true
    gitops:
      provider: flux
      repository: https://github.com/myorg/clusters
      path: clusters/prod-api
```

## See Also

- [Getting Started](../../getting-started/) - Create your first cluster
- [Tenant Lifecycle](../../architecture/tenant-lifecycle.md) - How clusters are provisioned
- [butlerctl cluster](../cli/butlerctl.md#butlerctl-cluster) - CLI commands
