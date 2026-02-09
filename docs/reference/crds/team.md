---
title: Team
sidebar_position: 4
---

A Team represents a multi-tenancy boundary in Butler, grouping users and clusters.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Cluster

## Short Name

`tm`

## Description

Team is the primary multi-tenancy resource in Butler. Each team owns a namespace where their TenantClusters and other resources are created. Teams have members with different roles (admin, operator, viewer) and can be synced with identity provider groups.

## Spec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `displayName` | string | No | Human-readable team name |
| `description` | string | No | Team description |
| `access` | TeamAccess | Yes | Team membership configuration |
| `providerConfigRef` | ObjectReference | No | Default provider for team clusters |
| `resourceQuota` | ResourceQuota | No | Limits on team resources |

### TeamAccess

| Field | Type | Description |
|-------|------|-------------|
| `users` | []TeamUser | Direct user memberships |
| `groups` | []TeamGroup | Group-based memberships (synced from IdP) |

### TeamUser

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | User email address |
| `role` | string | Role: `admin`, `operator`, or `viewer` |

### TeamGroup

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Group name (from identity provider) |
| `role` | string | Role assigned to group members |

### Roles

| Role | Permissions |
|------|-------------|
| `admin` | Full access to team resources, can manage members |
| `operator` | Create, update, delete clusters and addons |
| `viewer` | Read-only access to team resources |

## Example

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: platform-team
spec:
  displayName: Platform Engineering
  description: Core platform infrastructure team
  access:
    users:
      - name: alice@example.com
        role: admin
      - name: bob@example.com
        role: operator
    groups:
      - name: platform-engineers
        role: operator
      - name: platform-viewers
        role: viewer
  providerConfigRef:
    name: harvester-prod
    namespace: butler-system
  resourceQuota:
    maxClusters: 10
    maxNodesPerCluster: 20
```

## Status

| Field | Type | Description |
|-------|------|-------------|
| `namespace` | string | Team's namespace (team-{name}) |
| `ready` | bool | Whether team is fully provisioned |
| `clusterCount` | int32 | Number of clusters in team |
| `memberCount` | int32 | Number of team members |

## See Also

- [Multi-Tenancy Architecture](../../architecture/multi-tenancy.md)
- [TenantCluster](./tenantcluster.md) - Clusters owned by teams
