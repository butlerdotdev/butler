---
title: Post-Bootstrap Configuration
sidebar_position: 7
---

# Post-Bootstrap Configuration

After `butleradm bootstrap` finishes, the management cluster runs but is not yet reachable from outside the cluster network, has no TLS, and has only the bootstrap-generated admin credentials (a legacy admin env pair on `butler-console-server` plus a backing `User` CRD named `admin`). Work through the sections below before inviting users or creating tenant clusters.

Every step on this page runs against the management cluster's kubeconfig:

```bash
export KUBECONFIG=~/.butler/<cluster-name>-kubeconfig
kubectl get nodes
```

## Three Ways to Apply These Changes

Each step produces the same Kubernetes state; the paths differ only in how the change lands. Pick based on how your team operates:

- **kubectl**. Shortest path, no prerequisite setup. Right for first-time configuration, dev clusters, and verifying something works. This guide's primary examples use kubectl because it works for every step.
- **Console UI**. Right for ongoing administration once the console is reachable. Requires steps 1 and 2 to be complete first. Some steps (infrastructure like ingress and TLS, or server env vars) are not surfaced in the UI because they are not platform state; those are called out per step.
- **GitOps**. Right when the cluster is managed declaratively and all changes go through PR review. Requires Flux to be bootstrapped on the management cluster first (separate operations guide). Adds a review step to every change; drift goes down over time.

Which paths each step supports:

| Step | kubectl | Console UI | GitOps |
|---|---|---|---|
| 1. Expose the console | yes | no (infrastructure, not platform state) | yes |
| 2. Configure `butler-server` env | yes | no (server config, not a CRD) | yes, via Helm values |
| 3. Configure SSO | yes (env vars on `butler-console-server`) | partial (`Admin → Identity Providers` writes the CRD but does not set env vars; see step 3) | yes (chart values + CRD manifest) |
| 4. Create admin user + invite | yes | yes (`Admin → Users`, then `Resend Invite` on an existing row or the modal shown on create) | partial, see step 4 |
| 5. Tune `ButlerConfig` | yes | yes (`Admin → Settings`) | yes |
| 6. Verify | `butlerctl login` + curl | browser to `https://console.yourdomain` | not applicable |

The numbered walkthrough below uses kubectl as the primary path. Callouts on each step show the Console UI and GitOps equivalents.

## 1. Expose the Console

Bootstrap already installs an Ingress for the console. You change its host to one you own, add TLS, and create a DNS record pointing at the ingress controller's LoadBalancer IP.

### What bootstrap installed

Inspect the Ingress the bootstrap controller created:

```bash
kubectl -n butler-system get ingress butler-console -o yaml
```

Expect a resource with `ingressClassName: traefik`, a placeholder host matching your cluster name (for example `butler.butler-hvstr-test.local`), and three path rules:

| Path | Backend Service | Purpose |
|---|---|---|
| `/api` | `butler-console-server:8080` | REST API |
| `/ws` | `butler-console-server:8080` | Terminal and cluster-watch WebSocket |
| `/` | `butler-console-frontend:80` | Static web UI |

Do not delete this Ingress and create a new one. Do not create a second Ingress that overlaps. Edit the existing resource in place.

### DNS

Bootstrap installs Traefik as the default ingress controller in the `traefik` namespace. Its LoadBalancer IP comes from the MetalLB pool declared in `network.loadBalancerPool` (typically the first IP of the pool):

