---
title: butleradm
sidebar_position: 3
---

`butleradm` is the CLI for platform administrators to manage the Butler platform itself.

## Synopsis

```bash
butleradm [command] [flags]
```

## Global Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--kubeconfig` | | Path to kubeconfig file |
| `--context` | | Kubernetes context to use |
| `--output` | `-o` | Output format: `table`, `yaml`, `json` |
| `--help` | `-h` | Help for command |

---

## butleradm bootstrap

Bootstrap a new Butler management cluster.

### Common Flags

All `butleradm bootstrap <provider>` commands share the same flags. Provider-specific settings (credentials, regions, networks) are specified in the bootstrap config YAML file.

| Flag | Short | Description | Required |
|------|-------|-------------|----------|
| `--config` | `-c` | Path to bootstrap config file | Yes |
| `--dry-run` | | Show what would be created without executing | No |
| `--skip-cleanup` | | Don't delete KIND cluster on failure (for debugging) | No |
| `--local` | | Local development mode: build and load images from source | No |
| `--repo-root` | | Path to butlerdotdev repos (default: `~/code/github.com/butlerdotdev`) | No |

### butleradm bootstrap harvester

Bootstrap Butler on Harvester HCI.

```bash
butleradm bootstrap harvester [flags]
```

**Examples:**

```bash
# Bootstrap with config file
butleradm bootstrap harvester --config bootstrap.yaml

# Dry run to validate
butleradm bootstrap harvester --config bootstrap.yaml --dry-run
```

**Bootstrap Config Example:**

```yaml
apiVersion: bootstrap.butler.butlerlabs.dev/v1alpha1
kind: BootstrapConfig
metadata:
  name: butler-bootstrap
spec:
  # Cluster configuration
  cluster:
    name: butler-mgmt
    kubernetesVersion: "1.30.0"
    controlPlanes: 3
    workers: 3

  # Node configuration
  nodes:
    controlPlane:
      cpu: 4
      memory: 8Gi
      disk: 100Gi
    worker:
      cpu: 8
      memory: 16Gi
      disk: 200Gi

  # Network configuration
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    loadBalancerRange: 10.40.0.100-10.40.0.150

  # Storage
  storage:
    class: longhorn

  # Addons
  addons:
    cilium:
      enabled: true
    metallb:
      enabled: true
      addressPool: 10.40.0.100-10.40.0.150
    longhorn:
      enabled: true
    certManager:
      enabled: true
```

### butleradm bootstrap nutanix

Bootstrap Butler on Nutanix AHV.

```bash
butleradm bootstrap nutanix [flags]
```

**Examples:**

```bash
butleradm bootstrap nutanix --config bootstrap.yaml
```

### butleradm bootstrap proxmox

Bootstrap Butler on Proxmox VE.

> **Note**: Proxmox provider is planned. See [roadmap](../../roadmap/) for status.

```bash
butleradm bootstrap proxmox [flags]
```

### butleradm bootstrap gcp

Bootstrap Butler on Google Cloud Platform.

```bash
butleradm bootstrap gcp [flags]
```

**Examples:**

```bash
# Bootstrap on GCP
butleradm bootstrap gcp --config bootstrap.yaml

# Dry run to validate
butleradm bootstrap gcp --config bootstrap.yaml --dry-run
```

### butleradm bootstrap aws

Bootstrap Butler on Amazon Web Services.

```bash
butleradm bootstrap aws [flags]
```

**Examples:**

```bash
butleradm bootstrap aws --config bootstrap.yaml
```

### butleradm bootstrap azure

Bootstrap Butler on Microsoft Azure.

```bash
butleradm bootstrap azure [flags]
```

**Examples:**

```bash
butleradm bootstrap azure --config bootstrap.yaml
```

### Bootstrap Config Reference

The bootstrap config file is a ClusterBootstrap resource. See the [ClusterBootstrap CRD reference](../crds/clusterbootstrap.md) for the full spec. Key sections:

| Section | Purpose | On-Prem | Cloud |
|---------|---------|---------|-------|
| `spec.provider` | Infrastructure provider | `harvester`, `nutanix`, `proxmox` | `gcp`, `aws`, `azure` |
| `spec.cluster` | Topology, node sizing | Required | Required |
| `spec.network.vip` | Control plane VIP | Recommended (kube-vip requires it) | Not needed (LB endpoint used) |
| `spec.network.loadBalancerPool` | MetalLB IP range | Recommended | Not needed |
| `spec.talos` | Talos version, schematic, install disk | Required | Required |
| `spec.addons` | Platform addons | All available | kube-vip and MetalLB not installed |
| `spec.controlPlaneExposure` | Tenant API server exposure | Optional | Optional |

---

## butleradm status

Check Butler platform health.

```bash
butleradm status [flags]
```

**Examples:**

