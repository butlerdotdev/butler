---
title: Bootstrap
sidebar_position: 2
---

# Troubleshoot Bootstrap

## Bootstrap Hangs at "Creating KIND Cluster"

**Diagnosis:**

```bash
docker ps            # Verify Docker is running
kind get clusters    # Check for stale clusters
```

**Solutions:**

1. **Docker not running** -- Start Docker:
   ```bash
   sudo systemctl start docker  # Linux
   open -a Docker               # macOS
   ```

2. **Stale KIND cluster** -- Delete and retry:
   ```bash
   kind delete cluster --name butler-bootstrap
   butleradm bootstrap <provider> --config bootstrap.yaml
   ```

## Cannot Connect to Provider

**Symptoms:** `failed to connect to Harvester API` or `Prism Central connection refused`

**Diagnosis:**

```bash
# Harvester
kubectl --kubeconfig=harvester-kubeconfig.yaml get nodes

# Nutanix
curl -k https://prism-central:9440/api/nutanix/v3/clusters

# Cloud (AWS example)
aws sts get-caller-identity
```

**Solutions:**

1. **Network connectivity** -- Verify firewall rules allow access from your machine to the provider API. Check VPN if required.
2. **Invalid credentials** -- Verify the credentials file has not expired. For Harvester, regenerate the kubeconfig. For cloud providers, refresh access keys.
3. **Wrong API endpoint** -- Confirm the endpoint URL in the bootstrap config matches the provider's external API address.

## Re-run with Debug Output

Keep the temporary KIND cluster for inspection:

```bash
butleradm bootstrap <provider> --config <path> --skip-cleanup

# Inspect state in the KIND cluster
kubectl --context kind-butler-bootstrap get clusterbootstrap -n butler-system
kubectl --context kind-butler-bootstrap get machinerequest -n butler-system
kubectl --context kind-butler-bootstrap logs -n butler-system deploy/butler-bootstrap-controller
```
