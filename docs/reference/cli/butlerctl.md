---
title: butlerctl
sidebar_position: 2
---

# butlerctl

`butlerctl` is the CLI for developers and platform users to manage tenant clusters.

## Synopsis

```bash
butlerctl [command] [flags]
```

## Global Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--kubeconfig` | | Path to kubeconfig file (default: `~/.kube/config`) |
| `--context` | | Kubernetes context to use |
| `--namespace` | `-n` | Namespace for resources |
| `--output` | `-o` | Output format: `table`, `yaml`, `json` |
| `--help` | `-h` | Help for command |

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
NAME          NAMESPACE      PHASE     WORKERS   VERSION   AGE
dev-cluster   team-backend   Running   3/3       v1.30.0   2d
staging       team-backend   Running   5/5       v1.30.0   7d
```

### butlerctl cluster create

Create a new tenant cluster.

```bash
butlerctl cluster create <name> [flags]
```

**Flags:**

| Flag | Description | Default |
|------|-------------|---------|
| `--k8s-version` | Kubernetes version (e.g., v1.30.0) | `v1.30.0` |
| `--workers` | Number of worker nodes | `3` |
| `--cpu` | CPU cores per worker | `4` |
| `--memory` | Memory per worker | `8Gi` |
| `--disk` | Disk size per worker | `40Gi` |
| `--provider` | ProviderConfig name | (default provider) |
| `--wait` | Wait for cluster to be ready | `false` |
| `--timeout` | Timeout for --wait | `30m` |

**Examples:**

```bash
# Create minimal cluster
butlerctl cluster create dev-cluster

# Create with custom resources
butlerctl cluster create prod-api \
  --k8s-version v1.30.0 \
  --workers 5 \
  --cpu 8 \
  --memory 32Gi \
  --disk 200Gi

# Create and wait for ready
butlerctl cluster create dev --wait --timeout 20m
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

| Flag | Description |
|------|-------------|
| `--workers` | Desired number of workers (required) |
| `--wait` | Wait for scaling to complete |
| `--timeout` | Timeout for --wait |

**Examples:**

```bash
# Scale to 5 workers
butlerctl cluster scale dev-cluster --workers 5

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

## butlerctl team

Manage team membership (self-service).

### butlerctl team list

List teams you are a member of.

```bash
butlerctl team list
```

### butlerctl team get

Get details about a team.

```bash
butlerctl team get <name>
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
  Go version: go1.23.0
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
