---
title: Addons
sidebar_position: 5
---

# Troubleshoot Addons

## Cilium Not Starting

**Symptoms:** Pods stuck in `Pending`, Cilium agent not running on nodes.

**Diagnosis (on tenant cluster):**

```bash
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl describe pod -n kube-system -l k8s-app=cilium
```

**Solutions:**

1. **Missing kernel modules** -- Cilium requires eBPF support. Verify the node OS version supports eBPF (Talos 1.7+ and all supported OS types include it).
2. **Resource constraints** -- Cilium agent requests CPU and memory on each node. If nodes are undersized, increase `workers.machineTemplate.cpu` and `workers.machineTemplate.memory` in the TenantCluster spec.

## Addon Stuck in "Installing"

**Diagnosis:**

```bash
kubectl get tenantaddon -n <namespace>
kubectl describe tenantaddon <addon-name> -n <namespace>
```

**Solutions:**

1. **Helm chart repository unreachable** -- The management cluster needs network access to the Helm chart repository URL in the AddonDefinition. Check DNS resolution and firewall rules.
2. **Invalid chart values** -- Check TenantAddon events for Helm template rendering errors. Common causes: incorrect version strings, missing required values.
3. **Dependency not met** -- Some addons depend on others (Traefik requires MetalLB for its LoadBalancer Service). Check `spec.dependsOn` in the TenantAddon.

## Addon Health Degraded

**Diagnosis:**

```bash
# Check addon status
kubectl get tenantaddon <name> -n <namespace> -o yaml

# Check Helm release on tenant cluster
KUBECONFIG=<tenant-kubeconfig> helm list -A
```

**Solutions:**

1. **Pod crash loops** -- Investigate pod logs on the tenant cluster. Common causes: resource limits too low, image pull failures, misconfigured values.
2. **Version incompatibility** -- Verify the addon version is compatible with the tenant cluster's Kubernetes version.
