---
title: Tenant Clusters
sidebar_position: 3
---

# Tenant Clusters

A tenant cluster is a Kubernetes cluster that Butler provisions and manages for running workloads. Each tenant cluster is represented by a [TenantCluster](../reference/crds/tenantcluster.md) Custom Resource.

## What You Get

When you create a TenantCluster, Butler:

1. Creates a hosted control plane (API server, controller-manager, scheduler) as pods in the management cluster via Steward.
2. Provisions worker VMs on the configured infrastructure provider.
3. Bootstraps workers to join the cluster (Talos apply-config, kubeadm join, or Ignition depending on OS type).
4. Installs platform addons (CNI, load balancer, storage).
5. Delivers a kubeconfig for cluster access.

The result is a fully isolated Kubernetes cluster with its own API server endpoint, its own workload nodes, and its own set of addons.

## Lifecycle Phases

| Phase | Description |
|-------|-------------|
| Pending | TenantCluster CR created, awaiting reconciliation |
| Provisioning | Control plane and worker VMs being created |
| Installing | Platform addons being installed on the tenant cluster |
| Ready | Cluster fully operational, all addons healthy |
| Updating | Processing spec changes (scale, version, addons) |
| Deleting | Cleaning up workers, control plane, and associated resources |
| Failed | Error state; check conditions on the TenantCluster for details |

## How Spec Maps to Infrastructure

```
TenantCluster.spec
├── controlPlane       → Steward TenantControlPlane (pods in management cluster)
├── workers.replicas   → CAPI MachineDeployment → MachineRequests → VMs on provider
├── providerConfigRef  → Credentials and config for the target infrastructure
├── addons             → TenantAddon resources (Helm releases on tenant cluster)
└── networking         → IPAllocation from NetworkPool (on-prem) or cloud networking
```

## Multi-OS Workers

Butler supports five worker node operating systems:

| OS | Bootstrap Method | Notes |
|----|-----------------|-------|
| Talos | Machine config via talosctl | Default for new clusters. Immutable, API-managed. |
| Rocky | kubeadm join via cloud-init | Max K8s version: v1.30.2 |
| Flatcar | Ignition JSON | Auto-joins via bootstrap token |
| Bottlerocket | TOML settings | Minimal, container-optimized (E2E pending) |
| Kairos | Cloud-config YAML | Immutable, community-driven (E2E pending) |

Set the OS type in `spec.workers.machineTemplate.os.type`.

## See Also

- [Create Your First Tenant Cluster](../getting-started/first-tenant-cluster.md) -- Hands-on guide
- [Hosted Control Planes](./hosted-control-planes.md) -- How control planes run as pods
- [TenantCluster Reference](../reference/crds/tenantcluster.md) -- Full field specification
- [Tenant Lifecycle Architecture](../architecture/tenant-lifecycle.md) -- Internal reconciliation flow
