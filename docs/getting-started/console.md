---
title: Butler Console
sidebar_position: 9
---

# Butler Console

The Butler Console is a web UI for managing tenant clusters, addons, and images. Bootstrap installs it automatically.

## Access the Console

After bootstrap, find the Console URL:

```bash
kubectl get svc -n butler-system butler-console
```

On-prem deployments expose the Console through a Traefik ingress. Cloud deployments use a cloud load balancer. The URL appears in the bootstrap output under "Console URL."

## First Login

Butler creates an admin user during bootstrap. Set the initial password in your bootstrap config under `addons.console.auth.adminPassword` (defaults to `admin`). Log in with the admin account using that password.

To add more users, create User CRDs or configure an [IdentityProvider](../reference/crds/) for SSO with Google, Microsoft Entra ID, or Okta.

## What You Can Do

### View Clusters

The Clusters page lists all TenantClusters across teams. Each cluster shows its phase, Kubernetes version, worker count, and control plane endpoint. Click a cluster to see conditions, events, and addon status.

### Open Terminal Sessions

Click **Terminal** on any tenant cluster to open a kubectl session in the browser. The terminal connects through butler-server's WebSocket proxy using the cluster's kubeconfig.

### Manage Addons

The Addons tab on each cluster shows installed addons and their health. Add or remove addons from the AddonDefinition catalog without editing YAML.

### Sync Images

The Images page lists OS images available from Butler Image Factory. Create ImageSync resources to push images to your infrastructure providers. The console tracks sync progress and shows which providers have each image.

### Manage Teams

Platform administrators can create teams, assign users with roles (admin, operator, viewer), and set resource quotas.

### Manage Environments

Teams that opt into [environments](../concepts/environments.md) get a dedicated **Environments** item in the team sidebar under Workloads. Team admins and platform admins can create, edit, and delete envs from there; the rest of the team sees the read-only list.

The console surfaces envs in several places:

- **Environment picker** in the header, next to the team picker. Switching scopes the active session to one env; the cluster list, create form, and mutation handlers all pick up the selection. "All environments" is the default and clears the scope.
- **Cluster list** groups by env when the team has any defined and the header picker is on "All environments". Each section shows its cap, utilization, and a per-member-cap badge when set. Sections are collapsible and the state persists per team.
- **Create cluster form** shows an env selector when the team has envs. Env `clusterDefaults` pre-fill worker/version fields; overridden fields carry a "from env default" hint. Per-member cap visibility renders "You own N of M clusters in env X" near the picker — red when at cap, with submit disabled.
- **Change environment** action on each cluster detail page moves a single cluster between envs. Sets the migration-operation annotation automatically.
- **Migrate clusters...** action on the Environments page bulk-migrates unlabeled or cross-env clusters with per-cluster progress, retry-on-failure, and relabel warnings when the source filter crosses envs.
- **Admin cluster list** at `/admin/clusters` adds grouping chips (Environment, Team, or both for nested team → env layout).

Webhook denials from env mutations render inline on whichever form triggered them — per-member cap breaches, team-admin-only edits, env-label immutability without the migration annotation — rather than as generic toasts, so operators see the rejection text next to the field that failed.

## Next Steps

- **[Concepts](../concepts/)** -- Understand Butler's architecture and multi-tenancy model.
- **[Operations](../operations/)** -- Day-2 operations: upgrade, monitor, backup, scale.
