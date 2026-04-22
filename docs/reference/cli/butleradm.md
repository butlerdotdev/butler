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
| `--config` | `-c` | Path to config file |
| `--verbose` | `-v` | Enable verbose output |
| `--help` | `-h` | Help for command |

---

## butleradm bootstrap

Bootstrap a new Butler management cluster.

### Common Flags

All `butleradm bootstrap <provider>` commands share these flags. Each provider also has credential override flags documented in its section below.

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

**Provider Flags:**

| Flag | Description |
|------|-------------|
| `--harvester-kubeconfig` | Path to Harvester kubeconfig (overrides config file) |

**Examples:**

```bash
# Bootstrap with config file
butleradm bootstrap harvester --config bootstrap.yaml

# Override Harvester kubeconfig via flag
butleradm bootstrap harvester --config bootstrap.yaml \
  --harvester-kubeconfig /path/to/harvester-kubeconfig

# Dry run to validate
butleradm bootstrap harvester --config bootstrap.yaml --dry-run
```

**Bootstrap Config Example:**

The bootstrap config is a plain YAML file (not a Kubernetes resource). See [Bootstrap Config Reference](../bootstrap-config.md) for all fields.

```yaml
provider: harvester

cluster:
  name: butler-mgmt
  topology: ha
  controlPlane:
    replicas: 3
    cpu: 4
    memoryMB: 8192
    diskGB: 50
  workers:
    replicas: 3
    cpu: 8
    memoryMB: 16384
    diskGB: 100

network:
  podCIDR: 10.244.0.0/16
  serviceCIDR: 10.96.0.0/12
  vip: 10.40.0.100
  loadBalancerPool:
    start: 10.40.0.110
    end: 10.40.0.150

providerConfig:
  harvester:
    kubeconfigPath: ~/.butler/harvester-kubeconfig
    namespace: default
    networkName: default/vlan40-workloads
    imageName: default/talos-v1-12-1
```

### butleradm bootstrap nutanix

Bootstrap Butler on Nutanix AHV.

```bash
butleradm bootstrap nutanix [flags]
```

**Provider Flags:**

| Flag | Description |
|------|-------------|
| `--prism-endpoint` | Nutanix Prism Central endpoint (overrides config file) |
| `--prism-username` | Nutanix Prism Central username (overrides config file) |
| `--prism-password` | Nutanix Prism Central password (overrides config file) |

**Examples:**

```bash
# Bootstrap with config file
butleradm bootstrap nutanix --config bootstrap.yaml

# Override Prism credentials via flags
butleradm bootstrap nutanix --config bootstrap.yaml \
  --prism-endpoint https://prism.example.com:9440 \
  --prism-username admin \
  --prism-password "${PRISM_PASSWORD}"
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

**Provider Flags:**

| Flag | Description |
|------|-------------|
| `--credentials` | Path to GCP service account JSON key (overrides config file) |

**Examples:**

```bash
# Bootstrap on GCP
butleradm bootstrap gcp --config bootstrap.yaml

# Override service account key via flag
butleradm bootstrap gcp --config bootstrap.yaml \
  --credentials /path/to/sa-key.json

# Dry run to validate
butleradm bootstrap gcp --config bootstrap.yaml --dry-run
```

### butleradm bootstrap aws

Bootstrap Butler on Amazon Web Services.

```bash
butleradm bootstrap aws [flags]
```

**Provider Flags:**

| Flag | Description |
|------|-------------|
| `--access-key-id` | AWS access key ID (overrides config file) |
| `--secret-access-key` | AWS secret access key (overrides config file) |

**Examples:**

```bash
# Bootstrap on AWS
butleradm bootstrap aws --config bootstrap.yaml

# Override AWS credentials via flags
butleradm bootstrap aws --config bootstrap.yaml \
  --access-key-id "${AWS_ACCESS_KEY_ID}" \
  --secret-access-key "${AWS_SECRET_ACCESS_KEY}"
```

### butleradm bootstrap azure

Bootstrap Butler on Microsoft Azure.

```bash
butleradm bootstrap azure [flags]
```

**Provider Flags:**

| Flag | Description |
|------|-------------|
| `--client-id` | Azure service principal app ID (overrides config file) |
| `--client-secret` | Azure service principal password (overrides config file) |
| `--tenant-id` | Azure tenant ID (overrides config file) |
| `--subscription-id` | Azure subscription ID (overrides config file) |

**Examples:**

```bash
# Bootstrap on Azure
butleradm bootstrap azure --config bootstrap.yaml

