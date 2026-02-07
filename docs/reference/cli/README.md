---
title: CLI Reference
sidebar_position: 1
---

# Command-Line Interface

Butler provides two CLI tools for different personas:

## Overview

| CLI | Target User | Purpose |
|-----|-------------|---------|
| [butlerctl](./butlerctl.md) | Developers, Platform Users | Manage tenant clusters |
| [butleradm](./butleradm.md) | Platform Operators | Manage Butler platform |

This separation follows the `kubectl`/`kubeadm` pattern, providing focused tools for each use case.

## Installation

### Homebrew (macOS/Linux)

```bash
brew install butlerdotdev/tap/butler
```

### Direct Download

```bash
# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
[[ "$ARCH" == "x86_64" ]] && ARCH="amd64"
[[ "$ARCH" == "aarch64" ]] && ARCH="arm64"

# Download latest release
curl -sLO "https://github.com/butlerdotdev/butler-cli/releases/latest/download/butler_${OS}_${ARCH}.tar.gz"
tar xzf butler_${OS}_${ARCH}.tar.gz

# Install
sudo mv butleradm butlerctl /usr/local/bin/
```

### Verify Installation

```bash
butlerctl version
butleradm version
```

## Quick Reference

### butlerctl Commands

```bash
# Cluster operations
butlerctl cluster list                          # List clusters
butlerctl cluster create my-cluster             # Create cluster
butlerctl cluster get my-cluster                # Get cluster details
butlerctl cluster scale my-cluster --workers 5  # Scale workers
butlerctl cluster kubeconfig my-cluster         # Get kubeconfig
butlerctl cluster export my-cluster             # Export for GitOps
butlerctl cluster destroy my-cluster            # Destroy cluster
```

### butleradm Commands

```bash
# Platform operations
butleradm bootstrap harvester --config cfg.yaml # Bootstrap platform
butleradm status                                 # Check health
butleradm provider list                          # List providers
butleradm provider validate harvester-prod       # Test connectivity
```

## Configuration

Both CLIs use the standard kubeconfig for connecting to the management cluster:

```bash
# Use default kubeconfig
butlerctl cluster list

# Use specific kubeconfig
butlerctl cluster list --kubeconfig ~/.butler/mgmt-kubeconfig

# Use KUBECONFIG environment variable
export KUBECONFIG=~/.butler/mgmt-kubeconfig
butlerctl cluster list
```

## Output Formats

Both CLIs support multiple output formats:

```bash
# Table (default)
butlerctl cluster list

# YAML
butlerctl cluster list -o yaml

# JSON
butlerctl cluster list -o json
```

## Shell Completion

### Bash

```bash
butlerctl completion bash > /etc/bash_completion.d/butlerctl
butleradm completion bash > /etc/bash_completion.d/butleradm
```

### Zsh

```bash
butlerctl completion zsh > "${fpath[1]}/_butlerctl"
butleradm completion zsh > "${fpath[1]}/_butleradm"
```

### Fish

```bash
butlerctl completion fish > ~/.config/fish/completions/butlerctl.fish
butleradm completion fish > ~/.config/fish/completions/butleradm.fish
```

## See Also

- [butlerctl Reference](./butlerctl.md) - Full command reference
- [butleradm Reference](./butleradm.md) - Full command reference
- [Getting Started](../../getting-started/) - Hands-on guide
