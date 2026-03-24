---
title: Cluster Provisioning
sidebar_position: 3
---

# Troubleshoot Cluster Provisioning

## Cluster Stuck in "Provisioning"

**Diagnosis:**

```bash
kubectl get tenantcontrolplane -A
kubectl logs -n steward-system deploy/steward-controller --tail=100
kubectl get datastore -A
```

**Solutions:**

1. **DataStore unavailable** -- Check the etcd pods backing the DataStore:
   ```bash
   kubectl describe datastore default
   kubectl get pods -n steward-system -l app=etcd
   ```

2. **Certificate issues** -- Verify cert-manager is running and Certificate resources are in Ready state.

3. **Resource constraints** -- Ensure the management cluster has enough CPU and memory for control plane pods. Each TenantControlPlane requires ~12 mCPU and ~6 MiB at idle, more under load.

## Workers Not Joining

**Diagnosis:**

```bash
kubectl get cluster,machinedeployment,machine -A
kubectl describe machine -l cluster.x-k8s.io/cluster-name=<cluster-name>
kubectl logs -n butler-system deploy/butler-provider-<provider>
```

**Solutions:**

1. **VMs not starting** -- Check VM status on the infrastructure:
   ```bash
   # Harvester
   kubectl --kubeconfig harvester.yaml get virtualmachine -A
   ```

2. **Bootstrap failing on the node** -- For kubeadm-based OS types (Rocky), check cloud-init logs on the worker: `/var/log/cloud-init-output.log`. For Talos, check `talosctl dmesg` on the node IP.

3. **Node cannot reach API server** -- Verify the control plane LoadBalancer Service has an IP and the worker node can reach it on port 6443.

## Cluster Stuck in "Installing"

Workers are ready but addons fail to install.

**Diagnosis:**

```bash
kubectl get tenantaddon -n <cluster-namespace>
kubectl describe tenantaddon <name> -n <cluster-namespace>
kubectl logs -n butler-system deploy/butler-controller | grep addon
```

**Solutions:**

1. **Helm chart not found** -- Verify the AddonDefinition chart repository URL is accessible from the management cluster.
2. **Invalid values** -- Check TenantAddon events for Helm rendering errors. Validate addon values against the chart's values schema.
3. **Timeout** -- Cilium and Traefik require working LoadBalancer Services. Ensure MetalLB has an address pool allocated. See [Networking Troubleshooting](./networking.md).

## Kubeconfig Not Available

**Symptoms:** `butlerctl cluster kubeconfig` fails with "secret not found."

**Solutions:**

1. **Cluster not ready** -- The kubeconfig Secret is created when the cluster reaches the `Ready` phase. Wait for provisioning to complete.
2. **RBAC permissions** -- Verify your user has access to the team namespace containing the cluster.
