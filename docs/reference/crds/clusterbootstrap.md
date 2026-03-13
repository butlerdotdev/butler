---
title: ClusterBootstrap
sidebar_position: 8
---

A ClusterBootstrap drives the end-to-end provisioning of a Butler management cluster from bare metal or cloud VMs through to a fully operational platform.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Namespaced

## Short Name

`cb`

## Description

ClusterBootstrap is the central resource for management cluster provisioning. It coordinates VM creation, Talos Linux configuration, Kubernetes bootstrap, and addon installation across both on-prem and cloud providers.

The bootstrap controller runs inside a temporary KIND cluster on the operator's workstation. It watches ClusterBootstrap resources and reconciles through a strict phase sequence:

1. Create MachineRequest resources for each node
2. Wait for provider controllers to provision VMs and report IPs
3. Generate and apply Talos machine configurations
4. Bootstrap the first control plane node
5. Install platform addons in dependency order
6. (HA only) Pivot the management plane onto the new cluster

On-prem providers (Harvester, Nutanix, Proxmox) use kube-vip for control plane HA with a floating VIP. Cloud providers (GCP, AWS, Azure) create a LoadBalancerRequest to provision a cloud-native L4 load balancer as the control plane endpoint.

## Specification

### Full Example (HA On-Prem)

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: butler-mgmt
  namespace: butler-system
spec:
  provider: harvester
  providerRef:
    name: harvester-prod
    namespace: butler-system

  cluster:
    name: butler-mgmt
    topology: ha
    controlPlane:
      replicas: 3
      cpu: 4
      memoryMB: 16384
      diskGB: 100
    workers:
      replicas: 3
      cpu: 8
      memoryMB: 32768
      diskGB: 200
      extraDisks:
        - sizeGB: 500
          storageClass: longhorn

  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    vip: "10.40.0.200"
    vipInterface: eth0
    loadBalancerPool:
      start: "10.40.0.210"
      end: "10.40.0.250"

  talos:
    version: v1.9.2
    schematic: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
    installDisk: /dev/vda

  addons:
    cni:
      type: cilium
      hubbleEnabled: true
    storage:
      type: longhorn
      replicaCount: 3
    loadBalancer:
      type: metallb
    controlPlaneHA:
      type: kube-vip
    certManager:
      enabled: true
    ingress:
      type: traefik
      enabled: true
    controlPlaneProvider:
      type: steward
      enabled: true
    capi:
      enabled: true
      version: v1.9.4
    butlerController:
      enabled: true
    gitOps:
      type: flux
      enabled: true

  controlPlaneExposure:
    mode: LoadBalancer
