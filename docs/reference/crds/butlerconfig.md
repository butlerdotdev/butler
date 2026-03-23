---
title: ButlerConfig
sidebar_position: 1
---

ButlerConfig is the singleton platform configuration resource. It defines platform-wide defaults that affect all teams and tenant clusters.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Cluster

## Short Name

`bc`

## Description

ButlerConfig is a singleton resource named `butler` that configures platform-wide behavior. It is created automatically during bootstrap and lives for the lifetime of the management cluster. Operators edit it to:

- **Switch how tenant API servers are exposed** (LoadBalancer, Ingress, or Gateway mode)
- **Set default control plane resource limits** to prevent etcd starvation when running many tenants
- **Enable multi-tenancy enforcement** so all clusters must belong to a Team
- **Configure a default infrastructure provider** for teams that do not specify their own
- **Set up GitOps integration** to enable Flux-based workflows and cluster export
- **Configure the Image Factory** for automatic OS image syncing to providers
- **Define default addon versions** so tenant clusters inherit consistent versions
- **Set default team resource limits** applied to new Teams

Changes to ButlerConfig take effect on new TenantClusters. Existing clusters retain their settings unless explicitly updated.

## Common Operations

### Switch Control Plane Exposure Mode

To move from LoadBalancer mode (1 IP per tenant) to Ingress mode (shared IP via SNI):

```yaml
spec:
  controlPlaneExposure:
    mode: Ingress
    hostname: "*.k8s.platform.example.com"
    ingressClassName: haproxy
    controllerType: haproxy
```

### Set Default Control Plane Resources

Without resource requests, tenant control plane pods run as BestEffort QoS and can be starved by other workloads. At 10+ tenants, set explicit resources to prevent etcd-driven instability:

```yaml
spec:
  defaultControlPlaneResources:
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

### Enable Multi-Tenancy

```yaml
spec:
  multiTenancy:
    mode: Enforced
  defaultTeamLimits:
    maxClusters: 5
    maxWorkersPerCluster: 10
    maxTotalCPU: "64"
    maxTotalMemory: "128Gi"
```

### Configure Default Provider

```yaml
spec:
  defaultProviderConfigRef:
    name: harvester-prod
```

## Spec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `multiTenancy` | object | No | | Multi-tenancy configuration |
| `defaultNamespace` | string | No | `butler-tenants` | Namespace for TenantClusters when not using Teams |
| `defaultProviderConfigRef` | object | No | | Reference to the default ProviderConfig |
| `defaultTeamLimits` | object | No | | Default resource limits for new Teams |
| `defaultAddonVersions` | object | No | | Default addon versions for new TenantClusters |
| `gitProvider` | object | No | | Git provider for GitOps operations |
| `controlPlaneExposure` | object | No | | How tenant control planes are exposed |
| `defaultControlPlaneResources` | object | No | | Default resource limits for tenant control plane components |
| `observability` | object | No | | Platform-level observability pipeline |
| `imageFactory` | object | No | | Butler Image Factory connection |
| `sshAuthorizedKey` | string | No | | Default SSH public key for non-Talos worker nodes |

### multiTenancy

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `mode` | string | No | `Disabled` | Multi-tenancy mode: `Enforced`, `Optional`, or `Disabled` |

**Modes:**

| Mode | Description |
|------|-------------|
| `Enforced` | All TenantClusters must belong to a Team. Teams must exist before clusters. Recommended for enterprise |
| `Optional` | Teams are available but not required. Clusters can exist in the default namespace |
| `Disabled` | Team functionality is off. All clusters exist in the default namespace |

### defaultTeamLimits

Platform-wide defaults applied to new Teams. Individual Teams can override these in their spec.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `maxClusters` | integer | No | Maximum TenantClusters per Team |
| `maxWorkersPerCluster` | integer | No | Maximum workers per TenantCluster |
| `maxTotalCPU` | quantity | No | Maximum total CPU across all clusters |
| `maxTotalMemory` | quantity | No | Maximum total memory across all clusters |
| `maxTotalStorage` | quantity | No | Maximum total storage across all clusters |

### defaultAddonVersions

Default versions for addons when TenantClusters do not specify versions.

| Field | Type | Description |
|-------|------|-------------|
| `cilium` | string | Cilium version |
| `metallb` | string | MetalLB version |
| `certManager` | string | cert-manager version |
| `longhorn` | string | Longhorn version |
| `traefik` | string | Traefik version |
| `fluxcd` | string | FluxCD version |

### controlPlaneExposure

Configures how tenant API servers are exposed. Set during bootstrap and inherited by all TenantClusters.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `mode` | string | No | `LoadBalancer` | Exposure mode: `LoadBalancer`, `Ingress`, or `Gateway` |
| `hostname` | string | Ingress/Gateway | | Wildcard domain for tenant API servers (e.g., `*.k8s.platform.example.com`) |
| `ingressClassName` | string | No | | Ingress class when mode is `Ingress` |
| `controllerType` | string | No | | Ingress controller type for TLS passthrough: `haproxy`, `nginx`, `traefik`, `generic` |
| `gatewayRef` | string | Gateway | | Gateway resource reference (format: `namespace/name`) |

**Exposure Modes:**

| Mode | IPs | Routing | tcp-proxy |
|------|-----|---------|-----------|
| `LoadBalancer` | 1 per tenant | Direct TCP | Not required |
| `Ingress` | Shared | SNI via Ingress controller | Auto-enabled |
| `Gateway` | Shared | SNI via Gateway API TLSRoute | Auto-enabled |

### defaultControlPlaneResources

Default resource allocations for tenant control plane components. Applied to new TenantClusters that do not specify per-cluster overrides. If not set, pods run with no resource requests (BestEffort QoS).

Each component (apiServer, controllerManager, scheduler) has `requests` and `limits` with `cpu` and `memory` fields.

| Field | Type | Description |
|-------|------|-------------|
| `apiServer` | object | kube-apiserver resources |
| `controllerManager` | object | kube-controller-manager resources |
| `scheduler` | object | kube-scheduler resources |

### gitProvider

Configures the Git provider for GitOps operations (cluster export, Flux enablement, addon management via Git).

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | Yes | | Provider type: `github`, `gitlab`, `bitbucket` |
| `url` | string | No | `https://api.github.com` | Git provider API URL |
| `organization` | string | No | | Default organization/group for repositories |
| `secretRef` | object | Yes | | Reference to Secret containing credentials |

