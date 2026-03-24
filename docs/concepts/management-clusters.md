---
title: Management Clusters
sidebar_position: 2
---

# Management Clusters

The management cluster runs Butler platform components and hosts tenant control planes. It is the foundation of a Butler deployment.

## What Runs on It

| Component | Purpose |
|-----------|---------|
| butler-controller | Reconciles TenantCluster, Team, ProviderConfig, and other Butler CRDs |
| butler-bootstrap | Orchestrates management cluster creation during initial setup |
| butler-server | HTTP/WebSocket API backend for the Console and CLI |
| butler-console | Web UI for cluster management |
| Steward | Runs tenant API servers, controller-managers, and schedulers as pods |
| Cluster API | Manages tenant worker VM lifecycle through infrastructure providers |
| Cilium | CNI for management cluster networking |
| Longhorn | Persistent storage for etcd and platform state |
| cert-manager | TLS certificate automation |

## Topology Options

Butler supports two management cluster topologies:

**HA (3 control plane nodes)** -- Production deployments. Three control plane nodes with etcd distributed across them. kube-vip provides a virtual IP for API server high availability.

**Single-node** -- Development and evaluation. One node runs all control plane and worker components. Lower resource requirements, no HA.

Both topologies use [Talos Linux](https://www.talos.dev/) as the node operating system. Talos provides an immutable, API-managed OS with no SSH access and a minimal attack surface.

## Capacity

A single management cluster can host hundreds of tenant clusters. The practical limit depends on etcd capacity, node resources, and the resource footprint of each tenant control plane.

## See Also

- [Getting Started](../getting-started/) -- Bootstrap a management cluster
- [Architecture](../architecture/) -- Internal component interactions
- [ButlerConfig Reference](../reference/crds/butlerconfig.md) -- Platform-wide configuration