```

### Spec Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | Yes | -- | Infrastructure provider. One of: `harvester`, `nutanix`, `proxmox`, `gcp`, `aws`, `azure`. |
| `providerRef` | ProviderReference | Yes | -- | References the ProviderConfig with infrastructure credentials. |
| `cluster` | ClusterBootstrapClusterSpec | Yes | -- | Cluster topology and node sizing. |
| `network` | ClusterBootstrapNetworkSpec | Yes | -- | Pod CIDR, service CIDR, VIP, load balancer pool. |
| `talos` | ClusterBootstrapTalosSpec | Yes | -- | Talos Linux version, schematic, install disk, config patches. |
| `addons` | ClusterBootstrapAddonsSpec | No | See below | Platform addons to install. |
| `controlPlaneExposure` | ControlPlaneExposureSpec | No | LoadBalancer | How tenant control planes are exposed after bootstrap. Written to ButlerConfig. |
| `paused` | bool | No | false | Pauses reconciliation when true. |

### ClusterBootstrapClusterSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | -- | Cluster name. DNS-safe, 1-63 chars, pattern `^[a-z0-9]([-a-z0-9]*[a-z0-9])?$`. |
| `topology` | string | No | `ha` | `ha` for high-availability (3+ CP nodes + workers) or `single-node` (1 CP, no workers). |
| `controlPlane` | ClusterBootstrapNodePool | Yes | -- | Control plane node pool. |
| `workers` | ClusterBootstrapNodePool | No | -- | Worker node pool. Ignored when topology is `single-node`. |

### ClusterBootstrapNodePool

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `replicas` | int32 | Yes | 1-10 | Number of nodes. Forced to 1 for single-node topology. |
| `cpu` | int32 | Yes | 1-128 | CPU cores per node. |
| `memoryMB` | int32 | Yes | min 2048 | Memory in megabytes per node. |
| `diskGB` | int32 | Yes | min 20 | Root disk size in gigabytes per node. |
| `extraDisks` | []DiskSpec | No | -- | Additional disks. Each has `sizeGB` (min 1) and optional `storageClass`. |
| `labels` | map[string]string | No | -- | Labels applied to nodes in this pool. |

### ClusterBootstrapNetworkSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `podCIDR` | string | Yes | -- | CIDR for pod networking. Pattern: `^([0-9]{1,3}\.){3}[0-9]{1,3}/[0-9]{1,2}$`. |
| `serviceCIDR` | string | Yes | -- | CIDR for service networking. Same pattern. |
| `vip` | string | No | -- | Control plane endpoint. For on-prem: a floating IP managed by kube-vip. For cloud: optional, set automatically from LoadBalancerRequest endpoint. Accepts IP addresses and DNS hostnames. |
| `vipInterface` | string | No | -- | Network interface for the VIP. Only relevant for on-prem with kube-vip. Auto-detected by kube-vip if not specified. |
| `loadBalancerPool` | LoadBalancerPoolSpec | No | -- | IP range for MetalLB. Must not overlap with VIP. |

**LoadBalancerPoolSpec:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | string | Yes | First IP in the pool (inclusive). |
| `end` | string | Yes | Last IP in the pool (inclusive). |

Validation: `start` must be less than or equal to `end`. If `vip` is an IP address (not a hostname), it must not fall within the pool range.

### ClusterBootstrapTalosSpec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `version` | string | Yes | -- | Talos version. Pattern: `^v[0-9]+\.[0-9]+\.[0-9]+$`. |
| `schematic` | string | Yes | -- | Talos factory schematic ID for the boot image. |
| `installDisk` | string | No | `/dev/vda` | Disk device for Talos installation. |
| `configPatches` | []TalosConfigPatch | No | -- | Inline Talos config patches (RFC 6902 JSON Patch format). |

**TalosConfigPatch:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `op` | string | Yes | Patch operation: `add`, `remove`, or `replace`. |
| `path` | string | Yes | JSON path to patch. |
| `value` | string | No | Value to set (required for `add` and `replace`). |

### ClusterBootstrapAddonsSpec

All addon sub-specs are optional. Defaults are applied when the field is omitted.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `cni` | CNIAddonSpec | type: cilium | Container networking. |
| `storage` | StorageAddonSpec | type: longhorn, replicaCount: 3 | Persistent storage. |
| `loadBalancer` | LoadBalancerAddonSpec | type: metallb | LoadBalancer service implementation. |
| `controlPlaneHA` | ControlPlaneHAAddonSpec | type: kube-vip | Control plane HA (on-prem only). |
| `certManager` | CertManagerAddonSpec | enabled: true | TLS certificate automation. |
| `ingress` | IngressAddonSpec | type: traefik, enabled: true | Ingress controller. |
| `controlPlaneProvider` | ControlPlaneProviderAddonSpec | type: steward, enabled: true | Hosted control plane operator. |
| `capi` | CAPIAddonSpec | enabled: true, version: v1.9.4 | Cluster API core + infrastructure providers. |
| `butlerController` | ButlerControllerAddonSpec | enabled: true, version: latest | Butler platform controller. |
| `gitOps` | GitOpsAddonSpec | type: flux, enabled: true | GitOps controller. |
| `console` | ConsoleAddonSpec | enabled: false | Butler web console. |

The CNI, Storage, CAPI, and Console sub-specs are detailed below. The remaining sub-specs (LoadBalancer, ControlPlaneHA, CertManager, Ingress, ControlPlaneProvider, GitOps, ButlerController) follow the same pattern: `enabled` (bool), `type` or `provider` (string), and `version` (string).

#### CNIAddonSpec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | `cilium` | CNI type. Enum: `cilium`, `none`. |
| `version` | string | -- | Override chart version. |
| `hubbleEnabled` | bool | true | Enable Hubble observability (Cilium only). |

#### StorageAddonSpec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | `longhorn` | Storage type. Enum: `longhorn`, `none`. |
| `version` | string | -- | Override chart version. |
| `replicaCount` | *int32 | 3 | Default volume replica count. Forced to 1 for single-node topology. |

#### CAPIAddonSpec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | *bool | true | Install Cluster API. |
| `version` | string | `v1.9.4` | CAPI core version. |
| `infrastructureProviders` | []CAPIInfraProviderSpec | -- | Additional CAPI infrastructure providers. The management cluster's own provider is always included. |

**CAPIInfraProviderSpec:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Provider name. Enum: `harvester`, `nutanix`, `proxmox`, `gcp`, `aws`, `azure`. |
| `version` | string | No | Override provider version. |
| `credentialsSecretRef` | SecretReference | No | Credentials for providers other than the management cluster's own. |

#### ConsoleAddonSpec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | *bool | false | Install Butler Console. |
| `version` | string | `latest` | Console image tag. |
| `ingress` | ConsoleIngressSpec | -- | Ingress configuration for the console. |

**ConsoleIngressSpec:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | false | Create an Ingress resource for the console. |
| `host` | string | `butler.<cluster>.local` | Hostname for console access. |
| `className` | string | -- | Ingress class (e.g., `traefik`, `nginx`). |
| `tls` | bool | false | Enable TLS termination. |
| `tlsSecretName` | string | -- | Name of TLS Secret. |

### ControlPlaneExposureSpec

Configures how tenant control planes are exposed after bootstrap. This setting is written to the ButlerConfig singleton and inherited by all TenantClusters.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | string | `LoadBalancer` | Exposure mode. Enum: `LoadBalancer`, `Ingress`, `Gateway`. |
| `hostname` | string | -- | Wildcard domain for tenant API servers (e.g., `*.k8s.platform.example.com`). Required for Ingress and Gateway modes. |
| `ingressClassName` | string | -- | Ingress class for Ingress mode. |
| `controllerType` | string | -- | Ingress controller type for TLS passthrough. Enum: `haproxy`, `nginx`, `traefik`, `generic`. |
| `gatewayRef` | string | -- | Gateway resource reference for Gateway mode (format: `namespace/name`). Required for Gateway mode. |

## Status

```yaml
status:
  phase: Ready
  controlPlaneEndpoint: "10.40.0.200"
  kubeconfig: "base64-encoded-kubeconfig..."
  talosconfig: "base64-encoded-talosconfig..."
  consoleURL: "https://butler.mgmt.example.com"
  machines:
    - name: butler-mgmt-cp-0
      role: control-plane
      phase: Running
      ipAddress: "10.40.0.10"
      talosConfigured: true
      ready: true
    - name: butler-mgmt-cp-1
      role: control-plane
      phase: Running
      ipAddress: "10.40.0.11"
      talosConfigured: true
      ready: true
    - name: butler-mgmt-cp-2
      role: control-plane
      phase: Running
      ipAddress: "10.40.0.12"
      talosConfigured: true
      ready: true
  addonsInstalled:
    cilium: true
    cert-manager: true
    longhorn: true
    metallb: true
    kube-vip: true
    steward: true
    capi: true
    butler-controller: true
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2026-03-10T12:30:00Z"
      reason: "BootstrapComplete"
  lastUpdated: "2026-03-10T12:30:00Z"
  observedGeneration: 1