```bash
# Quick status check
butleradm status

# Detailed status
butleradm status -o yaml
```

**Output:**

```
Butler Platform Status
======================

Management Cluster: Healthy
  Kubernetes: v1.30.0
  Nodes: 6/6 Ready

Controllers:
  butler-controller:    Running (3 replicas)
  steward-controller:   Running (2 replicas)

DataStores:
  default (etcd):       Healthy (3 members)

Providers:
  harvester-prod:       Connected

Tenant Clusters: 12 total
  Running:    10
  Creating:    1
  Failed:      1
```

---

## butleradm provider

Manage infrastructure providers.

### butleradm provider list

List configured providers.

```bash
butleradm provider list
```

**Output:**

```
NAME             TYPE       STATUS      CLUSTERS   AGE
harvester-prod   harvester  Connected   8          30d
nutanix-dc1      nutanix    Connected   4          15d
```

### butleradm provider validate

Validate provider connectivity and permissions.

```bash
butleradm provider validate <name> [flags]
```

**Examples:**

```bash
# Validate provider
butleradm provider validate harvester-prod
```

**Output:**

```
Validating provider: harvester-prod (harvester)
  ✓ API connectivity
  ✓ Authentication
  ✓ Network access (vlan-100)
  ✓ Image available (ubuntu-22.04)
  ✓ Storage class (longhorn)
  ✓ Create VM permission
  ✓ Delete VM permission

Provider validation: PASSED
```

### butleradm provider add

Add a new provider configuration.

```bash
butleradm provider add <name> --type <type> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--type` | Provider type: `harvester`, `nutanix`, `proxmox` |
| `--kubeconfig` | Provider kubeconfig (Harvester) |
| `--endpoint` | API endpoint (Nutanix/Proxmox) |
| `--credentials` | Path to credentials file |
| `--namespace` | Namespace for ProviderConfig |

---

## butleradm team

Manage teams (admin operations).

### butleradm team list

List all teams.

```bash
butleradm team list
```

### butleradm team create

Create a new team.

```bash
butleradm team create <name> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--display-name` | Human-readable name |
| `--admin` | Initial admin user email |
| `--provider` | Default provider for team |

**Examples:**

```bash
butleradm team create backend \
  --display-name "Backend Team" \
  --admin admin@company.com \
  --provider harvester-prod
```

### butleradm team delete

Delete a team.

```bash
butleradm team delete <name> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--force` | Skip confirmation |
| `--cascade` | Delete all team clusters |

---

## butleradm user

Manage users.

### butleradm user list

List all users.

```bash
butleradm user list
```

### butleradm user create

Create a new user.

```bash
butleradm user create <email> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--name` | Display name |
| `--admin` | Grant platform admin role |
| `--send-invite` | Send invitation email |

### butleradm user delete

Delete a user.

```bash
butleradm user delete <email> [flags]
```

---

## butleradm addon

Manage addon catalog.

### butleradm addon list

List available addon definitions.

```bash
butleradm addon list
```

**Output:**

```
NAME           CATEGORY       VERSIONS                    DEFAULT
cilium         networking     1.17.0, 1.16.5, 1.16.0     1.17.0
metallb        loadbalancer   0.14.9, 0.14.5             0.14.9
longhorn       storage        1.7.0, 1.6.2               1.7.0
cert-manager   security       1.16.0, 1.15.3             1.16.0
prometheus     monitoring     25.0.0, 24.0.0             25.0.0
```

### butleradm addon sync

Sync addon definitions from repository.

```bash
butleradm addon sync [flags]
```

---

## butleradm upgrade

Upgrade Butler components.

### butleradm upgrade check

Check for available upgrades.

```bash
butleradm upgrade check
```

### butleradm upgrade apply

Apply an upgrade.

```bash
butleradm upgrade apply [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--version` | Target version |
| `--dry-run` | Show what would be upgraded |
| `--skip-backup` | Skip pre-upgrade backup |

---

## butleradm support-bundle

Collect diagnostics for troubleshooting.

```bash
butleradm support-bundle [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--output` | Output file path |
| `--include-secrets` | Include secrets (redacted) |
| `--since` | Include logs since duration |

**Examples:**

```bash
# Collect support bundle
butleradm support-bundle --output butler-support.tar.gz

# Include last 24 hours of logs
butleradm support-bundle --output support.tar.gz --since 24h
```

---

## butleradm version

Print version information.

```bash
butleradm version
```

---

## Exit Codes

| Code | Description |
|------|-------------|
| `0` | Success |
| `1` | General error |
| `2` | Configuration error |
| `3` | Resource not found |
| `4` | Bootstrap failed |

---

## See Also

- [butlerctl](./butlerctl.md) - Tenant cluster management CLI
- [Getting Started](../../getting-started/) - Bootstrap your first cluster
- [Provider Guides](../../providers/) - Infrastructure provider setup
