---
title: IPAM Operations
sidebar_position: 7
---

# IPAM Operations

This guide covers day-to-day IPAM operations: monitoring pool capacity, understanding allocation behavior during tenant provisioning, and planning for pool expansion. For the IPAM design and implementation details, see [Architecture: IPAM Internals](../architecture/ipam.md).

## Pool Capacity Planning

### Reading Pool State

```bash
# Overview of all pools
kubectl get networkpool -n butler-system

# Detailed status including capacity conditions
kubectl get networkpool -n butler-system -o custom-columns=\
'NAME:.metadata.name,TOTAL:.status.totalIPs,ALLOC:.status.allocatedIPs,AVAIL:.status.availableIPs,COUNT:.status.allocationCount,FRAG:.status.fragmentationPercent'
```

### Capacity Conditions

Every NetworkPool has three always-present conditions that surface utilization tiers:

| Condition | Threshold | What to do |
|-----------|-----------|------------|
| `CapacityWarning` | 70% | Start planning pool expansion. Current workloads are not affected. |
| `CapacityCritical` | 85% | Expansion is urgent. New tenant creation may fail if a large allocation is requested. |
| `CapacityExhausted` | 95% | The pool is effectively full. New allocations will likely fail. Existing tenants continue operating. |

```bash
# Check conditions across all pools
kubectl get networkpool -n butler-system -o custom-columns=\
'NAME:.metadata.name,WARN:.status.conditions[?(@.type=="CapacityWarning")].status,CRIT:.status.conditions[?(@.type=="CapacityCritical")].status,EXHAUSTED:.status.conditions[?(@.type=="CapacityExhausted")].status'
```

The NetworkPool controller also emits Kubernetes events at each threshold crossing, rate-limited to one event per 10 minutes per tier. Recovery events fire when utilization drops back below a threshold.

```bash
# View capacity events
kubectl get events -n butler-system --field-selector involvedObject.kind=NetworkPool
```

### Worked Example

A production deployment runs a /23 pool (10.92.90.0/23) with 32 usable IPs after reserved ranges. The ProviderConfig uses elastic IPAM with `initialPoolSize: 2` and `growthIncrement: 1`.

With 8 tenant clusters, each holding an initial allocation of 2 IPs:

```
Total IPs:     32
Allocated:     16 (8 tenants x 2 IPs)
Available:     16
Utilization:   50%
```

At this utilization, all three capacity conditions are `False` (below every threshold). The pool supports 8 more tenants at the current allocation rate before reaching the `CapacityWarning` threshold:

| Tenants | Allocated | Utilization | Conditions triggered |
|---------|-----------|-------------|---------------------|
| 8       | 16        | 50%         | None                |
| 12      | 24        | 75%         | `CapacityWarning`   |
| 14      | 28        | 87%         | `CapacityWarning`, `CapacityCritical` |
| 16      | 32        | 100%        | All three           |

This assumes no growth allocations. With elastic IPAM, tenants that run both a platform LB (Traefik) and a workload LB Service use both initial IPs. Tenants that run only Traefik leave 1 IP as idle headroom. In practice, 5 of 8 tenants in this deployment run at 100% utilization (both IPs in use), while 3 run at 50% (Traefik only). The actual pool usage is 13 IPs in use by Services plus 3 IPs as idle headroom.

Growth allocations consume additional IPs beyond the initial allocation. If a tenant with 2 initial IPs creates a third LB Service, demand-driven growth allocates 1 more IP from the pool, moving the pool from 16 to 17 allocated. Growth allocations are released when the Service is deleted and the grace period expires (10 minutes).

### When to Expand

Expand the pool when `CapacityWarning` transitions to `True`. At 70% utilization, there is still room for several tenants, but the remaining capacity is shrinking. Expansion options:

1. **Expand the CIDR**: Increase the pool's `spec.cidr` to a larger subnet. The CIDR can grow but never shrink below existing allocations. A webhook enforces this.
2. **Add a secondary pool**: Create a new NetworkPool and add it to the ProviderConfig's `poolRefs` with a higher priority number (lower priority). New allocations fall through to the secondary pool when the primary is exhausted.
3. **Reclaim idle headroom**: If many tenants run at 50% utilization (Traefik only), reducing `initialPoolSize` from 2 to 1 for new tenants saves 1 IP per tenant. Existing allocations are unaffected. Demand-driven growth allocates the second IP when a workload LB Service goes Pending (measured latency: 37 seconds).

## Bootstrap Timing

When a new TenantCluster is created with elastic IPAM, operators can expect:

| Stage | Timing | What happens |
|-------|--------|--------------|
| TenantCluster create to IPAllocation Allocated | < 1 second | The TenantCluster controller creates the IPAllocation and the NetworkPool controller fulfills it in the same second. |
| TenantCluster create to Ready | ~3 minutes | Infrastructure provisioning, control plane setup, worker bootstrap, and addon installation. IPAM completes in the first second; the remaining time is provider and bootstrap work. |
| Demand-driven growth (Pending Service to IP assigned) | ~37 seconds | The controller detects the Pending Service, creates a growth IPAllocation, the NetworkPool controller fulfills it, MetalLB is updated, and MetalLB assigns the IP to the Service. |
| Demand-driven shrink (Service deleted to allocation released) | ~10 minutes | The controller detects the IP is unused and waits for the 10-minute grace period before releasing the growth allocation. |

