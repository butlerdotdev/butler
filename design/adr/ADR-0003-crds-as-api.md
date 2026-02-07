# ADR-0003: CRDs as the API Contract

## Status

Accepted

## Context

Butler needs an API for users to interact with the platform. The options were:

1. **Custom REST API**: Build a custom API server with its own endpoints
2. **GraphQL API**: Flexible query language for complex operations
3. **CRDs + Kubernetes API**: Use Kubernetes Custom Resources as the API

Key requirements:
- GitOps compatibility (declarative, stored in Git)
- Multiple clients (CLI, Console, kubectl)
- Auditability (who changed what, when)
- Standard tooling (existing K8s ecosystem)
- Resumability (operations can be interrupted and resumed)

## Decision

All Butler operations are expressed as **Kubernetes Custom Resource Definitions (CRDs)**. The CLIs and Console are thin clients that:

1. Parse user input
2. Construct CR objects
3. Apply CRs to the Kubernetes API
4. Watch CR status for completion

### Core CRDs

| CRD | Purpose |
|-----|---------|
| ClusterBootstrap | Management cluster lifecycle |
| TenantCluster | Tenant cluster lifecycle |
| TenantAddon | Addon installation on tenant clusters |
| ManagementAddon | Addon installation on management cluster |
| Team | Multi-tenant isolation |
| User | User accounts |
| ProviderConfig | Infrastructure credentials |
| ButlerConfig | Platform configuration |

### API Flow

```
User -> CLI/Console -> Kubernetes API -> CRD -> Controller -> Infrastructure
```

A `TenantCluster` created via `butlerctl cluster create` is identical to one created via Console or `kubectl apply`.

## Consequences

### Positive

- **Consistency**: Same resource regardless of client
- **GitOps native**: CRs can be stored in Git, applied via Flux/ArgoCD
- **Auditability**: Kubernetes audit logs capture all changes
- **Resumability**: Controllers reconcile toward desired state
- **Extensibility**: New features = new CRDs, no API changes
- **Standard tooling**: kubectl, k9s, Lens all work
- **Ecosystem integration**: Prometheus, Grafana, etc.

### Negative

- **Kubernetes dependency**: Requires K8s cluster for anything
- **Learning curve**: Users must understand CRD concepts
- **Eventual consistency**: Status updates are async
- **Complex status**: Nested status fields can be hard to query

### Neutral

- Validation via OpenAPI schema
- Admission webhooks for complex validation
- Status subresource for separation

## Alternatives Considered

### Alternative 1: Custom REST API

Build a custom API server like Rancher.

**Why rejected:**
- Another thing to maintain
- Not GitOps-friendly (imperative)
- Requires custom database
- Loses K8s ecosystem benefits

### Alternative 2: GraphQL API

Flexible query language.

**Why rejected:**
- Overkill for our use case
- Not declarative
- Not GitOps-compatible
- Additional complexity

## Implementation Notes

### CRD Versioning

All CRDs start at `v1alpha1` and follow Kubernetes API versioning:
- `v1alpha1`: Experimental, breaking changes expected
- `v1beta1`: Feature-complete, may have breaking changes
- `v1`: Stable, no breaking changes

### Shared Types

CRD types are defined in `butler-api` repository and imported by all components:

```go
import "github.com/butlerdotdev/butler-api/api/v1alpha1"
```

## References

- [butler-api repository](https://github.com/butlerdotdev/butler-api)
- [Kubernetes CRD documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