```bash
kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Create an A or CNAME record pointing `console.yourdomain` at that IP. If you plan to expose tenant-cluster API servers via shared ingress later (the `Ingress` mode of `ButlerConfig.spec.controlPlaneExposure`), also create a wildcard `*.k8s.yourdomain` pointing at the same IP.

As of butler-cli v0.7.4, bootstrap installs Traefik unconditionally; the ingress controller is not configurable from the bootstrap config (`butler-bootstrap/internal/addons/installer.go` hardcodes the chart). Swapping to a different controller post-install is possible but out of scope for this guide.

### TLS certificate

`cert-manager` is installed during bootstrap in the `cert-manager` namespace. It does not ship with a `ClusterIssuer` by default; create one first:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: you@yourdomain
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

For air-gapped or internal-only deployments, replace the `ClusterIssuer` body with `selfSigned: {}` and skip the solver.

### Update the Ingress host and TLS

The Ingress is managed by the `butler-console` Helm chart. Update the chart values and re-apply the release; the chart regenerates the Ingress with the `/api`, `/ws`, and `/` path rules intact.

```yaml
# values.yaml override
ingress:
  enabled: true
  className: traefik
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: console.yourdomain
      paths:
        - path: /api
          pathType: Prefix
          service: server
        - path: /ws
          pathType: Prefix
          service: server
        - path: /
          pathType: Prefix
          service: frontend
  tls:
    - hosts:
        - console.yourdomain
      secretName: butler-console-tls
```

Apply with `helm upgrade` (or let Flux reconcile, under GitOps). Wait for the certificate to issue:

```bash
kubectl -n butler-system get certificate butler-console-tls -w
```

:::tip Break-glass: kubectl patch
For one-off clusters where you do not want to touch Helm values, patch the live Ingress directly:

```bash
kubectl -n butler-system annotate ingress butler-console \
  cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite

kubectl -n butler-system patch ingress butler-console --type=json -p='[
  {"op":"replace","path":"/spec/rules/0/host","value":"console.yourdomain"},
  {"op":"add","path":"/spec/tls","value":[{"hosts":["console.yourdomain"],"secretName":"butler-console-tls"}]}
]'
```

**This patch is ephemeral.** The next `helm upgrade` or Flux reconcile of the `butler-console` chart will revert it. Use only when you are not running Helm-managed upgrades.
:::

## 2. Configure `butler-server` for the Public URL

The server emits its public URL in several places: the CLI device-flow verification link and invite emails. By default the server derives this URL from the incoming request, but you must still tell it to trust the ingress-supplied forwarded headers.

Set these in the `butler-console` Helm chart values:

```yaml
# values.yaml override
server:
  config:
    baseURL: https://console.yourdomain
  auth:
    secureCookies: true
```

| Chart value | Env var | Reason |
|---|---|---|
| `server.config.baseURL` | `BUTLER_BASE_URL` | Canonical public URL. Used for invite links and as the fallback when request derivation is not possible. |
| `server.auth.secureCookies` | `BUTLER_SECURE_COOKIES` | Sets the `Secure` flag on session cookies. Required once TLS is active. |

Apply with `helm upgrade` (or let Flux reconcile).

#### `BUTLER_TRUST_PROXY_HEADERS` chart gap

The third env var you want set is `BUTLER_TRUST_PROXY_HEADERS=true`, which tells the server to honor `X-Forwarded-Proto` and `X-Forwarded-Host` from the ingress. Without it, `https` flows show up as `http` in emitted URLs. As of `butler-console` chart 0.4.1, this env var is not yet exposed through chart values; until the chart catches up, set it via `kubectl set env`:

```bash
kubectl -n butler-system set env deployment/butler-console-server \
  BUTLER_TRUST_PROXY_HEADERS=true
kubectl -n butler-system rollout status deployment/butler-console-server
```

This patch will be reverted on the next `helm upgrade` of the chart until the values key lands. Track the chart update and re-apply on each upgrade.

:::warning
Only set `BUTLER_TRUST_PROXY_HEADERS=true` when you are confident the ingress strips client-supplied `X-Forwarded-*` headers and sets its own. Traefik, nginx-ingress, and Envoy can all be configured to do this; check your deployment's forwarded-headers settings (for example, nginx-ingress `use-forwarded-headers` defaults to `false`, which has the effect of ignoring upstream values and writing its own; Traefik requires explicit trusted-IP configuration on the entrypoint). Verify by curling the ingress from outside the cluster with a spoofed `X-Forwarded-Host` header and checking the access log or the response on the ingress pod.
:::

:::tip Break-glass: kubectl set env
For one-off dev clusters where you do not want to touch Helm values, all three env vars can be set directly:

```bash
kubectl -n butler-system set env deployment/butler-console-server \
  BUTLER_BASE_URL=https://console.yourdomain \
  BUTLER_TRUST_PROXY_HEADERS=true \
  BUTLER_SECURE_COOKIES=true