### What a fresh tenant looks like

On bootstrap, a fresh tenant cluster running the default addon set (Cilium, MetalLB, cert-manager, Longhorn, Traefik) has one LB Service: Traefik. This consumes 1 of the initially allocated IPs.

If Traefik failed to install during initial addon setup, the cluster reaches Ready phase but the `AddonsReady` condition is `False`. The controller retries the install during steady-state reconciliation. Until Traefik is installed, no platform LB Service exists on the tenant, and the allocated IPs sit unused. See [Troubleshooting: Addons](../troubleshooting/addons.md#platform-addon-install-failed) for diagnosis.

```bash
# On a fresh tenant cluster
kubectl --kubeconfig <tenant-kubeconfig> get services -A --field-selector spec.type=LoadBalancer
```

Typical output:

```
NAMESPACE   NAME      TYPE           EXTERNAL-IP
traefik     traefik   LoadBalancer   10.92.90.49
```

The MetalLB pool on the tenant matches the management-side allocation:

```bash
kubectl --kubeconfig <tenant-kubeconfig> get ipaddresspool -n metallb-system default-pool -o jsonpath='{.spec.addresses}'
```

```
["10.92.90.49-10.92.90.50"]
```

## Monitoring Allocations

### Per-tenant allocation summary

```bash
# All allocations with role labels
kubectl get ipallocation -n butler-system \
  -o custom-columns='NAME:.metadata.name,TENANT:.metadata.labels.butler\.butlerlabs\.dev/tenant,ROLE:.metadata.labels.butler\.butlerlabs\.dev/allocation-role,PHASE:.status.phase,COUNT:.spec.count,RANGE:.status.cidr'
```

### Finding growth allocations

Growth allocations indicate that tenants have exceeded their initial pool. Under demand-driven IPAM, this is normal and expected. Zero growth allocations at rest (no Pending Services) indicates the system is stable.

```bash
# All growth allocations
kubectl get ipallocation -n butler-system \
  -l butler.butlerlabs.dev/allocation-role=growth

# Growth allocations for a specific tenant
kubectl get ipallocation -n butler-system \
  -l butler.butlerlabs.dev/tenant=<cluster-name>,butler.butlerlabs.dev/allocation-role=growth
```

### Verifying MetalLB sync

To confirm the tenant-side MetalLB pool matches management-side allocations:

```bash
# Management side: list allocated ranges for a tenant
kubectl get ipallocation -n butler-system \
  -l butler.butlerlabs.dev/tenant=<cluster-name> \
  -o custom-columns='NAME:.metadata.name,START:.status.startAddress,END:.status.endAddress'

# Tenant side: check MetalLB pool
kubectl --kubeconfig <tenant-kubeconfig> \
  get ipaddresspool -n metallb-system default-pool -o jsonpath='{.spec.addresses}'
```

The ranges should match. If they do not, the controller corrects the drift on the next reconcile via server-side apply.

## Operational Procedures

### Adding tenants to a near-capacity pool

When a pool is above 70% utilization (`CapacityWarning` is `True`):

1. Check how many IPs the new tenant needs: `initialPoolSize` (from ProviderConfig) plus any expected growth.
2. Compare against `status.availableIPs` on the pool.
3. If capacity is sufficient, create the TenantCluster normally. If not, expand the pool or add a secondary pool first.

```bash
kubectl get networkpool -n butler-system <pool-name> \
  -o jsonpath='Available: {.status.availableIPs}, Largest block: {.status.largestFreeBlock}'
```

The `largestFreeBlock` field matters when requesting large allocations. If 10 IPs are available but the largest contiguous block is 4, an allocation requesting 8 will fail even though total capacity exists.

### Manual allocation cleanup

In normal operation, the three-layer cleanup (TenantCluster deletion, IPAllocation finalizer, orphan GC) handles allocation lifecycle automatically. Manual cleanup is only needed if all three layers failed:

```bash
# Find orphaned allocations (tenant no longer exists)
kubectl get ipallocation -n butler-system \
  -l butler.butlerlabs.dev/tenant=<deleted-cluster-name>

# Delete them
kubectl delete ipallocation -n butler-system \
  -l butler.butlerlabs.dev/tenant=<deleted-cluster-name>
```

### Checking controller health

```bash
# Controller pods
kubectl get pods -n butler-system -l app.kubernetes.io/name=butler-controller

# Recent IPAM-related log entries
kubectl logs -n butler-system -l app.kubernetes.io/name=butler-controller --tail=100 \
  | grep -i -E "ipam|alloc|growth|shrink|metallb"
```

## See Also

- [Architecture: IPAM Internals](../architecture/ipam.md) -- Design, controllers, and allocation algorithms
- [Troubleshooting: Networking](../troubleshooting/networking.md) -- IPAM failure diagnosis
- [Concepts: Networking](../concepts/networking.md) -- IPAM modes overview
