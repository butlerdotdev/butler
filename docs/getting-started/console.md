---
title: Butler Console
sidebar_position: 8
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

Butler creates an admin user during bootstrap using the email and password from your bootstrap config (`auth.adminEmail` and `auth.adminPassword`). Log in with those credentials.

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

## Next Steps

- **[Concepts](../concepts/)** -- Understand Butler's architecture and multi-tenancy model.
- **[Operations](../operations/)** -- Day-2 operations: upgrade, monitor, backup, scale.
