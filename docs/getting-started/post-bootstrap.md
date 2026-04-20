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

The bootstrap installs `butler-console-server` with a `Service` of type `ClusterIP`. You expose it through the ingress controller that bootstrap installs (Traefik by default) with a matching DNS record and TLS certificate.

### DNS

Create an A or CNAME record that points `console.yourdomain` at the ingress LoadBalancer IP. If you configured `loadBalancerPool` during bootstrap, the ingress controller has already received an IP from that pool. The ingress installed by default is Traefik, in the `traefik` namespace:

```bash
kubectl get svc -n traefik traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

If you chose a different ingress controller during bootstrap, substitute the namespace and Service name.

If you plan to expose tenant-cluster API servers on shared ingress later, also create a wildcard record `*.k8s.yourdomain` pointing at the same IP.

### TLS certificate

`cert-manager` is installed during bootstrap. Create a `ClusterIssuer` and an `Ingress` that references it. The example below uses Let's Encrypt HTTP-01:

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
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: butler-console
  namespace: butler-system
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
    - host: console.yourdomain
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: butler-console-server
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: butler-console-frontend
                port:
                  number: 80
  tls:
    - hosts:
        - console.yourdomain
      secretName: butler-console-tls
```

The console ships as two Deployments: a static frontend (`butler-console-frontend`) and the API server (`butler-console-server`). The Ingress routes `/api` to the server and everything else to the frontend. If you prefer to manage the Ingress through Helm, the `butler-console` chart exposes equivalent `ingress.*` values.

For air-gapped or internal-only deployments, replace the `ClusterIssuer` with a self-signed issuer and skip the `http01` solver.

Wait for the certificate to be ready before continuing:

```bash
kubectl get certificate -n butler-system butler-console-tls -w
```

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
    scopes:
      - openid
      - profile
      - email
      - groups
    groupsClaim: groups
    emailClaim: email
```

After applying, confirm the resource is accepted:

```bash
butleradm idp get company-sso
```

The server picks up new IdentityProviders without a restart. Provider-specific guides (Google Workspace with Admin SDK group fetching, Microsoft Entra, Okta) are tracked as separate reference pages and are not yet published.

## 4. Create Your Admin User

The bootstrap creates a legacy admin account from `BUTLER_ADMIN_USERNAME` and `BUTLER_ADMIN_PASSWORD`. Use it once to create a real `User` resource for yourself, then retire the legacy account.

### Option A: SSO user

If you configured SSO in the previous step, log in to the console at `https://console.yourdomain` using SSO. Butler auto-creates a `User` resource for you on first login (name derived from your email, `spec.authType: sso`). Then promote that user to platform admin with `kubectl`:

```bash
# Resource names use the email local part plus a hash suffix.
# List to find yours, then patch it.
kubectl get users
kubectl patch user <resource-name> --type=merge \
  -p '{"spec":{"isPlatformAdmin":true}}'
```

A dedicated `butleradm user promote` command is on the roadmap; until it lands, the `kubectl patch` path above is the supported workflow.

### Option B: Internal user (password)

Create the User CRD via the CLI:

```bash
butleradm user create --email admin@yourdomain --admin
```

The CLI creates the resource with `spec.isPlatformAdmin: true`. It does not itself mint the password-set invite URL. To get the invite URL, call the console's admin endpoint with the legacy admin credentials (the console UI exposes this as a "Regenerate invite" action per user):

```bash
curl -u admin:<legacy-password> \
  -X POST https://console.yourdomain/api/admin/users/<resource-name>/regenerate-invite
```

The response body contains the one-time URL. Send it to the user. They open it in a browser, set a password, and sign in.

### Retire the legacy admin

Once your real account works, remove the legacy credentials from the Deployment:

```bash
kubectl set env deployment/butler-console-server -n butler-system \
  BUTLER_ADMIN_USERNAME- \
  BUTLER_ADMIN_PASSWORD-
kubectl rollout restart deployment/butler-console-server -n butler-system
```

## 5. Tune `ButlerConfig`

`ButlerConfig` is the cluster-scoped singleton that controls platform-wide defaults. Bootstrap creates it with sensible defaults, but review the values before onboarding teams.

```bash
kubectl edit butlerconfig butler
```

Fields worth reviewing:

| Field | Default | Notes |
|---|---|---|
| `spec.multiTenancy.mode` | `Disabled` | Set to `Optional` to allow `TenantCluster` resources to attach to a `Team` but not require it. Set to `Enforced` to require every cluster to belong to a `Team` with quota enforcement. |
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
| `butleradm user create` succeeds but the user has no way to set a password | The CLI only creates the User resource; the invite URL comes from the server's `/api/admin/users/{name}/regenerate-invite` endpoint. See section 4. |
| `/api/admin/users/.../regenerate-invite` returns `server misconfigured: cannot determine public URL` | `BUTLER_BASE_URL` is unset. Set it in step 2. |

See [Troubleshooting](../troubleshooting/) for broader platform issues.
