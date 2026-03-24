---
title: Backup and Restore
sidebar_position: 4
---

# Backup and Restore

## What to Back Up

| Component | Method | Frequency |
|-----------|--------|-----------|
| etcd | Velero or etcd snapshot | Daily |
| Secrets | Sealed Secrets backup | On change |
| CRDs/CRs | GitOps repository | Continuous |
| Kubeconfigs | Secure storage | On creation |

## Using Velero

```bash
# Install Velero
velero install --provider aws --bucket butler-backups ...

# Create a daily backup schedule
velero schedule create butler-daily \
  --schedule="0 2 * * *" \
  --include-namespaces butler-system \
  --include-cluster-resources=true
```

## Manual etcd Backup (Talos)

```bash
talosctl -n <control-plane-ip> etcd snapshot db.snapshot
```

Store the snapshot in a secure location outside the management cluster.

## Restore Process

1. Restore etcd from the snapshot.
2. Apply CRDs from the butler-crds Helm chart.
3. Apply CR state from your GitOps repository.
4. Verify tenant cluster connectivity with `butlerctl cluster list`.
