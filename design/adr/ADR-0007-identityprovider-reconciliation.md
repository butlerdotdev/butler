# ADR-0007: IdentityProvider CRD Reconciliation

## Status

Proposed (2026-04-20)

## Context

The `IdentityProvider` CRD is currently half-shipped:

- Schema exists in `butler-api` (`api/v1alpha1/identityprovider_types.go`), with fields for `type`, `displayName`, and a nested `oidc` block carrying issuer URL, client ID, client secret reference, redirect URL, scopes, and claim mappings.
- Cluster-scoped instances can be created via `kubectl apply` or the console admin UI at `/admin/identity-providers`. `butler-server`'s `IdentityProvidersHandler` does full CRUD against the CRD.
- The Teams handler at `butler-server/internal/api/handlers/teams.go:1018` uses the CRD only for a `metadata.name` existence check when adding a group-sync rule. It does not read spec fields.
- The active SSO login flow in `butler-server/internal/api/router.go:83-107` builds a single `oidcProvider` from `BUTLER_OIDC_*` environment variables exactly once at startup and never reloads it.

The result is a CRD whose spec fields are never consumed by any active code path. Operators who create an `IdentityProvider` through the console or `kubectl` see the resource land and get no SSO.

This contradicts [ADR-0003: CRDs as API](./ADR-0003-crds-as-api.md), which commits the platform to CRDs as the declarative interface. SSO configuration is the one place the platform relies on environment variables instead of CRDs.

