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

Create an A or CNAME record that points `console.yourdomain` at the ingress LoadBalancer IP. If you configured `loadBalancerPool` during bootstrap, the ingress controller has already received an IP from that pool:

```bash
kubectl get svc -n kube-system traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

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

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: IdentityProvider
metadata:
  name: company-sso
spec:
  displayName: "Company SSO"
  issuerURL: https://login.microsoftonline.com/<tenant-id>/v2.0
  clientID: <from-idp>
  clientSecretRef:
    name: company-sso-secret
    key: client-secret
  scopes:
    - openid
    - profile
    - email
    - groups
  groupsClaim: groups
  emailClaim: email
---
apiVersion: v1
kind: Secret
metadata:
  name: company-sso-secret
  namespace: butler-system
type: Opaque
stringData:
  client-secret: <from-idp>
```

Provider-specific references:

- [Google Workspace (Admin SDK + OIDC)](../reference/sso-google.md)
- [Microsoft Entra (Azure AD)](../reference/sso-entra.md)
- [Okta](../reference/sso-okta.md)

After applying, test discovery:

```bash
butleradm idp test company-sso
```

The server picks up new IdentityProviders without a restart.

## 4. Create Your Admin User

The bootstrap creates a legacy admin account from `BUTLER_ADMIN_USERNAME` and `BUTLER_ADMIN_PASSWORD`. Use it once to create a real `User` resource for yourself, then rotate or disable the legacy account.

### Option A: SSO user

If you configured SSO in the previous step, log in to the console at `https://console.yourdomain` using SSO. Butler auto-creates a `User` for you on first login. Then grant platform admin:

```bash
butleradm user promote <your-email> --platform-admin
```

### Option B: Internal user (password)

```bash
butleradm user invite admin@yourdomain --platform-admin
```

The command prints a one-time invite URL. Open it in a browser, set a password, and sign in.

### Retire the legacy admin

Once your real account works, remove or rotate the legacy credentials in the Deployment:

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
| `spec.multiTenancyMode` | `Optional` | Set to `Enforced` to require every `TenantCluster` to belong to a `Team` with quota enforcement. |
| `spec.defaultProvider` | (unset) | Sets the default `ProviderConfig` for new tenant clusters that do not specify one. |
| `spec.teamLimits` | (unset) | Platform-wide per-team caps on CPU, memory, storage, and cluster count. |
| `spec.defaultControlPlaneResources` | bootstrap defaults | Default CPU and memory for `TenantControlPlane` apiserver, controller-manager, and scheduler pods. |
| `spec.controlPlaneExposure` | `LoadBalancer` | How tenant API servers are reached. `Ingress` or `Gateway` modes share a single IP. |

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
| `butleradm user invite` fails with "server misconfigured" | `BUTLER_BASE_URL` is unset and the command runs outside a request context. Set it via step 2 and retry. |

See [Troubleshooting](../troubleshooting/) for broader platform issues.
