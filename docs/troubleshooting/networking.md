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

2. **IP pool exhausted** -- Expand the address pool or delete unused LoadBalancer Services. For elastic IPAM, Butler automatically requests additional IPs.

3. **L2 advertisement missing** -- MetalLB L2 mode requires an L2Advertisement resource matching the IPAddressPool. Butler creates this automatically; if missing, check the butler-controller logs.

## IPAM Allocation Failures

**Symptoms:** IPAllocation stuck in `Pending` or transitions to `Failed`.

**Diagnosis:**

```bash
kubectl describe ipallocation -n butler-system -l butler.butlerlabs.dev/tenant=<cluster-name>
kubectl describe networkpool -n butler-system <pool-name>
```

**Solutions:**

1. **Pool exhausted** -- Check the NetworkPool status for `allocatedIPs` vs `totalIPs`. Create a new NetworkPool or expand the CIDR.
2. **Range conflict** -- The requested range overlaps with an existing allocation or reserved range. Check the NetworkPool's `reserved` list.

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
