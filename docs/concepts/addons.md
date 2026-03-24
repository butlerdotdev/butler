---
title: Addons
sidebar_position: 7
---

# Addons

Butler manages cluster addons through a Helm-based catalog system. Addons extend cluster functionality with networking, storage, ingress, monitoring, and other capabilities.

## Addon Types

### Platform Addons

Required for cluster operation. Butler installs these automatically during cluster creation. They cannot be removed through the Console or CLI.

| Addon | Purpose |
|-------|---------|
| Cilium | CNI networking (pod-to-pod communication) |
| MetalLB | LoadBalancer Services (on-prem clusters only) |
| kube-vip | Control plane virtual IP (on-prem management clusters) |

### Optional Addons

Users install and remove these through TenantCluster spec, the Console, or directly via TenantAddon CRDs.

| Category | Available Addons |
|----------|-----------------|
| Storage | Longhorn, OpenEBS |
| Ingress | Traefik, nginx-ingress |
| Certificates | cert-manager |
| Monitoring | Prometheus, Victoria Metrics |
| GitOps | Flux, ArgoCD |

## How It Works

Three CRDs drive the addon system:

**AddonDefinition** -- Cluster-scoped catalog entries. Each definition specifies a Helm chart repository, chart name, available versions, default values, and whether the addon is a platform requirement. Platform admins manage the catalog.

**TenantAddon** -- Namespaced resources that represent an addon installed on a specific tenant cluster. Butler creates TenantAddons from `TenantCluster.spec.addons` or users can create them directly. The butler-controller installs the Helm chart on the tenant cluster and tracks its health.

**ManagementAddon** -- Cluster-scoped resources for addons installed on the management cluster itself (Steward, CAPI providers, monitoring stack).

## Specifying Addons

Define addons in the TenantCluster spec:

```yaml
spec:
  addons:
    cni:
      provider: cilium
      version: "1.17.0"
    loadBalancer:
      provider: metallb
      version: "0.14.9"
    storage:
      provider: longhorn
      version: "1.7.0"
    certManager:
      enabled: true
      version: "1.16.0"
```

Each addon's `version` field is required. Butler does not auto-select versions.

## See Also

- [Architecture > Addon System](../architecture/addon-system.md) -- Installation sequence, dependency resolution, and controller flow
- [TenantCluster Reference](../reference/crds/tenantcluster.md) -- Addon spec fields