kubectl -n butler-system rollout status deployment/butler-console-server
```

Reverted on the next `helm upgrade`.
:::

## 3. Configure SSO

:::note Platform direction for this section
The shape of SSO configuration is mid-migration. Reconciliation of `IdentityProvider` CRDs into the running server is tracked at [butlerdotdev/butler#21](https://github.com/butlerdotdev/butler/issues/21). Today the env var path described below is the supported configuration mechanism; once the tracked decision lands, this section may flip to CRD-primary.
:::

Butler supports OIDC (Google Workspace, Microsoft Entra, Okta, Keycloak, and any standards-compliant provider). There are two related pieces you may need to configure:

- **`butler-console-server` environment variables**: the server builds its OIDC provider from `BUTLER_OIDC_*` env vars once at startup. These drive the active SSO login flow.
- **`IdentityProvider` CRD**: a cluster-scoped record whose `metadata.name` is referenced by `Team` group-sync rules for existence checks, and is displayed in the console admin UI. Its spec fields (`issuerURL`, `clientID`, `clientSecretRef`, `redirectURL`, `scopes`, claim mappings) are not consumed by any active code path today. They exist in the schema and the admin UI for the day reconciliation lands.

Enabling SSO means setting env vars. Creating the CRD is optional and only necessary if you plan to reference a named IdP from `Team.spec.access.groups` group-sync rules.

### Register the OIDC client with your IdP

Create an OAuth client in your IdP and set the redirect URL to:

```
https://console.yourdomain/api/auth/callback
```

Note the issuer URL, client ID, and client secret. Google Workspace additionally requires an Admin SDK service account for group fetching (groups are not in Google OIDC tokens).

### Set the OIDC configuration on `butler-console-server`

Use the `butler-console` chart's `server.oidc.*` values. The chart reads OIDC credentials from a referenced Secret (`server.oidc.existingSecret`) with two keys, `client-id` and `client-secret`; the Secret keeps the credentials out of your values file.

Create the Secret:

```bash
kubectl -n butler-system create secret generic butler-oidc \
  --from-literal=client-id='<from-idp>' \
  --from-literal=client-secret='<from-idp>'
```

Then set the chart values:

```yaml
# values.yaml override
server:
  oidc:
    enabled: true
    issuerURL: https://accounts.google.com
    existingSecret: butler-oidc
    redirectURL: https://console.yourdomain/api/auth/callback
    groupsClaim: groups
    emailClaim: email
```

`enabled: true` is redundant when `issuerURL` and `clientID` (or `existingSecret`) are set, because `config.go` auto-enables OIDC in that case; it is kept here for explicitness.

Google Workspace only: also set `GOOGLE_SERVICE_ACCOUNT_JSON` (the service account key JSON) and `GOOGLE_ADMIN_EMAIL` (an admin user for domain-wide delegation) so `butler-server` can fetch groups via the Admin SDK. These are not yet exposed as chart values; set them via `kubectl set env --from=secret/...` against a Secret that contains both keys, and track the chart update.

Apply with `helm upgrade` (or let Flux reconcile). The release rolls the `butler-console-server` Deployment and the new OIDC provider is built at startup.

Verify the provider is now advertised:

```bash
curl -sS https://console.yourdomain/api/auth/providers | jq .providers
```

Expect an entry with your configured provider (for example `[{"name":"Google","type":"oidc",...}]`). An empty array means the env vars did not land or the rollout did not complete.

### Optional: create the IdentityProvider CRD

Create this only if you plan to use group-sync rules that name this provider. The CRD is reconciled by the console's admin CRUD handlers; the auth code path does not consume it today.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: company-sso-secret
  namespace: butler-system
type: Opaque
stringData:
  client-secret: <from-idp>
---
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: IdentityProvider
metadata:
  name: company-sso
spec:
  type: oidc
  displayName: "Company SSO"
  oidc:
    issuerURL: https://accounts.google.com
    clientID: <from-idp>
    clientSecretRef:
      name: company-sso-secret
    redirectURL: https://console.yourdomain/api/auth/callback
    scopes:
      - openid
      - profile
      - email
      - groups
    groupsClaim: groups
    emailClaim: email
```

