---
title: AddonDefinition
sidebar_position: 10
---

An AddonDefinition is a cluster-scoped catalog entry that describes an addon available for installation on tenant or management clusters.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Cluster

## Short Name

`ad`

## Description

Each AddonDefinition specifies a Helm chart, its default configuration, and metadata used by Butler Console for catalog display. Platform administrators manage the catalog by creating, updating, or removing AddonDefinitions. Butler ships with 27 built-in definitions; organizations can add their own.

When a user installs an addon on a cluster, Butler creates a TenantAddon or ManagementAddon that references the AddonDefinition by name. The butler-controller uses the definition to locate the Helm chart, merge default values with user overrides, and install the release.

## Specification

### Full Example

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: AddonDefinition
metadata:
  name: prometheus-operator
spec:
  displayName: "Prometheus Operator"
  description: "Monitoring stack operator providing ServiceMonitor and PrometheusRule CRDs"
  category: observability
  platform: false
  tier: infrastructure

  icon: "📊"
  iconData: "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA..."

  chart:
    repository: https://prometheus-community.github.io/helm-charts
    name: kube-prometheus-stack
    defaultVersion: "72.6.2"
    availableVersions:
      - "72.6.2"
      - "71.0.0"
      - "65.8.0"

  defaults:
    namespace: monitoring
    releaseName: prometheus
    createNamespace: true
    timeout: "10m"
    values:
      grafana:
        enabled: false
      alertmanager:
        enabled: true

  dependsOn:
    - cert-manager

  links:
    documentation: https://prometheus-operator.dev/docs/
    source: https://github.com/prometheus-operator/prometheus-operator
    homepage: https://prometheus-operator.dev

  maintainer:
    name: "Platform Team"
    email: "platform@example.com"
```

### Spec Fields

#### Required

| Field | Type | Description |
|-------|------|-------------|
| `displayName` | string (1-64 chars) | Human-readable name displayed in the Console catalog |
| `description` | string (1-512 chars) | What the addon does |
| `category` | enum | Catalog grouping. One of: `cni`, `loadbalancer`, `storage`, `certmanager`, `ingress`, `observability`, `backup`, `gitops`, `security`, `dns`, `database`, `messaging`, `service-mesh`, `other` |
| `chart.repository` | string | Helm chart repository URL (must start with `http://` or `https://`) |
| `chart.name` | string | Chart name within the repository |
| `chart.defaultVersion` | string | Chart version used when a TenantAddon does not specify one |

#### Optional

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `platform` | boolean | `false` | Platform addons are installed during cluster creation and cannot be removed through the Console |
| `tier` | enum | inferred | `infrastructure` or `apps`. Controls directory placement in GitOps exports. Platform addons default to `infrastructure`; all others default to `apps` |
| `icon` | string (max 8 chars) | — | Emoji or short text icon. Used as fallback when `iconData` is not set |
| `iconData` | string (max 131072 chars) | — | Base64-encoded SVG image. Takes priority over `icon` for catalog display |
| `chart.availableVersions` | string[] | — | Other chart versions known to work. Populates the version dropdown in the Console |
| `defaults.namespace` | string | addon name | Kubernetes namespace for the Helm release |
| `defaults.releaseName` | string | addon name | Helm release name |
| `defaults.createNamespace` | boolean | `true` | Whether Butler creates the target namespace if it does not exist |
| `defaults.timeout` | string | `10m` | Timeout for Helm install and upgrade operations |
| `defaults.values` | object | — | Default Helm values. Users can override these in `TenantAddon.spec.values` |
| `dependsOn` | string[] | — | Names of AddonDefinitions that must be installed before this addon |
| `links.documentation` | string | — | URL to the addon's documentation |
| `links.homepage` | string | — | URL to the addon's project homepage |
| `links.source` | string | — | URL to the addon's source code |
| `maintainer.name` | string | — | Maintainer name |
| `maintainer.email` | string | — | Maintainer email |

### iconData Format

The `iconData` field stores a product logo as a base64-encoded SVG string. Butler Console decodes it and renders the image inline in the addon catalog, cluster addon lists, and detail views.

To encode an SVG file:

```bash
# Linux
base64 -w 0 logo.svg

# macOS
base64 -i logo.svg | tr -d '\n'
```

Paste the output directly into `spec.iconData`. Do not include a `data:` URI prefix -- Butler Console adds the appropriate prefix when rendering.

Keep SVGs clean and reasonably sized. Logos from CNCF projects are typically under 10 KB encoded. The hard limit is 128 KB (131,072 characters).

### Tier Inference

When `tier` is omitted:

| Condition | Inferred Tier |
|-----------|--------------|
| `platform: true` | `infrastructure` |
| `platform: false` or unset | `apps` |

Set `tier` explicitly when the default does not match. For example, Flux and External Secrets are optional addons but classified as `infrastructure` because other workloads depend on their CRDs being present.

## Status

AddonDefinitions do not have a status subresource. Their presence in the cluster makes them available in the catalog. Removing an AddonDefinition does not affect already-installed TenantAddons or ManagementAddons that reference it, but prevents new installations.

## See Also

- [Architecture > Addon System](../../architecture/addon-system.md) -- Lifecycle, installation flow, and built-in addon catalog
- [Addon Catalog Operations](../../operations/addon-catalog.md) -- Using the Console to browse and manage addons
- [TenantCluster](./tenantcluster.md) -- Addon fields in the TenantCluster spec
