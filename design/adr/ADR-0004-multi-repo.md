# ADR-0004: Multi-Repository Architecture

## Status

Accepted

## Context

Butler consists of multiple components: API types, controllers, providers, CLI, console, charts. We needed to decide how to organize the code:

1. **Monorepo**: All components in one repository
2. **Multi-repo**: Separate repositories per component
3. **Hybrid**: Core in monorepo, providers separate

Considerations:

- Independent release cycles
- Team ownership boundaries
- CI/CD complexity
- Dependency management
- Developer experience

## Decision

We adopt a **multi-repository architecture** with clear boundaries:

```
butlerdotdev/
├── butler                      # Umbrella (docs, governance)
├── butler-api                  # Shared CRD types
├── butler-controller           # Core controllers
├── butler-bootstrap            # Bootstrap controller
├── butler-cli                  # CLI tools
├── butler-console              # Web UI frontend
├── butler-server               # API backend
├── butler-charts               # Helm charts
├── butler-provider-harvester   # Harvester provider
├── butler-provider-nutanix     # Nutanix provider
└── homebrew-tap                # Package distribution
```

### Repository Boundaries

| Repository | Contents | Rationale |
|------------|----------|-----------|
| butler-api | CRD types | Shared dependency, stable interface |
| butler-controller | TenantCluster, Team controllers | Core platform logic |
| butler-bootstrap | Bootstrap controller | Different lifecycle, occasional use |
| butler-cli | butleradm, butlerctl | User-facing, frequent releases |
| butler-console | React frontend | Different tech stack |
| butler-server | Go API server | Backend for console |
| butler-charts | All Helm charts | Unified chart publishing |
| butler-provider-* | Provider implementations | Independent development |

## Consequences

### Positive

- **Independent releases**: Components version independently
- **Clear ownership**: Teams own specific repos
- **Focused CI/CD**: Faster builds per component
- **Tech flexibility**: Different stacks per repo
- **Granular access**: Repo-level permissions

### Negative

- **Coordination overhead**: Cross-repo changes need coordination
- **Dependency management**: Must track butler-api version
- **Developer friction**: Multiple repos to clone/update
- **Release coordination**: Need compatibility matrix

### Neutral

- Go modules for dependency management
- GitHub organization for grouping
- CODEOWNERS per repo

## Alternatives Considered

### Alternative 1: Monorepo

All components in one repository.

**Why rejected:**

- Slower CI (all tests run on every change)
- Mixed tech stacks complicate tooling
- Harder to grant focused access
- Release coupling

### Alternative 2: Hybrid

Core (api, controller, bootstrap) in monorepo, providers separate.

**Why rejected:**

- Unclear boundary
- Still has monorepo drawbacks for core
- Inconsistent contributor experience

## Implementation Notes

### Dependency Flow

```
butler-api (types)
    ↓
butler-controller ← butler-provider-*
butler-bootstrap
butler-cli
butler-server
    ↓
butler-charts (aggregates)
```

### Version Coordination

- butler-api versions as `v0.x.y`
- Consumers pin to compatible versions
- Compatibility matrix in umbrella repo

### Cross-Repo Changes

1. Open tracking issue in umbrella
2. PR to butler-api first
3. PRs to dependent repos
4. Coordinate merge order

## References

- [COMPONENTS.md](../../COMPONENTS.md) - Component registry
- [Compatibility Matrix](../../releases/compatibility-matrix.md) - Version tracking