`spec.oidc.redirectURL` is a required field on the CRD schema. `spec.oidc.clientSecretRef.namespace` is optional; when unset, the admin handlers resolve the Secret in `butler-system` (`config.SystemNamespace`).

Confirm the resource was accepted:

```bash
butleradm idp get company-sso
```

:::tip Console UI path
`Admin → Identity Providers → Create` writes the `IdentityProvider` CRD. The env vars above remain the switch that activates SSO.
:::

:::tip GitOps path
Chart values block plus the optional `IdentityProvider` manifest, committed to your Flux repo. Run the `client-secret` through SOPS or your secret-management layer before commit.
:::

:::note Roadmap
Reconciliation of `IdentityProvider` CRDs into the running server is tracked at [butlerdotdev/butler#21](https://github.com/butlerdotdev/butler/issues/21). Today the env var path is the supported configuration mechanism. Provider-specific reference pages (Google Workspace Admin SDK, Microsoft Entra, Okta) are tracked separately and are not yet published.
:::

## 4. Create Your Admin User

Bootstrap creates two things you can authenticate as:

1. A legacy admin session sourced from `BUTLER_ADMIN_USERNAME` and `BUTLER_ADMIN_PASSWORD` on the `butler-console-server` Deployment. The password is generated and stored in the `butler-console-admin` Secret.
2. A `User` resource named `admin` with `spec.authType: internal` and `spec.isPlatformAdmin: true`. The CRD mirrors the bootstrap admin's identity (email `admin@localhost`) but does not reference the legacy password Secret; password authentication as `admin` goes through the legacy env-var code path, not through the CRD.

Either way, you should log in once as the bootstrap admin, create a real user for yourself, then retire the legacy credentials.

### Option A: SSO user

If you configured SSO in the previous step, log in to the console at `https://console.yourdomain` using SSO. Butler auto-creates a `User` resource for you on first login (name derived from your email, `spec.authType: sso`). Then promote that user to platform admin:

```bash
kubectl get users
kubectl patch user <resource-name> --type=merge \
  -p '{"spec":{"isPlatformAdmin":true}}'
```

A dedicated `butleradm user promote` command is roadmap-tracked; until it lands, the `kubectl patch` path above is the supported workflow.

### Option B: Internal user (password)

Create the User CRD:

```bash
butleradm user create --email admin@yourdomain --admin
```

The CLI creates the resource with `spec.isPlatformAdmin: true` but does not mint the password-set invite URL. The easiest way to get one is through the console UI once it's reachable: sign in as the bootstrap `admin`, go to `Admin → Users`, and click **Resend Invite** on the row for the new user. (When creating a user through the console, the invite URL is shown in a modal immediately after creation; `Resend Invite` is how you get a fresh URL for an already-created user.)

If the console UI is not reachable yet, do it via the HTTP API. The invite endpoint requires a session cookie, so log in first, then call it:

```bash
# Grab the generated bootstrap password
ADMIN_PASSWORD=$(kubectl -n butler-system get secret butler-console-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d)

# Log in; persist the butler_session cookie
curl -sS -c /tmp/butler-cj -X POST \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"admin\",\"password\":\"$ADMIN_PASSWORD\"}" \
  https://console.yourdomain/api/auth/login

# Request an invite URL for the new user, reusing the cookie
curl -sS -b /tmp/butler-cj -X POST \
  https://console.yourdomain/api/admin/users/<resource-name>/invite
```

The response body is `{"inviteUrl":"..."}`. Send the URL to the user; it's one-time, opens a password-set form, and signs them in afterward.

### Retire the legacy admin

:::danger Do not run this until your replacement admin works
Do not strip the legacy credentials until you have signed in successfully as your replacement admin (Option A or Option B above). The `admin` User CRD cannot be authenticated against after env vars are stripped; it has no referenced password Secret, and the `/api/admin/users/admin/invite` endpoint requires an existing session cookie that the stripped admin can no longer mint. If your replacement account does not work, stripping the env vars will lock you out of the cluster with no in-band recovery path.

Confirm the replacement works by signing in fully (console or `butlerctl login`) before running any of the commands below.
:::

Once your replacement account is verified working, remove the legacy credentials from the Deployment:

```bash
kubectl -n butler-system set env deployment/butler-console-server \
  BUTLER_ADMIN_USERNAME- \
  BUTLER_ADMIN_PASSWORD-
kubectl -n butler-system rollout restart deployment/butler-console-server
```

The `User` CRD named `admin` remains after the env vars are removed, but it has no password and no referenced password Secret; nobody can authenticate as it until an invite is regenerated for it and a password is set. Delete it if you do not want it sitting dormant:

```bash
kubectl delete user admin
```

:::tip Console UI path
`Admin → Users`. `Add New User` creates the `User` CRD and opens an invite-URL modal. `Resend Invite` on an existing row calls `POST /api/admin/users/{name}/invite`. Platform-admin promotion is not a UI toggle today; use the `kubectl patch` shown above.
:::

:::tip GitOps path (partial)
Commit `User` CRDs to your Flux repo. The invite URL itself is minted at claim time by `butler-server` and is not part of CRD state; operators still regenerate it via UI or API per user.
:::

:::note
The invite URL is built from `BUTLER_BASE_URL` captured at server startup. Migrating it to per-request derivation is tracked as a follow-up to the butler-server v0.5.5 device-flow fix.
:::

## 5. Tune `ButlerConfig`

`ButlerConfig` is the cluster-scoped singleton that controls platform-wide defaults. Bootstrap creates it with sensible defaults, but review the values before onboarding teams.

```bash
kubectl edit butlerconfig butler
```

Fields worth reviewing:

| Field | Default | Notes |
|---|---|---|
| `spec.multiTenancy.mode` | `Optional` (bootstrap sets this; CRD-level default if the field is unset is `Disabled`) | `Disabled` means no Team scoping. `Optional` lets `TenantCluster` resources attach to a `Team` but does not require it. `Enforced` requires every cluster to belong to a `Team` with quota enforcement. |
| `spec.defaultNamespace` | `butler-tenants` | Namespace for TenantClusters in `Disabled` or `Optional` mode when no Team is specified. |
| `spec.defaultProviderConfigRef` | (unset) | References the default `ProviderConfig` for new tenant clusters that do not specify one. |
| `spec.defaultTeamLimits` | (unset) | Platform-wide per-Team defaults (`maxClusters`, `maxWorkersPerCluster`). Admins can override per Team. |
| `spec.defaultControlPlaneResources` | bootstrap defaults | Default CPU and memory for `TenantControlPlane` apiserver, controller-manager, and scheduler pods. If unset, pods run BestEffort QoS. |
| `spec.controlPlaneExposure` | set by bootstrap | How tenant API servers are reached. `LoadBalancer` gives each tenant its own IP; `Ingress` or `Gateway` share one IP across tenants via SNI. |

Apply changes with `kubectl apply` or `kubectl edit`; the controller reconciles in seconds.

:::tip Console UI path
`Admin → Settings`. Sections cover general settings, control plane exposure, default addon versions, default team limits, default control plane resources, image factory, audit log, notifications, and the platform SSH authorized key. Writes back to the same singleton.
:::

:::tip GitOps path
Commit the `ButlerConfig` singleton (cluster-scoped, named `butler`) to your Flux repo. Avoid `kubectl edit` on Flux-managed clusters; edits are reverted on reconcile.
:::

## 6. Verify

The verification step is a check, not an apply, so there's no GitOps variant. Exercise both the Console UI and the CLI flow:

### Web console

Open `https://console.yourdomain`. Sign in via SSO or the internal admin account you created. You should see the dashboard with the management cluster listed.

### CLI login

```bash
butlerctl login --server https://console.yourdomain
```

The command prints a one-time user code and opens your browser to the approval URL. Confirm the code matches, approve, and the CLI reports a successful login:

```
Logged in as you@yourdomain
  Teams: ...
  Active team: ...
  Credentials saved to ~/.butler/credentials.json
```

:::tip
If the verification URL points at `http://localhost:8080` instead of your real URL, step 2 did not take. Check `kubectl set env` ran against the right Deployment and that the pods restarted.
:::

### Auth endpoint curl

```bash
curl -sS https://console.yourdomain/healthz
curl -sS https://console.yourdomain/api/auth/providers | jq
```

The first returns `ok`; the second lists your configured `IdentityProvider` resources.

## Next Steps

With the platform reachable, authenticated, and tuned:

1. [Create your first tenant cluster](./first-tenant-cluster.md)
2. [Tour the console](./console.md)
3. [Day-2 operations](../operations/): upgrades, monitoring, backup, scaling

Optional: if the team plans to operate at multiple risk tiers (dev, prod) or wants per-user sandbox caps, define [environments](../concepts/environments.md) on the team before users start creating clusters. Environments are optional — a team with no environments works fine; adding them later is supported but existing clusters stay outside env accounting until migrated.

## Troubleshooting

| Symptom | Check |
|---|---|
| CLI verification URL shows `localhost:8080` | `BUTLER_BASE_URL` is unset or pods have not restarted since you set it. |
| CLI verification URL uses `http://` despite a TLS ingress | `BUTLER_TRUST_PROXY_HEADERS` is not `true`, or the ingress is not forwarding `X-Forwarded-Proto`. |
| Session cookie not persisted after login | `BUTLER_SECURE_COOKIES` is `true` but the connection to the browser is HTTP. Terminate TLS before the request reaches the server. |
| OIDC callback returns "invalid redirect URI" | The OAuth client in your IdP is registered with a different callback URL than `BUTLER_OIDC_REDIRECT_URL`. Match them exactly; the path is `/api/auth/callback` on your public hostname. |
| Configured an `IdentityProvider` CRD but SSO login button still missing | The CRD does not drive auth today. Set the `BUTLER_OIDC_*` env vars on `butler-console-server` and roll the Deployment (step 3). |
| Certificate stuck in `Pending` | `kubectl describe certificate butler-console-tls -n butler-system`. Check the cert-manager logs and the `ClusterIssuer` status. |
| `butleradm user create` succeeds but the user has no way to set a password | The CLI only creates the User resource; the invite URL comes from the server's `/api/admin/users/{name}/invite` endpoint (the `Resend Invite` button in the console, or the modal shown on create). See section 4. |
| `/api/admin/users/.../invite` returns `unauthorized` | The endpoint requires a session cookie, not HTTP basic auth. Log in first via `POST /api/auth/login` with `{"username":...,"password":...}` and reuse the `butler_session` cookie. |
| `/api/admin/users/.../invite` returns an invite URL pointing at localhost | `BUTLER_BASE_URL` is unset on the server. The invite URL is constructed at server startup from this value. Set it per step 2 and roll the Deployment. |

See [Troubleshooting](../troubleshooting/) for broader platform issues.
