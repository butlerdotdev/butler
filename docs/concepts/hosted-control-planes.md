---
title: Hosted Control Planes
sidebar_position: 4
---

# Hosted Control Planes

Butler runs tenant cluster control planes as pods in the management cluster instead of dedicating VMs to each tenant's API server, controller-manager, and scheduler.

## How It Works

[Steward](https://github.com/butlerdotdev/steward) (a community-governed fork of Kamaji) manages hosted control planes. For each tenant cluster, Steward creates a TenantControlPlane resource that runs:

- **kube-apiserver** -- Serves the tenant's Kubernetes API on a dedicated endpoint.
- **kube-controller-manager** -- Reconciles tenant resources (deployments, services, etc.).
- **kube-scheduler** -- Schedules tenant pods onto worker nodes.

All three components run as a single Deployment in a dedicated namespace on the management cluster. etcd state is stored in a shared etcd cluster managed by Steward.

## Benefits

- **Faster provisioning** -- Control plane pods start in seconds. No VM boot time.
- **Lower resource cost** -- Each tenant control plane uses ~12 mCPU and ~6 MiB at idle. Hundreds of tenants fit on modest hardware.
- **Centralized operations** -- One etcd cluster, one set of certificates, one backup target.

## Trade-offs

- **Management cluster criticality** -- If the management cluster goes down, all tenant API servers become unavailable. Worker nodes continue running workloads, but kubectl access and new scheduling stop.
- **Resource scaling** -- At 10+ tenants, configure [control plane resource limits](../reference/crds/butlerconfig.md) to prevent etcd starvation.
- **Shared etcd** -- Tenant data is logically isolated (separate key prefixes) but shares physical storage. A misbehaving tenant with high write volume can affect others.

## Exposure Modes

Butler supports three ways to expose tenant API server endpoints:

| Mode | How It Works | Use Case |
|------|-------------|----------|
| LoadBalancer | Each tenant gets a dedicated LoadBalancer Service | On-prem with MetalLB or cloud |
| Ingress | Shared ingress controller routes by hostname | Reduce IP consumption |
| Gateway | Gateway API with per-tenant routes | Modern alternative to Ingress |

Configure the default mode in [ButlerConfig](../reference/crds/butlerconfig.md). Override per-cluster in TenantCluster.spec.controlPlane.serviceType.

## See Also

- [Tenant Clusters](./tenant-clusters.md) -- Full tenant cluster concept
- [Architecture > Tenant Lifecycle](../architecture/tenant-lifecycle.md) -- Reconciliation flow detail