```

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | ClusterBootstrapPhase | Current lifecycle phase (see Phases below). |
| `controlPlaneEndpoint` | string | API server endpoint (VIP for on-prem, LB IP/DNS for cloud). |
| `kubeconfig` | string | Base64-encoded admin kubeconfig for the new cluster. |
| `talosconfig` | string | Base64-encoded talosconfig for Talos API access. |
| `consoleURL` | string | URL of the Butler Console (if installed). |
| `machines` | []ClusterBootstrapMachineStatus | Per-machine status. |
| `failureReason` | string | Machine-readable failure reason. |
| `failureMessage` | string | Human-readable failure details. |
| `addonsInstalled` | map[string]bool | Tracks which addons completed installation. |
| `conditions` | []Condition | Standard Kubernetes conditions. |
| `lastUpdated` | Time | Timestamp of the last status update. |
| `observedGeneration` | int64 | Last observed spec generation. |

### ClusterBootstrapMachineStatus

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | MachineRequest name. |
| `role` | string | `control-plane` or `worker`. |
| `phase` | string | MachineRequest phase (Pending, Creating, Running, Failed). |
| `ipAddress` | string | Assigned IP address. |
| `talosConfigured` | bool | True after Talos config has been applied to the node. |
| `ready` | bool | True after the node has joined the Kubernetes cluster. |

## Phases

| Phase | Description |
|-------|-------------|
| `Pending` | ClusterBootstrap created, not yet reconciled. |
| `ProvisioningMachines` | MachineRequest resources created, waiting for VMs to report IPs. |
| `ConfiguringTalos` | All VMs have IPs. Generating and applying Talos machine configurations. |
| `BootstrappingCluster` | Talos configs applied. Bootstrapping etcd and Kubernetes on the first control plane node. |
| `InstallingAddons` | Kubernetes API available. Installing platform addons in dependency order. |
| `Pivoting` | (HA only) Moving management plane onto the new cluster. |
| `Ready` | Bootstrap complete. Management cluster is fully operational. |
| `Failed` | Bootstrap failed. Check `failureReason` and `failureMessage`. |

## Conditions

| Type | Description |
|------|-------------|
| `Ready` | True when bootstrap is complete and the cluster is operational. |
| `Progressing` | True while bootstrap is actively working. |
| `Failed` | True when bootstrap has encountered a terminal failure. |

## On-Prem vs Cloud Bootstrap

The bootstrap flow adapts based on the provider type:

| Aspect | On-Prem (Harvester, Nutanix, Proxmox) | Cloud (GCP, AWS, Azure) |
|--------|---------------------------------------|------------------------|
| Control plane HA | kube-vip floating VIP | Cloud L4 load balancer via LoadBalancerRequest |
| CP endpoint source | `network.vip` field | LoadBalancerRequest `status.endpoint` |
| MetalLB | Installed for LoadBalancer services | Not installed (no `loadBalancerPool` configured for cloud providers) |
| kube-vip | Installed | Skipped |
| Loopback patch | Not needed | Applied to each CP node so kube-apiserver accepts LB-routed packets |
| Traefik | Installed for ingress | Skipped |

For cloud providers, the bootstrap controller creates a LoadBalancerRequest resource after VMs are provisioned. The cloud provider controller provisions the load balancer and reports its endpoint. That endpoint becomes the `controlPlaneEndpoint` used in Talos machine configs.

## Validation Rules

CEL validation rules enforce these constraints:

- When `controlPlaneExposure.mode` is `Ingress`, `hostname` must be set.
- When `controlPlaneExposure.mode` is `Gateway`, both `hostname` and `gatewayRef` must be set.
- `network.vip` must not fall within `network.loadBalancerPool` range (validated in Go, prevents kube-vip and MetalLB conflicts).

## Topology Comparison

| | Single-Node | HA |
|---|-------------|-----|
| Control plane nodes | 1 | 3 (recommended) |
| Worker nodes | 0 (CP is schedulable) | 1+ |
| etcd | Single member | 3-member cluster |
| Storage replicas | Forced to 1 | Default 3 |
| kube-vip | Skipped | Installed (on-prem) |
| Pivoting | Skipped | Enabled |
| Use case | Dev, testing, edge | Production |

## Finalizer

| Finalizer | Purpose |
|-----------|---------|
| `clusterbootstrap.butler.butlerlabs.dev/finalizer` | Ensures cleanup of MachineRequests, LoadBalancerRequests, and Secrets before CR deletion. |

## kubectl Output

```
$ kubectl get cb -n butler-system
NAME           CLUSTER       TOPOLOGY   PHASE   ENDPOINT       AGE
butler-mgmt   butler-mgmt   ha         Ready   10.40.0.200    2h
```

## Examples

### Single-Node Development

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: dev-cluster
  namespace: butler-system
spec:
  provider: harvester
  providerRef:
    name: harvester-dev
    namespace: butler-system
  cluster:
    name: dev-cluster
    topology: single-node
    controlPlane:
      replicas: 1
      cpu: 4
      memoryMB: 16384
      diskGB: 100
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    vip: "10.40.0.100"
  talos:
    version: v1.9.2
    schematic: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
```

