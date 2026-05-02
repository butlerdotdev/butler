---
title: Addon Catalog
sidebar_position: 6
---

# Addon Catalog

Butler Console provides three views for managing addons. Each view serves a different scope and audience.

## Admin Addon Catalog

**Path:** Admin > Addon Catalog

The catalog lists every AddonDefinition in the cluster. Platform administrators use it to review available addons, add custom definitions, and check which addons are built-in versus custom.

Each entry shows the addon's icon, display name, category, tier, and description. Built-in addons (shipped with Butler) are read-only -- they can be viewed but not edited or deleted through the Console. Custom addons can be created, edited, and removed.

### Browsing the Catalog

The catalog grid groups addons by category. Use the search bar to filter by name or description, or the category tabs to narrow the view. Each addon card shows:

- **Icon** -- Product logo (from `iconData`) or emoji fallback
- **Display name** and addon name
- **Category** badge (e.g., observability, security)
- **Tier** badge -- `infrastructure` or `apps`
- **Source** -- `builtin` for addons shipped with Butler, `custom` for user-created definitions

Click an addon card to view its full definition, including chart details, default values, dependencies, and available versions.

### Adding a Custom Addon

Click **Create Addon** to open the creation form. Fill in:

1. **Name** -- A lowercase slug used as the AddonDefinition resource name (e.g., `my-monitoring-tool`). Cannot be changed after creation.
2. **Display Name** -- Human-readable name shown in the catalog.
3. **Description** -- What the addon does.
4. **Category** -- Select from the category dropdown.
5. **Chart Repository** -- The Helm chart repository URL (e.g., `https://charts.company.com`).
6. **Chart Name** -- The chart name within the repository.
7. **Default Version** -- The chart version to install by default.
8. **Tier** -- Select `infrastructure` or `apps`, or leave on Auto to let Butler infer it.

Optionally, set an icon by encoding your SVG logo:

```bash
# macOS
base64 -i logo.svg | tr -d '\n' | pbcopy

# Linux
base64 -w 0 logo.svg | xclip -selection clipboard
```

Paste the encoded string into the **Icon Data** field.

Common issues when creating custom addons:

- **Chart not reachable** -- Butler validates the chart repository URL. If the repository requires authentication, create a credentials Secret first (see [Private Helm Repositories](../architecture/addon-system.md#private-helm-repositories)).
- **Name collision** -- Addon names must be unique across the cluster. Check existing names with `kubectl get addondefinitions`.
- **Large SVGs** -- The `iconData` field has a 128 KB limit. Simplify complex logos or use the `icon` emoji field instead.

## Management Cluster Addons

**Path:** Admin > Management

The management view shows addons installed on the management cluster itself. These are typically observability and platform tools -- Victoria Metrics for long-term metrics, Grafana for dashboards, and similar infrastructure services.

### Installing a Management Addon

Click **Install Addon** to browse the catalog filtered to addons suitable for the management cluster. Select an addon, choose a version, and optionally override default values.

Management addons are cluster-scoped ManagementAddon resources. Only platform administrators can install or remove them.

### Viewing Status

Each installed addon shows its current phase:

| Phase | Meaning |
|-------|---------|
| **Installed** | Helm release is deployed and healthy |
| **Installing** | Helm install is in progress |
| **Upgrading** | A version or values change is being applied |
| **Failed** | The Helm operation encountered an error. Check the addon detail view for the error message |
| **Uninstalling** | The addon is being removed |

## Tenant Cluster Addons

**Path:** Cluster detail > Addons tab

The addons tab on a cluster detail page shows three sections:

### Platform Addons

Read-only list of addons installed during cluster creation (Cilium, MetalLB, cert-manager, etc.). These cannot be removed through the Console. The list shows the installed version and health status.

### Installed Optional Addons

Addons that a user has installed on this cluster. Each entry shows the addon icon, version, and status. From here you can:

- **View values** -- See the Helm values applied to the release.
- **Edit values** -- Modify Helm values and trigger an upgrade.
- **Change version** -- Select a different version from the available versions list.
- **Uninstall** -- Remove the addon from the cluster.

### Available Addons

Addons from the catalog that are not yet installed on this cluster. Click **Install** to add one. Butler creates a TenantAddon resource and the controller handles the Helm installation.

## Tier Classification Guidance

When creating custom addons, choosing the right tier affects GitOps export ordering.

**Use `infrastructure` when:**

- The addon provides CRDs that other workloads consume (e.g., database operators, certificate issuers, secrets managers)
- The addon must be running before application workloads can start
- Removing the addon would break dependent services

**Use `apps` when:**

- The addon is a standalone tool with no downstream dependents
- The addon consumes infrastructure but does not provide it
- Other workloads do not reference the addon's CRDs or services

For reference, here is how the built-in addons are classified:

| Tier | Reasoning | Examples |
|------|-----------|---------|
| `infrastructure` | Provides CRDs or services that other addons depend on | Cilium (CNI), cert-manager (Certificates), Flux (GitOps CRDs), External Secrets (SecretStore CRD), CloudNativePG (Cluster CRD) |
| `apps` | Standalone tools with no downstream dependents | Tempo (tracing), Velero (backup), Redis (cache), Jaeger (tracing), Victoria Metrics (metrics storage) |

## Status Reference

Addon status in the Console is indicated by color-coded badges:

| Status | Color | Description |
|--------|-------|-------------|
| Installed | Green | Helm release is deployed and running |
| Installing | Blue | Initial Helm install in progress |
| Upgrading | Blue | Version or values change being applied |
| Failed | Red | Helm operation failed. Expand the addon detail for the error message |
| Pending | Yellow | Addon created but not yet picked up by the controller |
| Uninstalling | Yellow | Helm uninstall in progress |

For failed addons, the detail view shows the Helm error output. Common causes:

- **Image pull errors** -- The chart references a container image not accessible from the cluster. Check network policies and registry credentials.
- **Timeout** -- The Helm operation exceeded the configured timeout. Increase `defaults.timeout` in the AddonDefinition or check for resource constraints.
- **Dependency not met** -- An addon listed in `dependsOn` is not installed. Install the dependency first.
- **Values conflict** -- Invalid Helm values. Check the chart's `values.yaml` for the correct schema.

## See Also

- [Architecture > Addon System](../architecture/addon-system.md) -- How the addon system works internally
- [AddonDefinition CRD Reference](../reference/crds/addondefinition.md) -- Full field specification
- [Concepts: Addons](../concepts/addons.md) -- Addon types and CRD overview
