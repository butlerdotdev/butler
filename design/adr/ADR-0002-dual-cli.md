# ADR-0002: Dual CLI Pattern

## Status

Accepted

## Context

Butler needs command-line tools for two distinct user groups with different needs:

1. **Platform Operators**: Manage the Butler platform itself (bootstrap, upgrade, backup)
2. **Platform Users**: Consume the platform to run workloads (create clusters, manage addons)

These groups have different:
- Permission requirements (cluster-admin vs. namespace-scoped)
- Usage frequency (occasional vs. daily)
- Mental models (infrastructure vs. application)

We needed to decide how to structure the CLI tooling:

1. **Single CLI with subcommands**: `butler adm bootstrap`, `butler ctl cluster create`
2. **Single CLI with role flags**: `butler --admin bootstrap`, `butler cluster create`
3. **Two separate CLIs**: `butleradm`, `butlerctl`
4. **kubectl plugins**: `kubectl butler-adm`, `kubectl butler-ctl`

## Decision

We implement **two separate CLI binaries**: `butleradm` and `butlerctl`.

### butleradm

Target: Platform Operators (infrastructure teams, SREs)

Commands:
- `butleradm bootstrap` - Create management cluster
- `butleradm upgrade` - Upgrade Butler components
- `butleradm backup` - Backup cluster state
- `butleradm restore` - Restore from backup
- `butleradm status` - Platform health check

### butlerctl

Target: Platform Users (developers, application teams)

Commands:
- `butlerctl cluster create/list/get/delete/scale` - Manage clusters
- `butlerctl cluster kubeconfig` - Get cluster credentials
- `butlerctl addon install/list/uninstall` - Manage addons
- `butlerctl team list/switch` - Team management

Both binaries share common code (logging, Kubernetes client) via `internal/common/`.

## Consequences

### Positive

- **Clear mental model**: Users know which tool for which task
- **Appropriate permissions**: Platform users don't need admin tooling
- **Smaller binaries**: Only install what you need
- **Familiar pattern**: Mirrors kubeadm/kubectl from Kubernetes
- **RBAC simplicity**: Can grant access per-tool

### Negative

- **Two binaries to maintain**: Separate build/test/release
- **Some code duplication**: Command structure boilerplate
- **Installation complexity**: Users may need both
- **Version coordination**: Tools must be compatible

### Neutral

- Same underlying CRD-based architecture
- Can be built from same repository
- Installation can bundle both tools

## Alternatives Considered

### Alternative 1: Single CLI with Subcommands

`butler adm bootstrap`, `butler ctl cluster create`

**Why rejected:** 
- Larger binary for users who only need one set of commands
- Confusing: which subcommand to use?
- Harder to configure RBAC based on command usage

### Alternative 2: kubectl Plugins

`kubectl butler-adm bootstrap`, `kubectl butler-ctl cluster create`

**Why rejected:**
- Requires kubectl installation
- Plugin discovery complexity
- Bootstrap runs before any cluster exists

### Alternative 3: Single Binary with Mode Flag

`butler --admin bootstrap`, `butler cluster create`

**Why rejected:**
- Easy to make mistakes (`--admin` on wrong command)
- Doesn't solve the RBAC/permission problem
- Less clear than separate tools

## References

- [butler-cli repository](https://github.com/butlerdotdev/butler-cli)
- [kubeadm reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
