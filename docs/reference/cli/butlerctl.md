---
title: butlerctl
sidebar_position: 2
---

`butlerctl` is the CLI for developers and platform users to manage tenant clusters.

## Synopsis

```bash
butlerctl [command] [flags]
```

## Global Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--verbose` | `-v` | Enable verbose output |
| `--help` | `-h` | Help for command |

> **Note:** Kubeconfig resolution follows the standard Kubernetes convention: `--kubeconfig` flag on subcommands, `KUBECONFIG` environment variable, or `~/.kube/config` default. Namespace and output format flags are available on individual subcommands, not globally.

---

## butlerctl cluster

Manage tenant clusters.

### butlerctl cluster list

List all accessible tenant clusters.

```bash
butlerctl cluster list [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--all-namespaces` | `-A` | List clusters in all namespaces |
| `--namespace` | `-n` | List clusters in specific namespace |
| `--environment` | | Filter to clusters in the given [environment](../../concepts/environments.md). Shows an ENV column when any cluster in the result carries the env label |

**Examples:**

```bash
# List clusters in current namespace
butlerctl cluster list

# List all clusters (if you have access)
butlerctl cluster list -A

# List clusters in team namespace
butlerctl cluster list -n team-backend
```

**Output:**

```
NAME          NAMESPACE      PHASE   WORKERS   VERSION   AGE
dev-cluster   team-backend   Ready   3/3       v1.30.0   2d
staging       team-backend   Ready   5/5       v1.30.0   7d
```

### butlerctl cluster create

Create a new tenant cluster.

```bash
butlerctl cluster create <name> [flags]
```

**Flags:**

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--k8s-version` | | Kubernetes version | `v1.30.2` |
| `--workers` | `-w` | Number of worker nodes (1-10) | `1` |
| `--cpu` | | CPU cores per worker (1-128) | `4` |
| `--memory` | | Memory per worker | `8Gi` |
| `--disk` | | Disk size per worker | `50Gi` |
| `--provider` | `-p` | ProviderConfig name (auto-detected if only one exists) | |
| `--image` | | OS image reference (UUID or namespace/name) | |
| `--schematic-id` | | Image Factory schematic ID | |
| `--lb-pool` | | LoadBalancer IP pool (SINGLE_IP or START-END) | |
| `--lb-pool-start` | | LoadBalancer pool start IP | |
| `--lb-pool-end` | | LoadBalancer pool end IP | |
| `--pod-cidr` | | Pod network CIDR | `10.244.0.0/16` |
| `--service-cidr` | | Service network CIDR | `10.96.0.0/12` |
| `--filename` | `-f` | Create from YAML file (skips flag validation) | |
| `--dry-run` | | Preview TenantCluster YAML without creating | `false` |
| `--wait` | | Wait for cluster to reach Ready | `false` |
| `--timeout` | | Timeout for --wait | `15m` |
| `--environment` | | Place the cluster in the given [team environment](../../concepts/environments.md). Required when the team has any environments defined. Stamps `butler.butlerlabs.dev/environment: <env>` label and applies env `clusterDefaults` to unset fields. | |

Control plane resource flags (all optional, override ButlerConfig defaults):

| Flag | Description |
|------|-------------|
| `--cp-apiserver-cpu-request` | API server CPU request (e.g., 100m) |
| `--cp-apiserver-memory-request` | API server memory request (e.g., 256Mi) |
| `--cp-apiserver-cpu-limit` | API server CPU limit |
| `--cp-apiserver-memory-limit` | API server memory limit |
| `--cp-controller-manager-cpu-request` | Controller manager CPU request |
| `--cp-controller-manager-memory-request` | Controller manager memory request |
| `--cp-controller-manager-cpu-limit` | Controller manager CPU limit |
| `--cp-controller-manager-memory-limit` | Controller manager memory limit |
| `--cp-scheduler-cpu-request` | Scheduler CPU request |
| `--cp-scheduler-memory-request` | Scheduler memory request |
| `--cp-scheduler-cpu-limit` | Scheduler CPU limit |
| `--cp-scheduler-memory-limit` | Scheduler memory limit |

**Examples:**

```bash
# Create minimal cluster
butlerctl cluster create dev-cluster --lb-pool 10.40.1.100-10.40.1.110

