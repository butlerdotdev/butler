---
title: Release 2026-04-23 / Production Readiness (Webhook Enforcement + F-SRV-001)
sidebar_position: 2
---

# Release 2026-04-23

Production-readiness round following the 2026-04-22 ADR-009 release. Three tracks plus one pre-existing-bug fix, shipped as a coordinated release across two repos.

## Component versions

| Component | Version | Image / Chart |
|---|---|---|
| butler-controller | no image change (0.15.0) | documented in ADR-012 |
| butler-server | v0.7.0 | `ghcr.io/butlerdotdev/butler-server:0.7.0` |
| butler-controller chart | 0.12.0 (appVersion 0.15.0) | `oci://ghcr.io/butlerdotdev/charts/butler-controller` |

## What ships

### Webhook infrastructure (ADR-012)

The ADR-009 admission webhooks (Team, TenantCluster, NetworkPool, ProviderConfig) now have a chart-side `ValidatingWebhookConfiguration` that routes apiserver admission traffic to the controller's webhook server. Opt-in via `controller.webhooksEnabled: true`; pre-enable default remains off to preserve existing deployments.

The `vteam.kb.io` entry that was missing from the chart in the prior release has been added. Prior to this release, enabling `webhooksEnabled: true` would route TenantCluster / NetworkPool / ProviderConfig traffic but silently skip the Team authority split (platform-admin `spec.resourceLimits` vs team-admin `spec.environments[].limits`).

An optional `selfsigned-issuer` ClusterIssuer template is now gated by `certManager.installSelfSignedIssuer` for self-contained enablement on clusters that lack a pre-provisioned issuer.

`failurePolicy: Fail` on all four webhooks. `namespaceSelector` is cluster-wide so the webhook cannot be silently bypassed by creating resources in an un-labeled namespace.

### F-SRV-001: WebSocket terminal authentication (ADR-013)

The three `/ws/*` endpoints on butler-server (`HandleClusterWatch`, `HandleTerminal`, `HandleManagementTerminal`) now gate on session validity before `upgrader.Upgrade()`. Pre-release state: `/ws/terminal/*` endpoints accepted any connection and loaded cluster kubeconfigs under the server SA; an attacker reaching the endpoint got shells into tenant and management clusters.

Per-endpoint authorization matches REST patterns:

- Tenant terminal: requires team membership in the URL's namespace (or platform admin).
- Management terminal: requires platform admin.
- Cluster watch: requires any authenticated session; filtering applied downstream.

Rejections log `user`, `path`, `remote`, `reason`, and (where applicable) `team` so incident-response triage distinguishes anonymous probes from authenticated-but-unauthorized attempts.

### Management cluster kubeconfig fallback

`GetManagementKubeconfig` previously errored on in-pod deployments because `~/.kube/config` doesn't exist in butler-server's container. The function now falls back to `rest.InClusterConfig()` when the file path is absent and synthesizes a kubeconfig YAML from the pod's mounted ServiceAccount token + CA. Fixes the "Failed to get management cluster kubeconfig" error surfaced on the console's Management GitOps tab.

### F-SRV-002: verified already mitigated

Audit of butler-server's password compare paths confirms `golang.org/x/crypto/bcrypt.CompareHashAndPassword` is used at all user-authentication sites. bcrypt is constant-time by construction (fixed-cost key derivation plus an internal `subtle.ConstantTimeCompare` on the final byte check). Documented inline at `auth/users.go` so future audits don't re-flag. No code change required.

## Rollback and first-enablement guidance

### Rollback: Flux-managed deployments

For clusters where butler-controller is managed by a Flux `HelmRelease` (butler-beta and Company 1 both fall in this category), do not run `helm upgrade` directly. Flux will re-assert the GitOps-repo state on the next sync and undo the override. Two correct paths:

**(a) GitOps-repo values edit.** Update `HelmRelease.spec.values.controller.webhooksEnabled` to `false` in the cluster's GitOps repo, commit, push. Wait for Flux to reconcile (default 10 minutes; `flux reconcile helmrelease butler-controller -n butler-system` forces immediate).

**(b) Suspend + direct upgrade.** When the GitOps round-trip is too slow for incident response:

```bash
flux suspend helmrelease butler-controller -n butler-system
helm upgrade butler-controller \
  oci://ghcr.io/butlerdotdev/charts/butler-controller \
  --reuse-values \
  --set controller.webhooksEnabled=false \
  -n butler-system
```

Fix the underlying issue, update the GitOps values to match the current cluster state, then:

```bash
flux resume helmrelease butler-controller -n butler-system
```

