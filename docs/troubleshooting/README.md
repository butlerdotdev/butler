---
title: Troubleshooting
sidebar_position: 1
---

# Troubleshooting

## Diagnostic Commands

Gather diagnostic information before investigating issues:

```bash
# Platform status
butleradm status

# Controller logs
kubectl logs -n butler-system deploy/butler-controller --tail=100

# All Butler pods
kubectl get pods -n butler-system

# TenantCluster status and conditions
kubectl describe tenantcluster <name> -n <namespace>

# Recent events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## Issues by Area

| Area | Common Problems |
|------|-----------------|
| [Bootstrap](./bootstrap.md) | KIND cluster hangs, provider connectivity, VM provisioning failures |
| [Cluster Provisioning](./provisioning.md) | Stuck in Provisioning, workers not joining, control plane issues |
| [Networking](./networking.md) | MetalLB not assigning IPs, IPAM allocation failures, DNS issues |
| [Addons](./addons.md) | Cilium not starting, Helm chart failures, addon stuck in Installing |

## Enable Debug Logging

```bash
kubectl set env deploy/butler-controller -n butler-system LOG_LEVEL=debug
```

## Get Help

- [GitHub Issues](https://github.com/butlerdotdev/butler/issues)
- [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions)