Issue [butlerdotdev/butler#21](https://github.com/butlerdotdev/butler/issues/21) tracks the open decision.

## Decision

Finish the reconciliation. Add a controller that watches `IdentityProvider` CRDs and rebuilds the running OIDC provider on change. Deprecate the `BUTLER_OIDC_*` environment variables once the controller is the source of truth, but keep them as a bootstrap-phase input so the first IdP can land before a controller is available.

## Consequences

### Positive

- Aligns SSO configuration with the CRDs-as-API commitment in ADR-0003.
- Enables multi-IdP federation. Env vars can describe one provider; CRDs can describe many, each scoped or prioritized.
- Console admin UI starts writing resources that actually affect auth, matching what operators already expect when they click the form.
- GitOps workflows become uniform across platform state: no separate Helm values override path for IdP config.
- Post-bootstrap operator guide collapses from "set env vars, optionally create a CRD that has no effect" to "create a CRD."

### Negative

- Real engineering work. A watcher, a provider-build function that can run more than once, OIDC discovery re-run on change, and probably session revocation or grace periods for users whose IdP was changed out from under them.
- Changes the auth hot path from a startup-only init to a controller-driven rebuild. The error mode for a malformed CRD moves from "server fails to start" to "server runs with stale provider while the controller retries." Both are surfacable but the latter is subtler.
- Breaks the pattern where auth config is "owned" by the deployment layer (Helm values, env vars). Operators managing auth through GitOps now have to split their declarative state between two reconcilers (Flux + butler-server).
- Migration cost. Existing deployments have `BUTLER_OIDC_*` configured; they need a clear transition path.

### Neutral

- Default provider selection when multiple `IdentityProvider` resources exist. Pick by ordering, by a `default: true` field, or by first-created wins. Needs to be defined in the controller spec.
- Scope of the CRD. Currently cluster-scoped. Multi-tenant federation may want per-team or per-namespace IdPs; that widens scope but is a future extension, not a blocker for this ADR.

## Alternatives Considered

### Alternative 1: Delete the CRD

Remove `IdentityProvider` from `butler-api`. Remove the console admin UI for it. Replace Team group-sync validation with a looser check (or drop the validation). Consolidate SSO configuration on `BUTLER_OIDC_*` env vars, possibly with a `butler-server` admin endpoint that lets the console edit those env vars at runtime.

**Why rejected:** Inverts ADR-0003. Every other platform-state resource (Teams, Users, ProviderConfig, TenantCluster) is a CRD. Removing the CRD for SSO specifically creates a one-off architectural carveout that will resurface every time someone asks "why is this one thing different." Also loses the multi-IdP path entirely; env vars do not naturally describe a list.

### Alternative 2: Keep Both Paths Permanently

Leave env vars as the runtime source of truth and treat the CRD purely as documentation and as a stable anchor for Team group-sync `identityProvider` references. Freeze the current state.

**Why rejected:** The current state is the problem. Operators see the CRD and the admin UI and reasonably assume applying them does something. Keeping the shape without a reconciler means the platform continues to teach operators an incorrect mental model. If the decision is "env vars win," delete the CRD (Alternative 1). Having both without reconciliation is the worst shape.

## Implementation Notes

Rough phases. Each phase produces something shippable on its own.

### Phase 1: Controller and build function

- Extract `NewOIDCProvider` construction into a function that can be called more than once. Today it is inline in `router.go:86`. Move to a builder that takes the spec and returns a `*auth.OIDCProvider` or an error.
- Add an `IdentityProviderReconciler` in `butler-server` (or a new small controller binary if sharing state with the server becomes awkward). On Add/Update/Delete of `IdentityProvider` CRDs, rebuild the active provider. Store the current provider behind an atomic pointer so in-flight requests see a consistent view.
- Honor `spec.type: "oidc"` and ignore non-OIDC types until their controllers ship (future SAML, LDAP).

### Phase 2: Env var deprecation

- Add a log warning at startup when both `BUTLER_OIDC_ISSUER_URL` is set and an `IdentityProvider` CRD exists. The CRD wins; the env var is deprecated.
- Update bootstrap to create the `IdentityProvider` CRD alongside the env vars, so fresh clusters ship with both. A future release removes the env var path.

### Phase 3: Migration path

- `butleradm` command to migrate existing env-var config into a CRD. Reads the running Deployment's env vars, writes out an `IdentityProvider` YAML, lets operators review + apply.
- Once the CRD exists, the env vars can be stripped from the Deployment. The controller keeps serving the IdP from the CRD.

### Phase 4: Documentation

- Update `docs/getting-started/post-bootstrap.md` Step 3 to lead with the CRD instead of env vars. The existing "Roadmap" callout becomes the changelog entry.
- Update ADR-0003 if this surfaces any new constraints on the CRDs-as-API position.

### Open engineering questions

- Session revocation on provider change. If an IdP's `issuerURL` or `clientID` changes, do existing sessions stay valid? Probably yes, since the JWT was already minted, but the OIDC token refresh path needs review.
- Hot-reload correctness under concurrent requests. The atomic pointer handles the provider swap; what about in-flight OAuth callbacks carrying a state tied to the old provider's AuthCodeURL? State store TTL is probably already short enough.
- Google Workspace `GoogleServiceAccountJSON` / `GoogleAdminEmail` are server-global today. Lift into the IdentityProvider CRD's `spec.oidc.google` block, or keep as env-var-level config shared across all CRDs? Lifting into the CRD is cleaner.

## References

- Issue: [butlerdotdev/butler#21](https://github.com/butlerdotdev/butler/issues/21)
- [ADR-0003: CRDs as API](./ADR-0003-crds-as-api.md)
- Post-bootstrap doc Step 3: `docs/getting-started/post-bootstrap.md`
- Code today:
  - `butler-api/api/v1alpha1/identityprovider_types.go` (schema)
  - `butler-server/internal/api/router.go:83-107` (singleton provider build)
  - `butler-server/internal/api/handlers/identity_providers.go` (CRUD handler, no reconciliation)
  - `butler-server/internal/api/handlers/teams.go:1018` (existence-check-only validation)
- Related half-shipped pattern: [butler-server#39](https://github.com/butlerdotdev/butler-server/issues/39) invite URL still captures `BUTLER_BASE_URL` at startup, same class of issue.