The resume must not drift values relative to what's deployed, or Flux will re-enable webhooks on the next sync.

### Rollback: direct-Helm deployments (non-Flux clusters)

```bash
helm upgrade butler-controller \
  oci://ghcr.io/butlerdotdev/charts/butler-controller \
  --reuse-values \
  --set controller.webhooksEnabled=false \
  -n butler-system
```

### First-enablement checklist

When enabling webhooks for the first time on a cluster, cert-manager has to provision the webhook Certificate and ca-injector has to populate the `caBundle` on the `ValidatingWebhookConfiguration`. During that window, `failurePolicy: Fail` blocks Team / TenantCluster / NetworkPool / ProviderConfig mutations. Run through the checklist before assuming the gate is live:

1. **Verify cert-manager + ca-injector pods Ready.** `kubectl -n cert-manager get pods`; every pod should show `Ready: True`. Pods in CrashLoopBackOff or ContainerCreating delay cert issuance.
2. **Enable via the cluster's deployment model.** Flux-managed: update `HelmRelease.spec.values.controller.webhooksEnabled: true` (and `certManager.installSelfSignedIssuer: true` if no issuer exists) in the GitOps repo. Direct-Helm: `helm upgrade --set controller.webhooksEnabled=true --set certManager.installSelfSignedIssuer=true`.
3. **Wait for the Certificate to be Ready.** `kubectl -n butler-system get certificate butler-controller-webhook-cert -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'` must return `True`. Typical elapsed time: a few seconds to a minute.
4. **Verify `caBundle` populated on the webhook config.** `kubectl get validatingwebhookconfiguration butler-controller -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | wc -c` must be non-zero (typically ~1500 characters for a self-signed cert).
5. **Smoke-test a benign Team read.** `kubectl get teams` must succeed. Reads don't fire the webhook, but a green read confirms the apiserver is still responsive to the tenancy API group. Then attempt an intentional benign Team mutation (e.g., edit the `displayName` field on a test Team) to confirm the webhook is serving.

If any step fails, roll back via the procedure above before investigating.

## Upgrade notes

Backward-compatible. Existing clusters without `webhooksEnabled: true` behave identically; the new chart entries gate on the values flag. Existing WS clients (butler-console's cluster-watch subscription, cluster-detail terminal) continue to work because browser cookies flow automatically during same-origin WS upgrade handshakes.

## Operational notes

### Webhook cert rotation

`Certificate` duration is 8760h (1 year) with `renewBefore: 720h` (30 days). cert-manager automatically rotates. controller-runtime's certwatcher polls the mounted secret every 10 seconds and reloads TLS on change. cert-manager ca-injector updates the `caBundle` on the `ValidatingWebhookConfiguration` when the Certificate renews. No operator action required.

### Company 1 deploy sequence

This release is required before Company 1's ADR-009 enforcement rollout. Recommended sequence:

1. Flux picks up the new butler-controller chart version (webhooks remain dormant per default).
2. Flux picks up butler-server v0.7.0 (WS auth activates, kubeconfig fix activates).
3. Update the cluster's GitOps values: `controller.webhooksEnabled: true` and `certManager.installSelfSignedIssuer: true` (or `certManager.issuerRef` pointing at an existing issuer).
4. Flux reconciles the HelmRelease with the new values.
5. Follow the first-enablement checklist above.
6. Run the ADR-009 enforcement scenarios end-to-end (platform-admin resourceLimits gate, team-admin env-limits gate, per-member cap, env-label immutability).

## Company 1 pre-deploy checklist status

| Item | State |
|---|---|
| Team environments (ADR-009) | SHIPPED (2026-04-22) |
| GitLab provider + export UX | SHIPPED (2026-04-22) |
| Webhook infrastructure | SHIPPED (this release) |
| F-SRV-001 unauthenticated WS terminal | SHIPPED (this release) |
| F-SRV-002 constant-time password compare | SHIPPED (this release; verified already mitigated) |
| Management cluster kubeconfig (console GitOps tab) | SHIPPED (this release) |
| F-DOC-001 Apache 2.0 Kamaji attribution | REMAINING (tracked separately) |

## References

- [ADR-012 Admission Webhook Deployment](https://github.com/butlerdotdev/butler-controller/blob/main/docs/architecture/ADR-012-admission-webhook-deployment.md)
- [ADR-013 WebSocket Authentication](https://github.com/butlerdotdev/butler-server/blob/main/docs/architecture/ADR-013-websocket-authentication.md)
- [Prior release: ADR-009 Team Environments + GitLab GitOps](/releases/2026-04-22)
