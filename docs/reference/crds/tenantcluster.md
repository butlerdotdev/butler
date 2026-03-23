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
  kubernetesVersion: "v1.31.0"

  teamRef:
    name: backend-team

  controlPlane:
    replicas: 1
    externalCloudProvider: true
    resources:
      apiServer:
        requests:
          cpu: "100m"
          memory: "256Mi"
        limits:
          cpu: "2"
          memory: "1Gi"

  workers:
    replicas: 3
    machineTemplate:
      cpu: 4
      memory: "16Gi"
      diskSize: "100Gi"
      os:
        type: talos
        version: "v1.12.2"

  providerConfigRef:
    name: harvester-prod

  networking:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    loadBalancerPool:
      start: "10.40.1.100"
      end: "10.40.1.120"

  addons:
    cni:
      provider: cilium
      version: "1.17.0"
    loadBalancer:
      provider: metallb
      version: "0.14.9"
    storage:
      provider: longhorn
      version: "1.7.2"
    certManager:
      enabled: true
      version: "1.16.3"
```

### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kubernetesVersion` | string | Yes | Kubernetes version in format vX.Y.Z (e.g., "v1.31.0") |
| `teamRef` | object | No | Reference to the Team resource. Required when multi-tenancy mode is Enforced |
| `controlPlane` | object | No | Control plane configuration |
| `workers` | object | Yes | Worker node configuration |
| `providerConfigRef` | object | No | Reference to ProviderConfig. Falls back to Team or platform default |
| `networking` | object | No | Network CIDR and load balancer pool configuration |
| `addons` | object | No | Addon configuration |
| `managementPolicy` | object | No | How Butler manages this cluster after initial setup |
| `infrastructureOverride` | object | No | Per-cluster overrides for provider-specific settings |
| `workspaces` | object | No | Cloud development environment configuration |

### controlPlane

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `replicas` | integer | No | `1` | API server replicas (1-3) |
| `dataStoreRef` | object | No | | Reference to Steward DataStore |
| `serviceType` | string | No | | LoadBalancer, NodePort, or ClusterIP. Inherits from ButlerConfig if not set |
| `certSANs` | array | No | | Additional SANs for API server cert |
| `externalCloudProvider` | bool | No | `true` | Enable `--cloud-provider=external` on apiserver and controller-manager. Required for Harvester, vSphere, and cloud providers |
| `resources` | object | No | | Per-component resource limits. Overrides ButlerConfig defaults |

### controlPlane.resources

Per-component resource limits for control plane pods. If a component is set here, it fully replaces the ButlerConfig default for that component. Components not set here inherit from `ButlerConfig.spec.defaultControlPlaneResources`.

```yaml
controlPlane:
  resources:
    apiServer:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "2"
        memory: "1Gi"
    controllerManager:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    scheduler:
      requests:
        cpu: "25m"
        memory: "32Mi"
      limits:
        cpu: "250m"
        memory: "128Mi"
```

### workers

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `replicas` | integer | Yes | Number of worker nodes (minimum 1) |
| `machineTemplate` | object | No | Machine specifications |

### workers.machineTemplate

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `cpu` | integer | No | `4` | CPU cores per worker |
| `memory` | quantity | No | `16Gi` | Memory per worker |
| `diskSize` | quantity | No | `100Gi` | Root disk size per worker |
| `os` | object | No | | Operating system configuration |

### workers.machineTemplate.os

Butler supports multiple operating systems for worker nodes. The OS type determines how the node is bootstrapped and joins the cluster.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | No | `rocky` | OS type: `talos`, `rocky`, `flatcar`, `bottlerocket`, `kairos` |
| `version` | string | No | `9.5` | OS version |
| `imageRef` | string | No | | Specific image reference. Overrides type and version |
| `schematicID` | string | No | | Butler Image Factory schematic ID. Enables automatic image syncing to the target provider |
| `sshAuthorizedKey` | string | No | | SSH public key for node access. Falls back to ButlerConfig default. Not applicable to Talos |
| `talos` | object | No | | Talos-specific configuration. Required when type is `talos` |

**OS Types:**