# Override Azure credentials via flags
butleradm bootstrap azure --config bootstrap.yaml \
  --client-id "${AZURE_CLIENT_ID}" \
  --client-secret "${AZURE_CLIENT_SECRET}" \
  --tenant-id "${AZURE_TENANT_ID}" \
  --subscription-id "${AZURE_SUBSCRIPTION_ID}"
```

### Bootstrap Config Reference

The bootstrap config is a plain YAML file parsed by the CLI. See the [Bootstrap Config Reference](../bootstrap-config.md) for all fields. Key sections:

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
| `--type` | Provider type: `harvester`, `nutanix`, `proxmox`, `aws`, `azure`, `gcp` |
| `--kubeconfig` | Provider kubeconfig (Harvester) |
| `--endpoint` | API endpoint (Nutanix/Proxmox) |
| `--credentials` | Path to credentials file |
| `--namespace` | Namespace for ProviderConfig |

---

## butleradm env

Manage team [environments](../../concepts/environments.md). An environment is a named slot within a Team that optionally caps cluster count, per-member cluster count, and default cluster shape.

All env subcommands require `--team <name>`.

### butleradm env list

List environments on a team with live cluster counts.

```bash
butleradm env list --team <team> [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--team` | | Team name (required) |
| `--output` | `-o` | Output format: `table` (default), `json`, `yaml` |

### butleradm env create

Append an environment to a team.

```bash
butleradm env create <name> --team <team> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--team` | Team name (required) |
| `--description` | Free-text description shown in the env list and console |
| `--max-clusters` | Maximum TenantClusters in this env (0 = no cap) |
| `--max-clusters-per-member` | Per-user cap inside this env (0 = no cap) |
| `--cluster-default` | Repeatable `key=value` default. Keys: `kubernetesVersion`, `workerCount`, `workerCPU`, `workerMemoryGi`, `workerDiskGi` |
| `--access-user` | Repeatable `email:role`. Role must be `admin`, `operator`, or `viewer` |
| `--access-group` | Repeatable `name:role` or `name:role:identityProvider` |

**Examples:**

```bash
# Minimal: name only
butleradm env create staging --team payments

# With caps
butleradm env create sandbox --team payments --max-clusters-per-member 1
butleradm env create prod --team payments --max-clusters 10

# Full shape
butleradm env create prod --team payments \
  --description "production workloads" \
  --cluster-default kubernetesVersion=v1.31.0 \
  --cluster-default workerCount=3 \
  --access-user alice@example.com:admin \
  --access-group sre:admin
```

### butleradm env update

Patch an environment in place. Unset flags leave fields unchanged; the env name is immutable.

```bash
butleradm env update <name> --team <team> [flags]
```

**Flags:** same as `env create`, plus:

| Flag | Description |
|------|-------------|
| `--clear-limits` | Remove the entire limits block |
| `--clear-cluster-defaults` | Remove the entire clusterDefaults block |
| `--clear-access` | Remove the entire access block (users and groups) |

Partial clears use the same flags as set, with an empty value:

```bash
# Clear one cluster-default key, keep others
butleradm env update prod --team payments --cluster-default workerCount=

# Remove one access-user entry, keep others
butleradm env update prod --team payments --access-user alice@example.com:

# Remove one access-group entry (by name + idp match)
butleradm env update prod --team payments --access-group developers::okta
```

### butleradm env delete

Remove an environment from a team.

```bash
butleradm env delete <name> --team <team> [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--team` | | Team name (required) |
| `--force` / `--yes` | `-y` | Skip confirmation prompt |

Clusters that carried the deleted env's label remain but are no longer accounted under any env's cap. They continue to count against the team's total `maxClusters`.

### butleradm env migrate

Bulk-label existing TenantClusters into a target env. Use this after adding envs to a team that already has clusters; the pre-existing clusters stay unlabeled until migrated.

```bash
butleradm env migrate --team <team> --environment <env> [NAMES...] [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--team` | Team name (required) |
| `--environment` | Target env name (required) |
| `--all` | Migrate every unlabeled cluster in the team namespace |
| `--relabel` | Also rewrite clusters already carrying a different env label (requires extra confirmation) |
| `--force` / `-y` | Skip the confirmation prompt |

**Examples:**

```bash
# Migrate every unlabeled cluster into prod
butleradm env migrate --team payments --environment prod --all

# Migrate specific clusters
butleradm env migrate --team payments --environment prod web-1 web-2

# Relabel clusters already in dev into staging
butleradm env migrate --team payments --environment staging --relabel --all
```

The migration writes both the env label and the `butler.butlerlabs.dev/migration-operation: "true"` annotation the admission webhook requires for env-label mutations.

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

- [butlerctl](./butlerctl.md) -- Tenant cluster management CLI
- [Getting Started](../../getting-started/) -- Bootstrap guides for each provider
