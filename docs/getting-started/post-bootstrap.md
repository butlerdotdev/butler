---
title: Post-Bootstrap Configuration
sidebar_position: 7
---

# Post-Bootstrap Configuration

After `butleradm bootstrap` finishes, the management cluster runs but is not yet reachable from outside the cluster network, has no TLS, and has only a legacy admin account. Work through the sections below before inviting users or creating tenant clusters.

Every step on this page runs against the management cluster's kubeconfig:

```bash
export KUBECONFIG=~/.butler/<cluster-name>-kubeconfig
kubectl get nodes
```

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

Create an A or CNAME record pointing `console.yourdomain` at that IP. If you plan to expose tenant-cluster API servers via shared ingress later, also create a wildcard `*.k8s.yourdomain` pointing at the same IP.

If you chose a different ingress controller during bootstrap, substitute its namespace and Service name; the Ingress `ingressClassName` must match.

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

Patch the bootstrap-created Ingress to use your real hostname and reference the `ClusterIssuer`. This preserves the `/api`, `/ws`, and `/` path rules:

```bash
kubectl -n butler-system annotate ingress butler-console \
  cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite

kubectl -n butler-system patch ingress butler-console --type=json -p='[
  {"op":"replace","path":"/spec/rules/0/host","value":"console.yourdomain"},
  {"op":"add","path":"/spec/tls","value":[{"hosts":["console.yourdomain"],"secretName":"butler-console-tls"}]}
]'
```

Wait for the certificate to issue before continuing:

```bash
kubectl -n butler-system get certificate butler-console-tls -w
```

If you manage the install with Helm, set `ingress.hosts[0].host`, `ingress.tls`, and `ingress.annotations` in the chart values instead of patching the Ingress directly.

## 2. Configure `butler-server` for the Public URL

The server emits its public URL in several places: the CLI device-flow verification link, invite emails, and OIDC callback hints. By default the server derives this URL from the incoming request, but you must still tell it to trust the ingress-supplied forwarded headers.

Set three environment variables on the `butler-console-server` Deployment. If you are managing the install with Helm, set them in the chart values; otherwise edit the Deployment directly:

```bash
kubectl set env deployment/butler-console-server -n butler-system \
  BUTLER_BASE_URL=https://console.yourdomain \
  BUTLER_TRUST_PROXY_HEADERS=true \
  BUTLER_SECURE_COOKIES=true
```

| Variable | Value | Reason |
|---|---|---|
| `BUTLER_BASE_URL` | `https://console.yourdomain` | Canonical public URL. Used for invite links and as the fallback when request derivation is not possible. |
| `BUTLER_TRUST_PROXY_HEADERS` | `true` | Tells the server to honor `X-Forwarded-Proto` and `X-Forwarded-Host` set by the ingress. Without this, `https` flows show up as `http` in emitted URLs. |
| `BUTLER_SECURE_COOKIES` | `true` | Sets the `Secure` flag on session cookies. Required once TLS is active. |

Restart to pick up the new values:

```bash
kubectl rollout restart deployment/butler-console-server -n butler-system
kubectl rollout status deployment/butler-console-server -n butler-system
```

:::warning
Only set `BUTLER_TRUST_PROXY_HEADERS=true` when you are confident the ingress strips client-supplied `X-Forwarded-*` headers and sets its own. Traefik, nginx-ingress, and Envoy do this by default. Verify by curling the ingress from outside the cluster with a spoofed `X-Forwarded-Host` header and checking the access log on the ingress pod.
:::

## 3. Configure SSO

Butler supports OIDC (Google Workspace, Microsoft Entra, Okta, Keycloak, and any standards-compliant provider). Create an `IdentityProvider` CRD and register the callback URL with your IdP.

### Register the OIDC client with your IdP

Create an OAuth client in your IdP and set the redirect URL to:

```
https://console.yourdomain/api/auth/callback
```

Note the issuer URL, client ID, and client secret. Google Workspace additionally requires an Admin SDK service account for group fetching (groups are not in Google OIDC tokens).

### Apply the IdentityProvider

The Secret lives in the same namespace as the IdentityProvider (cluster-scoped resources look up referenced Secrets in `butler-system` by default; set `spec.oidc.clientSecretRef.namespace` explicitly if you store it elsewhere). The Secret must have a key named `client-secret`.

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
    issuerURL: https://login.microsoftonline.com/<tenant-id>/v2.0
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

`spec.oidc.redirectURL` is required and must match the callback URL registered with your IdP exactly. `spec.oidc.clientSecretRef.namespace` is optional and defaults to `butler-system`.

After applying, confirm the resource is accepted:

```bash
butleradm idp get company-sso
```

The server picks up new IdentityProviders without a restart. Provider-specific guides (Google Workspace with Admin SDK group fetching, Microsoft Entra, Okta) are tracked as separate reference pages and are not yet published.

## 4. Create Your Admin User

Bootstrap creates two things you can authenticate as:

1. A legacy admin session sourced from `BUTLER_ADMIN_USERNAME` and `BUTLER_ADMIN_PASSWORD` on the `butler-console-server` Deployment. The password is generated and stored in the `butler-console-admin` Secret.
2. A `User` resource named `admin` with `spec.authType: internal` and `spec.isPlatformAdmin: true`. This is a real CRD record tied to the legacy credentials.

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

The CLI creates the resource with `spec.isPlatformAdmin: true` but does not mint the password-set invite URL. The easiest way to get one is through the console UI once it's reachable: sign in as the bootstrap `admin`, open the new user's page, and click **Regenerate invite**.

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

Once your real account works, remove the legacy credentials from the Deployment:

```bash
kubectl -n butler-system set env deployment/butler-console-server \
  BUTLER_ADMIN_USERNAME- \
  BUTLER_ADMIN_PASSWORD-
kubectl -n butler-system rollout restart deployment/butler-console-server
```

The `User` CRD named `admin` remains; delete it separately if you don't want to keep it as a break-glass internal account.

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

## 6. Verify

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

## Troubleshooting

| Symptom | Check |
|---|---|
| CLI verification URL shows `localhost:8080` | `BUTLER_BASE_URL` is unset or pods have not restarted since you set it. |
| CLI verification URL uses `http://` despite a TLS ingress | `BUTLER_TRUST_PROXY_HEADERS` is not `true`, or the ingress is not forwarding `X-Forwarded-Proto`. |
| Session cookie not persisted after login | `BUTLER_SECURE_COOKIES` is `true` but the connection to the browser is HTTP. Terminate TLS before the request reaches the server. |
| OIDC callback returns "invalid redirect URI" | The OAuth client in your IdP is registered with a different callback URL than the server advertises. Must be exactly `https://<BUTLER_BASE_URL>/api/auth/callback`. |
| Certificate stuck in `Pending` | `kubectl describe certificate butler-console-tls -n butler-system`. Check the cert-manager logs and the `ClusterIssuer` status. |
| `butleradm user create` succeeds but the user has no way to set a password | The CLI only creates the User resource; the invite URL comes from the server's `/api/admin/users/{name}/invite` endpoint (the Regenerate invite button in the console). See section 4. |
| `/api/admin/users/.../invite` returns `unauthorized` | The endpoint requires a session cookie, not HTTP basic auth. Log in first via `POST /api/auth/login` with `{"username":...,"password":...}` and reuse the `butler_session` cookie. |
| `/api/admin/users/.../invite` returns an invite URL pointing at localhost | `BUTLER_BASE_URL` is unset on the server. The invite URL is constructed at server startup from this value. Set it per step 2 and roll the Deployment. |

See [Troubleshooting](../troubleshooting/) for broader platform issues.