| Type | Bootstrap Method | Description |
|------|-----------------|-------------|
| `talos` | Talos machine config via `dataSecretName`, post-boot `apply-config` via talosctl | Immutable Linux. Addons wait for config-applied annotation on all workers |
| `rocky` | CABPK KubeadmConfigTemplate via `configRef`, kubeadm join via cloud-init | Rocky Linux. Max K8s version: v1.30.2 (v1.31+ causes BootstrapFailed on Harvester) |
| `flatcar` | Ignition JSON via `dataSecretName`, kubelet auto-joins via bootstrap token | Flatcar Container Linux. Immutable |
| `bottlerocket` | TOML settings via `dataSecretName` | Bottlerocket. Immutable |
| `kairos` | cloud-config YAML via `dataSecretName` | Kairos. Immutable, cloud-config based |

### workers.machineTemplate.os.talos

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `installDisk` | string | No | `/dev/vda` | Disk device for Talos installation |
| `installerImage` | string | No | | Talos installer image (e.g., `factory.talos.dev/installer/<schematic>:v1.12.2`) |
| `version` | string | No | `v1.9.3` | Talos version |

### providerConfigRef

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the ProviderConfig |
| `namespace` | string | No | Namespace of the ProviderConfig. Defaults to butler-system |

### networking

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `podCIDR` | string | No | `10.244.0.0/16` | Pod network CIDR |
| `serviceCIDR` | string | No | `10.96.0.0/12` | Service network CIDR |
| `loadBalancerPool` | object | No | | IP pool for LoadBalancer services. Auto-populated when IPAM is active |
| `lbPoolSize` | integer | No | | Override the default LB pool size from the provider. Only used with IPAM |

### networking.loadBalancerPool

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | string | Yes | First IP address in the pool |
| `end` | string | Yes | Last IP address in the pool |

### addons

Addons are configured as structured fields, not an array. Each addon has a `version` field that is required.

| Field | Type | Description |
|-------|------|-------------|
| `cni` | object | CNI configuration (Cilium) |
| `loadBalancer` | object | Load balancer (MetalLB) |
| `certManager` | object | Certificate management |
| `storage` | object | Persistent storage (Longhorn, Linstor) |
| `ingress` | object | Ingress controller (Traefik, Nginx) |
| `gitops` | object | GitOps (Flux or ArgoCD) |

#### addons.cni

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | No | `cilium` | CNI provider. Only `cilium` is currently supported |
| `version` | string | Yes | | Cilium version (e.g., `1.17.0`) |
| `values` | object | No | | Custom Helm values |

#### addons.loadBalancer

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | No | `metallb` | Load balancer provider. Only `metallb` is currently supported |
| `version` | string | Yes | | MetalLB version (e.g., `0.14.9`) |
| `values` | object | No | | Custom Helm values |

#### addons.certManager

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `enabled` | bool | No | `true` | Whether to install cert-manager |
| `version` | string | Yes | | cert-manager version (e.g., `1.16.3`) |
| `values` | object | No | | Custom Helm values |

#### addons.storage

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | No | | Storage provider: `longhorn` or `linstor` |
| `version` | string | Yes | | Storage provider version |
| `values` | object | No | | Custom Helm values |

#### addons.ingress

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | No | | Ingress controller: `traefik` or `nginx` |
| `version` | string | Yes | | Ingress controller version |
| `values` | object | No | | Custom Helm values |

#### addons.gitops

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | No | | GitOps tool: `fluxcd` or `argocd` |
| `version` | string | No | | GitOps tool version |
| `repository` | object | No | | Git repository configuration |

```yaml
addons:
  gitops:
    provider: fluxcd
    version: "2.4.0"
    repository:
      url: https://github.com/org/clusters
      branch: main
      path: clusters/my-cluster
      secretRef:
        name: git-credentials
```

### managementPolicy

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `mode` | string | No | `Active` | Management mode: `Active`, `Observe`, or `GitOps` |

- **Active**: Butler actively manages addons. New addons in spec are installed.
- **Observe**: Butler only observes after initial install. Spec changes are ignored.
- **GitOps**: Butler bootstraps Flux and hands off addon management to GitOps.

### infrastructureOverride

Per-cluster overrides for provider-specific settings. These take precedence over ProviderConfig defaults. Only the section matching the cluster's provider is used.

