---
title: Create Your First Tenant Cluster
sidebar_position: 8
---

# Create Your First Tenant Cluster

After bootstrapping a management cluster, create a tenant cluster to run workloads.

## 1. Set Your Kubeconfig

Point kubectl at the management cluster. The kubeconfig path comes from your bootstrap config's `cluster.name` field.

```bash
export KUBECONFIG=~/.butler/<cluster-name>-kubeconfig
kubectl get nodes  # Verify connectivity
```

## 2. Create a Team

Teams provide multi-tenancy isolation. Each team gets a dedicated namespace for its clusters.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: dev-team
spec:
  displayName: "Development Team"
  access:
    users:
      - name: you@example.com
        role: admin
```

```bash
kubectl apply -f team.yaml
```

## 3. Create a TenantCluster

### Option A: CLI

```bash
butlerctl cluster create my-first-cluster \
  --namespace dev-team \
  --workers 2 \
  --k8s-version v1.30.2
```

### Option B: YAML

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantCluster
metadata:
  name: my-first-cluster
  namespace: dev-team
spec:
  kubernetesVersion: "v1.30.2"
  providerConfigRef:
    name: default
  controlPlane:
    replicas: 1
  workers:
    replicas: 2
  addons:
    cni:
      provider: cilium
      version: "1.17.0"
    loadBalancer:
      provider: metallb
      version: "0.14.9"
```

```bash
kubectl apply -f tenant-cluster.yaml
```

**Addon notes**: Every cluster needs a CNI (Cilium) to enable pod networking. On-prem clusters also need a LoadBalancer provider (MetalLB) for Services of type LoadBalancer. Cloud clusters get cloud load balancers automatically.

**On-prem networking**: If your ProviderConfig uses IPAM mode, Butler allocates IP ranges from your NetworkPool automatically. If you use static addressing, add a `networking.loadBalancerPool` to the TenantCluster spec.

## 4. Watch Progress

```bash
kubectl get tenantcluster my-first-cluster -n dev-team -w
```

The cluster transitions through phases: `Pending` -> `Provisioning` -> `Installing` -> `Ready`.

Provisioning time varies by infrastructure provider and load. If the cluster stays in `Provisioning` for more than 10 minutes, check [Troubleshooting](../troubleshooting/provisioning.md).

## 5. Get the Kubeconfig

```bash
butlerctl cluster kubeconfig my-first-cluster -n dev-team > my-first-cluster.yaml
```

## 6. Use Your Cluster

```bash
export KUBECONFIG=my-first-cluster.yaml

kubectl get nodes
kubectl get pods -A
```

You should see worker nodes in `Ready` state, Cilium pods running on each node, and CoreDNS pods running.

## Next Steps

- **[Butler Console](./console.md)** -- Manage clusters from the web UI.
- **[Concepts](../concepts/)** -- Understand hosted control planes, multi-tenancy, and addons.
- **[TenantCluster Reference](../reference/crds/tenantcluster.md)** -- Full specification for all TenantCluster fields.
- **[Operations](../operations/)** -- Upgrade, monitor, and scale your clusters.