**Secret keys by provider:**

| Provider | Required Keys |
|----------|--------------|
| `github` | `token` (PAT with repo scope) |
| `gitlab` | `token` (PAT with api scope) |
| `bitbucket` | `username`, `app-password` |

### imageFactory

Configures the Butler Image Factory for automatic OS image syncing.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `url` | string | Yes | | Base URL of the Image Factory API |
| `credentialsRef` | object | No | | Reference to Secret containing the `apiKey` |
| `defaultSchematicID` | string | No | | Default schematic when TenantClusters do not specify one |
| `autoSync` | bool | No | `true` | Auto-sync images when a TenantCluster references one not yet on the target provider |

### observability

Platform-level observability pipeline configuration.

| Field | Type | Description |
|-------|------|-------------|
| `pipeline` | object | Centralized observability pipeline cluster |
| `pipeline.clusterRef` | object | Reference to the TenantCluster serving as the pipeline |
| `pipeline.logEndpoint` | string | Vector aggregator ingestion URL |
| `pipeline.metricEndpoint` | string | Remote-write endpoint for metrics |
| `pipeline.traceEndpoint` | string | OTLP endpoint for traces |
| `collection` | object | Default collection settings for tenant clusters |
| `collection.autoEnroll` | object | Auto-install agents: `vectorAgent`, `prometheus`, `otelCollector` (booleans) |
| `collection.logs` | object | Log collection: `podLogs`, `journald`, `kubernetesEvents` (booleans) |
| `collection.metrics` | object | Metric collection: `enabled` (bool), `retention` (string, e.g., "15d") |

## Status

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard Kubernetes conditions |
| `observedGeneration` | integer | Last observed generation |
| `teamCount` | integer | Current number of Teams |
| `clusterCount` | integer | Current number of TenantClusters |
| `gitProvider` | object | Git provider connection status |
| `controlPlaneExposureMode` | string | Active exposure mode |
| `tcpProxyRequired` | bool | Whether tcp-proxy is auto-enabled (true for Ingress/Gateway) |
| `observability` | object | Observability pipeline status |

## Examples

### LoadBalancer Mode (Default)

Simplest configuration for on-premises deployments with MetalLB providing 1 IP per tenant:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ButlerConfig
metadata:
  name: butler
spec:
  multiTenancy:
    mode: Disabled
  defaultProviderConfigRef:
    name: harvester-prod
  controlPlaneExposure:
    mode: LoadBalancer
```

### Ingress Mode with HAProxy

Share a single IP across tenants using SNI-based routing through an HAProxy Ingress controller:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ButlerConfig
metadata:
  name: butler
spec:
  multiTenancy:
    mode: Enforced
  defaultProviderConfigRef:
    name: harvester-prod
  controlPlaneExposure:
    mode: Ingress
    hostname: "*.k8s.butlerlabs.dev"
    ingressClassName: haproxy
    controllerType: haproxy
  defaultTeamLimits:
    maxClusters: 5
    maxWorkersPerCluster: 10
```

### Resource-Tuned for 10+ Tenants

Production configuration with control plane resource limits to prevent etcd starvation under multi-tenant load:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ButlerConfig
metadata:
  name: butler
spec:
  multiTenancy:
    mode: Enforced
  defaultProviderConfigRef:
    name: harvester-prod
  controlPlaneExposure:
    mode: LoadBalancer
  defaultControlPlaneResources:
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
  defaultTeamLimits:
    maxClusters: 10
    maxWorkersPerCluster: 20
    maxTotalCPU: "128"
    maxTotalMemory: "256Gi"
  defaultAddonVersions:
    cilium: "1.17.0"
    metallb: "0.14.9"
    longhorn: "1.7.2"
    certManager: "1.16.3"
```

## See Also

- [TenantCluster](./tenantcluster.md) -- Inherits control plane resources and exposure mode
- [Team](./team.md) -- Inherits default limits from `defaultTeamLimits`
- [ProviderConfig](./providerconfig.md) -- Referenced via `defaultProviderConfigRef`
- [Bootstrap Config Reference](../bootstrap-config.md) -- Bootstrap writes ButlerConfig automatically