```yaml
infrastructureOverride:
  harvester:
    namespace: custom-namespace
    networkName: default/vlan50-dev
    imageName: default/talos-v1-12-2
  nutanix:
    clusterUUID: "..."
    subnetUUID: "..."
    imageUUID: "..."
    storageContainerUUID: "..."
  proxmox:
    node: pve-node-03
    storage: ceph-ssd
    templateID: 9001
  gcp:
    zone: us-central1-b
    machineType: n2-standard-8
    image: talos-v1-12-5-custom
    subnetwork: custom-subnet
```

## Status

The status subresource tracks the current state of the cluster.

```yaml
status:
  phase: Ready
  controlPlaneEndpoint: "10.40.0.201:6443"
  workerNodesReady: 3
  workerNodesDesired: 3
  tenantNamespace: "tenant-my-cluster"
  kubeconfigSecretRef:
    name: my-cluster-kubeconfig
  observedState:
    kubernetesVersion: "v1.31.0"
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
| `tenantNamespace` | string | Namespace containing CAPI/Steward resources |
| `kubeconfigSecretRef` | object | Reference to kubeconfig Secret |
| `observedState` | object | Observed cluster state |
| `ipAllocationRef` | object | Reference to the node IP allocation from IPAM |
| `lbAllocationRef` | object | Reference to the load balancer IP allocation from IPAM |
| `imageSyncRef` | object | Reference to the ImageSync resource for this cluster's OS image |

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
| `NetworkReady` | IP allocation is complete |
| `ProviderAccessGranted` | Provider scope check passed |
| `ImageReady` | OS image is synced to the provider |
| `Ready` | Overall cluster readiness |

## Labels

Butler automatically applies these labels:

| Label | Description |
|-------|-------------|
| `butler.butlerlabs.dev/team` | Team name |
| `butler.butlerlabs.dev/tenant` | Cluster name |
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
  kubernetesVersion: "v1.31.0"
  workers:
    replicas: 2
  addons:
    cni:
      provider: cilium
      version: "1.17.0"
    loadBalancer:
      provider: metallb
      version: "0.14.9"
```

### Production Cluster with Talos Workers

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: prod-api
  namespace: team-backend
spec:
  kubernetesVersion: "v1.31.0"
  controlPlane:
    replicas: 3
    externalCloudProvider: true
    resources:
      apiServer:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "4"
          memory: "2Gi"
  workers:
    replicas: 5
    machineTemplate:
      cpu: 8
      memory: "32Gi"
      diskSize: "200Gi"
      os:
        type: talos
        version: "v1.12.2"
        schematicID: "71e06ba76d3cf365bb4ab4d8f8f4fea55a7620811666b9c25623734ab18ddd27"
  providerConfigRef:
    name: harvester-prod
  networking:
    loadBalancerPool:
      start: "10.40.2.100"
      end: "10.40.2.150"
  addons:
    cni:
      provider: cilium
      version: "1.17.0"
    loadBalancer:
      provider: metallb
      version: "0.14.9"
    storage:
      provider: longhorn
      version: "1.7.2"
    certManager:
      enabled: true
      version: "1.16.3"
    ingress:
      provider: traefik
      version: "3.3.0"
    gitops:
      provider: fluxcd
      version: "2.4.0"
      repository:
        url: https://github.com/myorg/clusters
        branch: main
        path: clusters/prod-api
```

### Multi-OS: Rocky Linux Workers

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: rocky-cluster
  namespace: team-legacy
spec:
  kubernetesVersion: "v1.30.2"
  workers:
    replicas: 3
    machineTemplate:
      cpu: 4
      memory: "16Gi"
      diskSize: "100Gi"
      os:
        type: rocky
        version: "9.5"
  addons:
    cni:
      provider: cilium
      version: "1.17.0"
    loadBalancer:
      provider: metallb
      version: "0.14.9"
```

## See Also

- [Getting Started](../../getting-started/) - Create your first cluster
- [Tenant Lifecycle](../../architecture/tenant-lifecycle.md) - How clusters are provisioned
- [ButlerConfig](./butlerconfig.md) - Platform-wide defaults (exposure mode, control plane resources)
- [butlerctl cluster](../cli/butlerctl.md#butlerctl-cluster) - CLI commands
