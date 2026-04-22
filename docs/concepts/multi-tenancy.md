---
title: Multi-Tenancy
sidebar_position: 6
---

# Multi-Tenancy

Butler provides multi-tenancy through Teams. Each Team owns a Kubernetes namespace and controls who can create and access clusters within it.

## Teams

A Team is a cluster-scoped CRD that groups users and resources. When you create a Team, Butler creates a namespace with the team's name and sets up RBAC bindings for the team's members.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: backend-team
spec:
  displayName: "Backend Engineering"
  access:
    users:
      - name: alice@example.com
        role: admin
      - name: bob@example.com
        role: operator
    groups:
      - name: backend-engineers
        role: operator
```

All TenantClusters, TenantAddons, and Secrets (kubeconfigs) for this team live in the `backend-team` namespace.

## Roles

| Role | Create/delete clusters | Scale clusters | Manage addons | Manage team members | View clusters |
|------|----------------------|----------------|---------------|--------------------|----|
| admin | Yes | Yes | Yes | Yes | Yes |
| operator | Yes | Yes | Yes | No | Yes |
| viewer | No | No | No | No | Yes |

All roles can retrieve kubeconfigs for clusters in their team.

## Multi-Tenancy Modes

Configure the multi-tenancy mode in [ButlerConfig](../reference/crds/butlerconfig.md):

| Mode | Behavior |
|------|----------|
| Enforced | Every TenantCluster must belong to a team namespace. Users can only access their team's resources. |
| Optional | Teams are available but not required. Resources without a team are accessible to all authenticated users. |
| Disabled | No team isolation. All authenticated users see all resources. |

## Platform Administrators

Platform admins have full access to all teams and clusters regardless of team membership. They can create teams, manage platform configuration (ButlerConfig), and access the management cluster directly.

## Resource Quotas

Teams support resource limits to prevent any single team from consuming the entire platform:

- Maximum clusters per team
- Maximum workers per cluster
- Maximum total CPU, memory, and storage across all clusters

Butler tracks current usage against these limits in the Team's status.

## Environments

Teams can optionally subdivide into named [environments](./environments.md) (dev, prod, per-user sandboxes). Each environment carries its own quotas, access overrides, and default cluster shape, sharing the team's overall ceiling.

## See Also

- [Environments](./environments.md) -- Subdivisions within a team with per-env quotas and access
- [Architecture > Multi-Tenancy Implementation](../architecture/multi-tenancy.md) -- Kubernetes RBAC mapping and namespace creation flow
- [Team CRD Reference](../reference/crds/team.md) -- Full Team specification
- [ButlerConfig Reference](../reference/crds/butlerconfig.md) -- Multi-tenancy mode configuration
