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

### Networking

| CRD | Scope | Short Name | Description |
|-----|-------|------------|-------------|
| [NetworkPool](./networkpool.md) | Namespaced | `np` | IP address pool for on-prem IPAM |
| [IPAllocation](./ipallocation.md) | Namespaced | `ipa` | Individual IP allocation from a pool |

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
| `butler.butlerlabs.dev/network-pool` | Associated network pool |
| `butler.butlerlabs.dev/allocation-type` | IP allocation type (loadbalancer, nodes) |

### Annotations

| Annotation | Description |
|------------|-------------|
| `butler.butlerlabs.dev/description` | Human-readable description |
| `butler.butlerlabs.dev/created-by` | User who created the resource |

## Finalizers

Butler uses finalizers to ensure proper cleanup:

| Finalizer | Applied To | Purpose |
|-----------|------------|---------|
| `butler.butlerlabs.dev/tenantcluster` | TenantCluster | Cleanup child resources |
| `butler.butlerlabs.dev/team` | Team | Cleanup namespace and RBAC |
| `butler.butlerlabs.dev/tenantaddon` | TenantAddon | Uninstall Helm release |
| `butler.butlerlabs.dev/networkpool` | NetworkPool | Block deletion with active allocations |
| `butler.butlerlabs.dev/ipallocation` | IPAllocation | Ensure Released phase for audit trail |
| `butler.butlerlabs.dev/providerconfig` | ProviderConfig | Block deletion if TenantClusters reference it |

## See Also

- [Architecture](../../architecture/) - System design
- [CLI Reference](../cli/) - Command-line tools
