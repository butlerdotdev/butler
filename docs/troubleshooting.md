---
title: Troubleshooting
sidebar_position: 99
---

Common issues and how to resolve them.

## Diagnostic Commands

Before troubleshooting, gather diagnostic information:

```bash
# Platform status
butleradm status

# Controller logs
kubectl logs -n butler-system deploy/butler-controller --tail=100

# All Butler pods
kubectl get pods -n butler-system

# TenantCluster status
kubectl describe tenantcluster <name> -n <namespace>

# Events in namespace
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

---

## Bootstrap Issues

### Bootstrap Hangs at "Creating KIND Cluster"

**Symptoms:**
- `butleradm bootstrap` stops responding
- No KIND cluster appears

**Diagnosis:**
```bash
# Check Docker is running
docker ps

# Check for existing KIND clusters
kind get clusters
```

**Causes and Solutions:**

1. **Docker not running**
   ```bash
   # Start Docker
   sudo systemctl start docker  # Linux
   open -a Docker               # macOS
   ```

2. **Stale KIND cluster**
   ```bash
   # Delete old cluster
   kind delete cluster --name butler-bootstrap

   # Retry bootstrap
   butleradm bootstrap harvester --config bootstrap.yaml
   ```

### Bootstrap Fails with "Cannot Connect to Provider"

**Symptoms:**
- Error: `failed to connect to Harvester API`
- Error: `Prism Central connection refused`

**Diagnosis:**
```bash
# Test Harvester connectivity
kubectl --kubeconfig=harvester-kubeconfig.yaml get nodes

# Test Nutanix connectivity
curl -k https://prism-central:9440/api/nutanix/v3/clusters
```

**Causes and Solutions:**

1. **Network connectivity**
   - Verify firewall rules allow access
   - Check VPN connection if required

2. **Invalid credentials**
   - Verify kubeconfig/credentials file
   - Check credentials have not expired

3. **Wrong API endpoint**
   - Update server URL in kubeconfig
   - For Harvester, ensure external API address is used

---

## Cluster Provisioning Issues

### Cluster Stuck in "Provisioning"

**Symptoms:**
- TenantCluster shows `Provisioning` phase for >10 minutes
- No control plane pods appearing

**Diagnosis:**
```bash
# Check TenantControlPlane
kubectl get tenantcontrolplane -A

# Check Steward controller logs
kubectl logs -n steward-system deploy/steward-controller --tail=100

# Check DataStore status
kubectl get datastore -A
```

**Causes and Solutions:**

1. **DataStore unavailable**
   ```bash
   # Check DataStore status
   kubectl describe datastore default

   # Check etcd pods
   kubectl get pods -n steward-system -l app=etcd
   ```

2. **Certificate issues**
   - Check cert-manager is running
   - Verify Certificate resources are Ready

3. **Resource constraints**
   - Ensure management cluster has capacity for control plane pods

### Workers Not Joining Cluster

**Symptoms:**
- Control plane is ready
- Workers not joining

**Diagnosis:**
```bash
# Check CAPI resources
kubectl get cluster,machinedeployment,machine -A

# Check Machine status
kubectl describe machine -l cluster.x-k8s.io/cluster-name=<cluster>

# Check provider controller
kubectl logs -n butler-system deploy/butler-provider-<provider>
```

**Causes and Solutions:**

1. **VMs not starting**
   ```bash
   # Check VM status in infrastructure
   # For Harvester:
   kubectl --kubeconfig harvester.yaml get virtualmachine -A
   ```

2. **Bootstrap failing**
   - Check cloud-init logs on worker: `/var/log/cloud-init-output.log`
   - Verify network connectivity to API server

3. **Node cannot reach API server**
   - Check LoadBalancer service for control plane
   - Verify firewall rules

### Cluster Stuck in "Installing"

**Symptoms:**
- Workers are ready
- Addons not installing

**Diagnosis:**
```bash
# Check TenantAddon resources
kubectl get tenantaddon -n <cluster-namespace>
kubectl describe tenantaddon <name> -n <cluster-namespace>

# Check controller logs
kubectl logs -n butler-system deploy/butler-controller | grep -i addon
```

**Causes and Solutions:**

1. **Helm chart not found**
   - Verify AddonDefinition chart repository
   - Check network access to chart repository

2. **Invalid values**
   - Check TenantAddon events for Helm errors
   - Validate addon values against schema

---

## Addon Issues

### Cilium Not Starting

**Symptoms:**
- Pods stuck in `Pending`
- Cilium agent not running

**Diagnosis:**
```bash
# On tenant cluster
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl describe pod -n kube-system -l k8s-app=cilium
```

**Causes and Solutions:**

1. **Missing kernel modules**
   - Ensure nodes have eBPF support
   - Check Talos/OS version compatibility

2. **Resource constraints**
   - Increase node resources
   - Reduce Cilium resource requests

### MetalLB Not Assigning IPs

**Symptoms:**
- LoadBalancer services stuck in `Pending`
- No external IP assigned

**Diagnosis:**
```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check IPAddressPool
kubectl get ipaddresspool -n metallb-system
```

**Causes and Solutions:**

1. **No IP pool configured**
   ```yaml
   # Ensure addressPool is set in addon values
   addons:
     - name: metallb
       values:
         addressPool: 10.40.1.100-10.40.1.120
   ```

2. **IP pool exhausted**
   - Expand the address pool
   - Delete unused LoadBalancer services

---

## Access Issues

### Cannot Get Kubeconfig

**Symptoms:**
- `butlerctl cluster kubeconfig` fails
- Secret not found

**Causes and Solutions:**

1. **Cluster not ready**
   - Wait for cluster to reach `Running` phase

2. **RBAC permissions**
   - Verify user has access to the namespace
   - Check team membership

### Console Not Loading

**Symptoms:**
- Butler console shows blank page
- API errors in browser console

**Diagnosis:**
```bash
# Check console pods
kubectl get pods -n butler-system -l app=butler-console

# Check server pods
kubectl get pods -n butler-system -l app=butler-server
```

---

## Getting Help

### Collect Support Bundle

```bash
# Collect diagnostics
butleradm support-bundle --output butler-support.tar.gz
```

This collects:
- Pod logs from butler-system
- CRD state
- Event logs
- Node status

### Community Support

- [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions)
- [GitHub Issues](https://github.com/butlerdotdev/butler/issues)

### Enable Debug Logging

```bash
# Edit controller deployment
kubectl edit deploy butler-controller -n butler-system

# Add environment variable
env:
  - name: LOG_LEVEL
    value: debug
```

---

## See Also

- [Operations Guide](./operations/) - Day-2 operations
- [Architecture](./architecture/) - System design
- [CLI Reference](./reference/cli/) - Command documentation
