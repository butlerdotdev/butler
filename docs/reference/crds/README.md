---
title: CRD Reference
sidebar_position: 1
---

Butler extends the Kubernetes API with Custom Resource Definitions (CRDs) under the `butler.butlerlabs.dev` API group.

## API Groups

| API Group | Version | Description |
|-----------|---------|-------------|
| `butler.butlerlabs.dev` | `v1alpha1` | Butler platform resources |
| `steward.butlerlabs.dev` | `v1alpha1` | Hosted control plane resources |

## Core Resources

### Cluster Management

| CRD | Scope | Short Name | Description |
|-----|-------|------------|-------------|
| [TenantCluster](./tenantcluster.md) | Namespaced | `tc` | Tenant Kubernetes cluster |
| ClusterBootstrap | Namespaced | `cb` | Management cluster bootstrap |
| MachineRequest | Namespaced | `mr` | VM provisioning request |

### Platform Configuration

| CRD | Scope | Short Name | Description |
|-----|-------|------------|-------------|
| ButlerConfig | Cluster | `bc` | Platform-wide configuration |
| [ProviderConfig](./providerconfig.md) | Namespaced | `pc` | Infrastructure provider credentials |

### Multi-Tenancy

| CRD | Scope | Short Name | Description |
|-----|-------|------------|-------------|
| [Team](./team.md) | Cluster | `tm` | Team isolation boundary |
| User | Cluster | — | User account |
| IdentityProvider | Cluster | `idp` | SSO/OIDC configuration |

### Addons

| CRD | Scope | Short Name | Description |
|-----|-------|------------|-------------|
| AddonDefinition | Cluster | `ad` | Addon catalog entry |
| TenantAddon | Namespaced | `ta` | Addon on tenant cluster |
| ManagementAddon | Cluster | `ma` | Addon on management cluster |

## Steward Resources

Steward provides hosted control plane management.

| CRD | Scope | Description |
|-----|-------|-------------|
| TenantControlPlane | Namespaced | Hosted Kubernetes control plane |
| DataStore | Cluster | Backend storage for control planes |

## Labels and Annotations

### Standard Labels

| Label | Description |
|-------|-------------|
| `app.kubernetes.io/managed-by: butler` | Resource managed by Butler |
| `butler.butlerlabs.dev/team: <name>` | Team ownership |
| `butler.butlerlabs.dev/cluster: <name>` | Associated tenant cluster |

### Annotations

| Annotation | Description |
|------------|-------------|
| `butler.butlerlabs.dev/description` | Human-readable description |
| `butler.butlerlabs.dev/created-by` | User who created the resource |

## Finalizers

Butler uses finalizers to ensure proper cleanup:

| Finalizer | Applied To | Purpose |
|-----------|------------|---------|
| `butler.butlerlabs.dev/tenant-cluster` | TenantCluster | Cleanup child resources |
| `butler.butlerlabs.dev/team` | Team | Cleanup namespace and RBAC |
| `butler.butlerlabs.dev/addon` | TenantAddon | Uninstall Helm release |

## See Also

- [Architecture](../../architecture/) - System design
- [CLI Reference](../cli/) - Command-line tools
