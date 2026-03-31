---
title: Edit Cluster Spec
sidebar_position: 3
---

# Edit Cluster Spec

After a TenantCluster is created, you can modify its specification to change the Kubernetes version, control plane sizing, and worker machine template. Butler reconciles spec changes into the underlying infrastructure automatically.

## What Can Be Edited

| Field | Path | Effect |
|-------|------|--------|
| Kubernetes version | `spec.kubernetesVersion` | Steward rolls the control plane deployment. No downgrades. |
| CP replicas | `spec.controlPlane.replicas` | Steward scales the control plane (1 or 3). |
| CP resources | `spec.controlPlane.resources` | Steward updates apiserver/controller-manager/scheduler resource requests and limits. |
| Worker replicas | `spec.workers.replicas` | CAPI scales the MachineDeployment. See [Scale](./scaling.md). |
| Worker CPU | `spec.workers.machineTemplate.cpu` | Creates a new MachineTemplate and triggers a CAPI rolling update. |
| Worker memory | `spec.workers.machineTemplate.memory` | Same rolling update. |
| Worker disk | `spec.workers.machineTemplate.diskSize` | Same rolling update. |
| Infrastructure overrides | `spec.infrastructureOverride` | Platform admin only. Provider-specific overrides (image, network, storage). |

## Console

Open the cluster detail page and click **Edit**. The modal shows editable fields with current values pre-filled. Control plane replicas are only visible to admins.

Reducing CP replicas from 3 to 1 requires checking an explicit acknowledgment checkbox (loses HA).

## CLI

```bash
kubectl patch tenantcluster my-cluster -n team-a --type merge -p '{
  "spec": {
    "kubernetesVersion": "v1.32.0",
    "controlPlane": {
      "replicas": 3,
      "resources": {
        "apiServer": {
          "requests": {"cpu": "1", "memory": "1Gi"},
          "limits": {"cpu": "2", "memory": "2Gi"}
        }
      }
    },
    "workers": {
      "machineTemplate": {
        "cpu": 8,
        "memory": "32Gi",
        "diskSize": "200Gi"
      }
    }
  }
}'
```

Or use the server API:

```bash
curl -X PUT https://console.example.com/api/clusters/team-a/my-cluster \
  -H "Content-Type: application/json" \
  -d '{
    "resourceVersion": "12345",
    "kubernetesVersion": "v1.32.0"
  }'
```

The API uses optimistic concurrency via `resourceVersion`. On conflict (409), re-fetch the cluster and retry.

## How It Works

### Control Plane Changes

The butler-controller detects drift between the TenantCluster spec and the live StewardControlPlane. When version, replicas, or resources differ, it patches the StewardControlPlane using a JSON merge patch. Steward then rolls the control plane deployment.

The cluster phase stays `Ready` during upgrades. The `ControlPlaneReady` condition reflects the rollout state.

### Worker Machine Template Changes

CPU, memory, and disk changes trigger a rolling update:

1. The controller creates a new immutable MachineTemplate (e.g., `my-cluster-worker-v2`) with the updated specs.
2. The controller patches the MachineDeployment's `infrastructureRef` to point to the new template.
3. CAPI performs a rolling update: creates new VMs, drains and deletes old VMs.

This is supported for Harvester (KubevirtMachineTemplate) and Nutanix (NutanixMachineTemplate). Cloud provider templates are not yet supported for tenant clusters.

### Validation

The server validates edits before applying:

- Kubernetes version downgrades are rejected.
- CP replicas must be 1 or 3 (etcd quorum requirement).
- Worker replicas must be between 1 and 100.
- Worker CPU must be at least 1.
- Infrastructure overrides require platform admin privileges.

Validation errors are returned as structured field errors with the field path and reason.

## Limitations

- Edits are only allowed when the cluster is in `Ready` or `Pending` phase. Clusters in `Provisioning`, `Installing`, or `Deleting` phase must stabilize first.
- Removing CP resource limits after they are set is not supported (merge patch cannot express key deletion).
- Cloud provider tenant clusters (AWS, Azure, GCP) do not support machine template rolling updates yet.

## See Also

- [Scale](./scaling.md) -- Worker replica scaling
- [Concepts > Tenant Clusters](../concepts/tenant-clusters.md) -- Lifecycle and architecture
