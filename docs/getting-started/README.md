---
title: Getting Started
sidebar_position: 1
---

# Getting Started

Set up a Butler management cluster on your infrastructure, then create your first tenant cluster.

## Prerequisites

| Requirement | Description |
|-------------|-------------|
| Docker | Runs the temporary KIND bootstrap cluster |
| kubectl | Kubernetes CLI (1.28+) |
| Infrastructure | One of: Harvester, Nutanix, AWS, GCP, or Azure |

## Install the CLI

Butler ships two binaries: `butleradm` (bootstrap and platform administration) and `butlerctl` (tenant operations).

### Homebrew (macOS / Linux)

```bash
brew install butlerdotdev/tap/butler
```

### Direct Download (macOS / Linux)

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
[[ "$ARCH" == "x86_64" ]] && ARCH="amd64"
[[ "$ARCH" == "aarch64" ]] && ARCH="arm64"

curl -sLO "https://github.com/butlerdotdev/butler-cli/releases/latest/download/butler_${OS}_${ARCH}.tar.gz"
tar xzf butler_${OS}_${ARCH}.tar.gz
sudo mv butleradm butlerctl /usr/local/bin/
```

### Verify

```bash
butleradm version
butlerctl version
```

## Bootstrap a Management Cluster

Each provider guide walks through infrastructure prerequisites, config file creation, running bootstrap, and validation. Pick your provider:

| Provider | Type | Guide |
|----------|------|-------|
| Harvester | On-prem | [Harvester](./harvester.md) |
| Nutanix | On-prem | [Nutanix](./nutanix.md) |
| AWS | Cloud | [AWS](./aws.md) |
| GCP | Cloud | [GCP](./gcp.md) |
| Azure | Cloud | [Azure](./azure.md) |

Bootstrap takes 15-30 minutes and produces a Kubernetes cluster running Talos Linux with Steward, Butler, and platform addons installed.

## After Bootstrap

Once your management cluster passes the validation checklist in the provider guide:

1. **[Create your first tenant cluster](./first-tenant-cluster.md)** -- Define a Team, create a TenantCluster, and get a kubeconfig.
2. **[Explore the Console](./console.md)** -- Open the web UI to view clusters, manage addons, and access terminals.

## Troubleshooting

See [Troubleshooting](../troubleshooting/) for common issues with bootstrap, cluster provisioning, networking, and addons.
