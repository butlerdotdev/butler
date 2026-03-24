---
title: Scale
sidebar_position: 5
---

# Scale

## Scale Management Cluster Workers

Add worker nodes to the management cluster to increase capacity for tenant control plane pods:

```bash
talosctl --nodes <new-node-ip> apply-config --file worker.yaml
kubectl get nodes  # Verify the node joined
```

## Scale etcd (Steward)

For deployments with many tenant clusters, configure a dedicated Steward DataStore:

```yaml
apiVersion: steward.butlerlabs.dev/v1alpha1
kind: DataStore
metadata:
  name: dedicated-etcd
spec:
  driver: etcd
  endpoints:
    - etcd-0.etcd:2379
    - etcd-1.etcd:2379
    - etcd-2.etcd:2379
```

At 10+ tenants, tune etcd resource limits and snapshot count. See the [etcd tuning guide](https://github.com/butlerdotdev/steward/blob/master/docs/etcd-tuning-for-multi-tenant.md) for details.

## Scale Tenant Clusters

### Scale Workers

```bash
butlerctl cluster scale my-cluster --workers 5 -n team-a
```

Or patch the TenantCluster directly:

```bash
kubectl patch tenantcluster my-cluster -n team-a \
  --type merge -p '{"spec":{"workers":{"replicas":5}}}'
```

### Resource Quotas

Limit per-team resource consumption with Team resource limits:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: backend-team
spec:
  resourceLimits:
    maxClusters: 10
    maxTotalNodes: 100
```

Butler enforces these limits when creating or scaling TenantClusters.

## See Also

- [Concepts > Management Clusters](../concepts/management-clusters.md) -- Topology options
- [Concepts > Multi-Tenancy](../concepts/multi-tenancy.md) -- Team quotas
