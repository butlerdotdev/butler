---
title: Networking
sidebar_position: 4
---

# Troubleshoot Networking

## MetalLB Not Assigning IPs

**Symptoms:** LoadBalancer Services stuck in `Pending` with no external IP.

**Diagnosis:**

```bash
# On the tenant cluster
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

**Solutions:**

1. **No IP pool configured** -- Verify the TenantCluster has a `loadBalancerPool` or that IPAM allocated a range. Check IPAllocation:
   ```bash
   kubectl get ipallocation -n butler-system -l butler.butlerlabs.dev/tenant=<cluster-name>
   ```

2. **IP pool exhausted** -- All IPs in the MetalLB pool are assigned. For elastic IPAM, Butler detects the Pending Service and creates a growth allocation automatically. If growth does not fire, check whether the per-tenant quota (`maxLoadBalancerIPs`) has been reached, or whether the controller can reach the tenant API server.

3. **L2 advertisement missing** -- MetalLB L2 mode requires an L2Advertisement resource matching the IPAddressPool. Butler creates this automatically; if missing, check the butler-controller logs.

4. **MetalLB pool drift** -- If the MetalLB `default-pool` on the tenant does not match the management-side IPAllocations, the controller corrects it automatically on the next reconcile via server-side apply. Manual edits to `default-pool` are overwritten. Operators who need custom pools should create additional IPAddressPool resources with different names.

## IPAM Allocation Failures

**Symptoms:** IPAllocation stuck in `Pending` or transitions to `Failed`.

**Diagnosis:**

```bash
kubectl describe ipallocation -n butler-system -l butler.butlerlabs.dev/tenant=<cluster-name>
kubectl describe networkpool -n butler-system <pool-name>
```

**Solutions:**

1. **Pool exhausted** -- Check the NetworkPool status for `allocatedIPs` vs `totalIPs`. Check the `CapacityExhausted` condition on the pool. Create a new NetworkPool or expand the CIDR.
2. **Range conflict** -- The requested range overlaps with an existing allocation or reserved range. Check the NetworkPool's `reserved` list.
3. **Quota reached** -- The tenant has reached `quotaPerTenant.maxLoadBalancerIPs`. Check the ProviderConfig for quota settings and the total allocated IPs for the tenant:
   ```bash
   kubectl get ipallocation -n butler-system \
     -l butler.butlerlabs.dev/tenant=<cluster-name> \
     -o custom-columns='NAME:.metadata.name,COUNT:.spec.count,PHASE:.status.phase'
   ```

## Growth Allocation Not Firing

**Symptoms:** Tenant LB Service stuck Pending, but no growth IPAllocation created.

**Diagnosis:**

```bash
# Confirm elastic IPAM mode
kubectl get providerconfig <name> -n butler-system \
  -o jsonpath='{.spec.network.loadBalancer.allocationMode}'

# Check quota vs current allocations
kubectl get providerconfig <name> -n butler-system \
  -o jsonpath='{.spec.network.quotaPerTenant.maxLoadBalancerIPs}'
kubectl get ipallocation -n butler-system \
  -l butler.butlerlabs.dev/tenant=<cluster-name>

# Check controller logs for the tenant
kubectl logs -n butler-system -l app.kubernetes.io/name=butler-controller --tail=100 \
  | grep <cluster-name>
```

**Solutions:**

1. **Not elastic mode** -- Growth only fires with `allocationMode: elastic`. Static mode allocates once at creation.
2. **Quota reached** -- Total allocated IPs equal `maxLoadBalancerIPs`. Increase the quota or delete unused Services.
3. **Service too new** -- The controller waits 30 seconds after Service creation before treating it as a demand signal. Wait and check again.
4. **Tenant API unreachable** -- The controller skips elastic IPAM when it cannot connect to the tenant. Check tenant control plane health.

## Console Not Loading

**Symptoms:** Blank page or API errors in the browser.

**Diagnosis:**

```bash
kubectl get pods -n butler-system -l app=butler-console
kubectl get pods -n butler-system -l app=butler-server
kubectl logs -n butler-system deploy/butler-server --tail=50
```

**Solutions:**

1. **Server pod not running** -- Check butler-server pod events for crash reasons.
2. **Ingress misconfiguration** -- Verify the ingress or LoadBalancer Service for the console has an external IP.
3. **CORS errors** -- If accessing the console from a different hostname than expected, check the butler-server CORS configuration.
