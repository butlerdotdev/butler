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

## Platform Addon Install Failed

**Symptoms:** `AddonsReady` condition is `False` with reason `AddonInstallFailed`. The cluster reaches `Ready` phase but has one or more missing platform addons (cert-manager, longhorn, metallb, traefik).

**Diagnosis:**

```bash
# Check which addons failed
kubectl get tc <cluster-name> -n <namespace> -o jsonpath='{.status.observedState.addons}' | jq .

# Check condition message for the failure list
kubectl get tc <cluster-name> -n <namespace> -o jsonpath='{.status.conditions[?(@.type=="AddonsReady")]}'

# Check events for specific errors
kubectl get events -n <namespace> --field-selector involvedObject.name=<cluster-name> \
  --sort-by='.lastTimestamp' | grep AddonInstallFailed
```

**Solutions:**

1. **Transient failure (network, timeout)** -- The controller retries failed addons automatically during steady-state reconciliation. Check events for `AddonRetrySucceeded` to confirm self-healing. The retry interval is capped at 10 minutes for clusters with failed addons.
2. **Helm chart repository unreachable** -- The management cluster needs network access to the Helm chart repositories. Check DNS and firewall rules from the controller pod.
3. **Tenant API server unreachable** -- Platform addon installs require a working kubeconfig for the tenant cluster. Verify the control plane is healthy and the admin-kubeconfig secret exists in the tenant namespace.
4. **MetalLB without IP allocation** -- MetalLB install requires an allocated IP pool. If the IPAllocation is still `Pending`, MetalLB install will fail and retry once the allocation is fulfilled.

**Note:** Cilium (CNI) failures are fatal and block cluster progression. Non-CNI addon failures (cert-manager, longhorn, metallb, traefik) allow the cluster to reach Ready phase. The steady-state loop retries these on each reconcile.

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