# Create with custom resources
butlerctl cluster create prod-api \
  --k8s-version v1.31.0 \
  --workers 5 \
  --cpu 8 \
  --memory 32Gi \
  --disk 200Gi \
  --lb-pool-start 10.40.2.100 \
  --lb-pool-end 10.40.2.150

# Dry run to preview YAML
butlerctl cluster create dev --dry-run

# Create from YAML file
butlerctl cluster create -f cluster.yaml

# Create and wait for ready
butlerctl cluster create dev \
  --lb-pool 10.40.1.100-10.40.1.110 \
  --wait --timeout 20m
```

### butlerctl cluster get

Get details about a specific cluster.

```bash
butlerctl cluster get <name> [flags]
```

**Examples:**

```bash
# Get cluster details (table)
butlerctl cluster get dev-cluster

# Get as YAML
butlerctl cluster get dev-cluster -o yaml

# Get as JSON
butlerctl cluster get dev-cluster -o json
```

### butlerctl cluster scale

Scale worker nodes in a cluster.

```bash
butlerctl cluster scale <name> [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--namespace` | `-n` | Namespace of the cluster |
| `--workers` | | Desired number of workers (required) |
| `--wait` | | Wait for scaling to complete |
| `--timeout` | | Timeout for --wait |

**Examples:**

```bash
# Scale to 5 workers
butlerctl cluster scale dev-cluster --workers 5

# Scale in a specific namespace
butlerctl cluster scale dev-cluster -n team-backend --workers 5

# Scale and wait
butlerctl cluster scale dev-cluster --workers 10 --wait
```

### butlerctl cluster kubeconfig

Get kubeconfig for accessing a tenant cluster.

```bash
butlerctl cluster kubeconfig <name> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--output` | Write to file instead of stdout |
| `--merge` | Merge into existing kubeconfig |
| `--set-current` | Set as current context (with --merge) |

**Examples:**

```bash
# Print kubeconfig to stdout
butlerctl cluster kubeconfig dev-cluster

# Save to file
butlerctl cluster kubeconfig dev-cluster --output ~/.kube/dev-cluster.yaml

# Merge into default kubeconfig
butlerctl cluster kubeconfig dev-cluster --merge

# Use directly with kubectl
butlerctl cluster kubeconfig dev-cluster | kubectl --kubeconfig /dev/stdin get nodes
```

### butlerctl cluster export

Export cluster configuration as clean, git-ready YAML.

Unlike `kubectl get -o yaml`, this produces clean output suitable for:
- Checking into Git (GitOps workflows)
- Using as a template for new clusters
- Sharing for support/debugging
- Disaster recovery backups

The output strips all the noise: `resourceVersion`, `uid`, `creationTimestamp`, `managedFields`, and status.

```bash
butlerctl cluster export <name> [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--output` | `-o` | Write to file or directory instead of stdout |
| `--as` | | Rename the cluster in the exported YAML |
| `--all` | | Export all clusters in namespace |
| `--all-namespaces` | `-A` | Export from all namespaces (with --all) |
| `--include-status` | | Include status in output (excluded by default) |

**Examples:**

```bash
# Export to stdout
butlerctl cluster export my-cluster

# Export to file
butlerctl cluster export my-cluster -o my-cluster.yaml

# Export as a template with new name
butlerctl cluster export my-cluster --as team-beta-cluster

# Export all clusters to a directory
butlerctl cluster export --all -o clusters/

# Export all clusters from all namespaces
butlerctl cluster export --all -A -o clusters/

# Include status for debugging
butlerctl cluster export my-cluster --include-status
```

### butlerctl cluster destroy

Destroy a tenant cluster.

```bash
butlerctl cluster destroy <name> [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--force` | Skip confirmation prompt |
| `--wait` | Wait for deletion to complete |
| `--timeout` | Timeout for --wait |

**Examples:**

```bash
# Destroy with confirmation
butlerctl cluster destroy dev-cluster

# Force destroy (no confirmation)
butlerctl cluster destroy dev-cluster --force

# Destroy and wait
butlerctl cluster destroy dev-cluster --force --wait
```

---

## butlerctl image

Manage OS images and image sync operations.

Butler uses ImageSync resources to sync OS images (Talos, Flatcar, Rocky, etc.) from the Butler Image Factory to infrastructure providers. These commands manage that lifecycle.

### butlerctl image list

List ImageSync resources.

```bash
butlerctl image list [flags]
```

**Aliases:** `ls`

**Flags:**

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--namespace` | `-n` | Namespace | `butler-system` |
| `--output` | `-o` | Output format: table, json, yaml | `table` |
| `--provider` | | Filter by provider name | |
| `--status` | | Filter by phase (Pending, Building, Downloading, Uploading, Ready, Failed) | |
| `--schematic` | | Filter by schematic ID prefix | |

**Output:**

```
NAME                         PROVIDER         PHASE   SCHEMATIC   VERSION    IMAGE REF              AGE
image-71e06ba7-v1122-hvstr   harvester-prod   Ready   71e06ba7    v1.12.2    default/talos-v1122    3d
image-f4f34401-v4459-hvstr   harvester-prod   Ready   f4f34401    4459.2.1   default/flatcar-4459   1d
```

### butlerctl image sync

Create an ImageSync to sync an image to a provider.

```bash
butlerctl image sync [flags]
```

**Flags:**

| Flag | Description | Required |
|------|-------------|----------|
| `--schematic-id` | Schematic ID from the Image Factory | Yes |
| `--version` | OS version (e.g., v1.12.4) | Yes |
| `--provider` | Provider config reference (namespace/name or name) | Yes |
| `--arch` | CPU architecture (default: amd64) | No |
| `--format` | Image format: qcow2, raw, iso (default: qcow2) | No |
| `--transfer-mode` | Transfer mode: direct, proxy (default: direct) | No |
| `--display-name` | Human-readable name for the image on the provider | No |
| `--name` | Name for the ImageSync resource (auto-generated if not set) | No |
| `--namespace` `-n` | Namespace (default: butler-system) | No |

**Examples:**

```bash
# Sync Talos image to Harvester
butlerctl image sync \
  --schematic-id 71e06ba76d3cf365bb4ab4d8f8f4fea55a7620811666b9c25623734ab18ddd27 \
  --version v1.12.2 \
  --provider harvester-prod

# Sync with custom name and format
butlerctl image sync \
  --schematic-id f4f3440b79585e90ceeb866544513671b4be94fcf4326fc97bdd136b01a0a1b5 \
  --version 4459.2.1 \
  --provider harvester-prod \
  --format raw \
  --display-name "Flatcar 4459.2.1"
```

### butlerctl image status

Get detailed status of an ImageSync.

```bash
butlerctl image status <name> [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--namespace` | `-n` | Namespace (default: butler-system) |
| `--output` | `-o` | Output format: yaml, json |

### butlerctl image delete

Delete an ImageSync resource. This stops the sync process but does not remove the image from the provider.

```bash
butlerctl image delete <name> [flags]
```

**Flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--namespace` | `-n` | Namespace (default: butler-system) |
| `--yes` | `-y` | Skip confirmation prompt |

### butlerctl image catalog

Show available images from existing ImageSync resources.

```bash
butlerctl image catalog [flags]
```

**Output:**

```
SCHEMATIC   VERSIONS          FORMATS     READY   TOTAL
71e06ba7    v1.12.2, v1.12.1  qcow2       2       2
f4f34401    4459.2.1          raw         1       1
```

---

## butlerctl version

Print version information.

```bash
butlerctl version
```

**Output:**

```
butlerctl version v0.1.0
  Built: 2026-01-15T10:00:00Z
  Git commit: abc1234
  Go version: go1.24.6
```

---

## Exit Codes

| Code | Description |
|------|-------------|
| `0` | Success |
| `1` | General error |
| `2` | Configuration error |
| `3` | Resource not found |
| `4` | Permission denied |

---

## See Also

- [butleradm](./butleradm.md) - Platform admin CLI
- [Getting Started](../../getting-started/) - Create your first cluster