### HA Cloud (GCP)

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: butler-prod
  namespace: butler-system
spec:
  provider: gcp
  providerRef:
    name: gcp-prod
    namespace: butler-system
  cluster:
    name: butler-prod
    topology: ha
    controlPlane:
      replicas: 3
      cpu: 4
      memoryMB: 16384
      diskGB: 100
    workers:
      replicas: 3
      cpu: 8
      memoryMB: 32768
      diskGB: 200
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
  talos:
    version: v1.9.2
    schematic: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
  addons:
    cni:
      type: cilium
      hubbleEnabled: true
    storage:
      type: longhorn
    controlPlaneProvider:
      type: steward
    capi:
      enabled: true
      infrastructureProviders:
        - name: gcp
  controlPlaneExposure:
    mode: LoadBalancer
```

### HA On-Prem with Gateway Exposure

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: butler-mgmt
  namespace: butler-system
spec:
  provider: nutanix
  providerRef:
    name: nutanix-dc1
    namespace: butler-system
  cluster:
    name: butler-mgmt
    topology: ha
    controlPlane:
      replicas: 3
      cpu: 4
      memoryMB: 16384
      diskGB: 100
    workers:
      replicas: 3
      cpu: 8
      memoryMB: 32768
      diskGB: 200
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    vip: "10.0.0.200"
    loadBalancerPool:
      start: "10.0.0.210"
      end: "10.0.0.250"
  talos:
    version: v1.9.2
    schematic: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
  addons:
    ingress:
      type: traefik
    console:
      enabled: true
      ingress:
        enabled: true
        host: butler.platform.example.com
        className: traefik
        tls: true
  controlPlaneExposure:
    mode: Gateway
    hostname: "*.k8s.platform.example.com"
    gatewayRef: "butler-system/tenant-gateway"
```

## See Also

- [LoadBalancerRequest](./loadbalancerrequest.md) - cloud control plane load balancer
- [MachineRequest](./machinerequest.md) - VM provisioning interface
- [ProviderConfig](./providerconfig.md) - infrastructure credentials
- [Bootstrap Flow](../../architecture/bootstrap-flow.md) - end-to-end bootstrap sequence
- [Provider Guides](../../providers/) - per-provider setup instructions
