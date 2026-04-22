---
title: Networking and IPAM
sidebar_position: 9
---

# Networking and IPAM

Butler manages IP address allocation for on-prem tenant clusters. Cloud provider clusters skip IPAM entirely because the cloud handles networking.

## Two Networking Modes

### IPAM Mode (On-Prem)

For Harvester, Nutanix, and Proxmox deployments, Butler allocates contiguous IP ranges from administrator-defined pools. Each tenant cluster receives:

- **Node IPs** -- Addresses for worker VMs.
- **LoadBalancer IPs** -- Addresses for MetalLB to assign to Services of type LoadBalancer.

Butler configures MetalLB on each tenant cluster with the allocated LoadBalancer range.

### Cloud Mode

For AWS, GCP, and Azure deployments, Butler delegates networking to the cloud provider. VMs get IPs from VPC subnets. LoadBalancer Services use cloud load balancers. No NetworkPool or IPAllocation resources are needed.

Set the mode in `ProviderConfig.spec.network.mode` (`ipam` or `cloud`).

## NetworkPool

Platform admins define IP address pools as NetworkPool resources:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: NetworkPool
metadata:
  name: lab-pool
  namespace: butler-system
spec:
  cidr: "10.40.0.0/21"
  reserved:
    - cidr: "10.40.0.0/28"
      description: "Management cluster"
  tenantAllocation:
    start: "10.40.1.0"
    end: "10.40.7.254"
    defaults:
      nodesPerTenant: 5
      lbPoolPerTenant: 8
```

The pool defines the total allocatable range, reserved ranges (management cluster, infrastructure), and default allocation sizes per tenant.

## IPAllocation

Butler creates IPAllocation resources automatically when provisioning a tenant cluster. Each allocation records the assigned IP range, the source pool, and the tenant cluster reference.

Allocations follow a lifecycle: `Pending` -> `Allocated` -> `Released`. Released allocations are retained for audit before garbage collection.

## Elastic Scaling

Butler supports two load balancer allocation modes:

- **Static** -- Allocate a fixed pool of IPs at cluster creation.
- **Elastic** -- Start with a small pool and grow it automatically as the tenant creates more LoadBalancer Services.

Configure the mode in `ProviderConfig.spec.network.loadBalancer.allocationMode`.

## See Also

- [Architecture > IPAM Internals](../architecture/ipam.md) -- Bitmap allocator, garbage collection, and elastic scaling algorithm
- [NetworkPool Reference](../reference/crds/networkpool.md) -- Full NetworkPool specification
- [IPAllocation Reference](../reference/crds/ipallocation.md) -- Full IPAllocation specification
- [ProviderConfig Reference](../reference/crds/providerconfig.md) -- Network configuration fields
