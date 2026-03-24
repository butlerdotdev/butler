---
title: Team
sidebar_position: 4
---

A Team represents a multi-tenancy boundary in Butler, grouping users, clusters, and resource quotas under a single organizational unit.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Cluster

## Short Name

`tm`

## Description

Team is the primary multi-tenancy resource in Butler. Each Team owns a dedicated namespace (`team-{name}`) where its TenantClusters, ProviderConfigs, and other namespaced resources are created. Teams control access through user and group memberships with role-based permissions, enforce resource quotas across all clusters, and define default cluster configurations.

The Team controller reconciles the following:

- Creates and manages the team namespace
- Sets up RBAC (Roles and RoleBindings) for team members
- Tracks resource usage across all TenantClusters in the team
- Evaluates quota limits and sets conditions when limits are breached

## Spec

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `displayName` | string | No | — | Human-readable team name |
| `description` | string | No | — | Team description |
| `access` | TeamAccess | Yes | — | Team membership configuration |
| `resourceLimits` | TeamResourceLimits | No | — | Limits on team resource consumption |
| `providerConfigRef` | LocalObjectReference | No | — | Default provider for team clusters |
| `clusterDefaults` | ClusterDefaults | No | — | Default values for new TenantClusters |

### TeamAccess

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `users` | []TeamUser | No | Direct user memberships |
| `groups` | []TeamGroup | No | Group-based memberships (synced from IdP) |

### TeamUser

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | User email address |
| `role` | TeamRole | No | `viewer` | Role assigned to this user |

### TeamGroup

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Group name (from identity provider) |
| `role` | TeamRole | No | `viewer` | Role assigned to all group members |
| `identityProvider` | string | No | — | Name of the IdentityProvider CRD to match against. If omitted, matches groups from any configured provider. |

### Roles

| Role | Description | Permissions |
|------|-------------|-------------|
| `admin` | Team administrator | Full access to team resources. Can manage members, create/delete clusters, configure addons, and modify team settings. |
| `operator` | Cluster operator | Create, update, and delete clusters and addons. Cannot manage team membership. |
| `viewer` | Read-only member | Read-only access to team resources. Can view clusters, addons, and team configuration. |

### TeamResourceLimits

All fields are optional. When a limit is not set, no enforcement is applied for that dimension.

#### Cluster Limits

| Field | Type | Validation | Description |
|-------|------|------------|-------------|
| `maxClusters` | *int32 | min: 0 | Maximum number of TenantClusters the team can create |
| `maxNodesPerCluster` | *int32 | min: 0 | Maximum worker nodes per individual cluster |
| `maxTotalNodes` | *int32 | min: 0 | Maximum total worker nodes across all clusters |

#### Compute Limits

| Field | Type | Description |
|-------|------|-------------|
| `maxCPUCores` | *resource.Quantity | Maximum total CPU cores across all clusters |
| `maxMemory` | *resource.Quantity | Maximum total memory across all clusters |
| `maxStorage` | *resource.Quantity | Maximum total storage across all clusters |

#### Per-Cluster Defaults

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `defaultNodeCount` | *int32 | 3 | Default number of worker nodes when not specified |
| `defaultCPUPerNode` | *resource.Quantity | — | Default CPU per worker node |
| `defaultMemoryPerNode` | *resource.Quantity | — | Default memory per worker node |

#### Feature Restrictions

| Field | Type | Description |
|-------|------|-------------|
| `allowedKubernetesVersions` | []string | Kubernetes versions permitted for this team (e.g., `["1.29.x", "1.30.x"]`). Empty means all versions allowed. |
| `allowedProviders` | []string | Infrastructure providers this team can use (e.g., `["harvester", "nutanix"]`). Empty means all providers allowed. |
| `allowedAddons` | []string | Addons this team can install. Empty means all addons allowed. |
| `deniedAddons` | []string | Addons this team cannot install. Takes precedence over `allowedAddons`. |

### ClusterDefaults

Default values applied to new TenantClusters created in this team when the corresponding field is not explicitly set.

| Field | Type | Description |
|-------|------|-------------|
| `kubernetesVersion` | string | Default Kubernetes version |
| `workerCount` | int32 | Default number of worker nodes |
| `workerCPU` | resource.Quantity | Default CPU per worker |
| `workerMemoryGi` | int32 | Default memory per worker (GiB) |
| `workerDiskGi` | int32 | Default disk size per worker (GiB) |
| `defaultAddons` | []string | Addons installed by default on new clusters |

### ProviderConfigRef

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the ProviderConfig in the team namespace |

## Status

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard Kubernetes conditions (see below) |
| `phase` | string | Current team phase |
| `namespace` | string | Team's namespace (`team-{name}`) |
| `observedGeneration` | int64 | Last generation reconciled by the controller |
| `clusterCount` | int32 | Number of TenantClusters in the team |
| `memberCount` | int32 | Total number of team members (users + resolved group members) |
| `resourceUsage` | TeamResourceUsage | Current resource consumption |
| `quotaStatus` | string | Overall quota status |
| `quotaMessage` | string | Human-readable quota status message |

### Conditions

| Type | Description |
|------|-------------|
| `NamespaceReady` | The team namespace has been created and is available |
| `RBACReady` | Roles and RoleBindings have been created for all team members |
| `Ready` | The team is fully reconciled and operational |
| `QuotaExceeded` | One or more resource limits have been breached |

### Phases

| Phase | Description |
|-------|-------------|
| `Pending` | Team created, waiting for namespace and RBAC setup |
| `Ready` | Team is fully provisioned and operational |
| `Terminating` | Team is being deleted, cleanup in progress |
| `Failed` | An error occurred during reconciliation |

### TeamResourceUsage

| Field | Type | Description |
|-------|------|-------------|
| `clusters` | int32 | Number of active TenantClusters |
| `totalNodes` | int32 | Total worker nodes across all clusters |
| `totalCPU` | *resource.Quantity | Total CPU allocated across all clusters |
| `totalMemory` | *resource.Quantity | Total memory allocated across all clusters |
| `totalStorage` | *resource.Quantity | Total storage allocated across all clusters |
| `clusterUtilization` | int32 | Percentage of maxClusters used (0-100) |
| `nodeUtilization` | int32 | Percentage of maxTotalNodes used (0-100) |
| `cpuUtilization` | int32 | Percentage of maxCPUCores used (0-100) |
| `memoryUtilization` | int32 | Percentage of maxMemory used (0-100) |

### Quota Status Values

| Value | Description |
|-------|-------------|
| `OK` | All resource usage is within limits |
| `Warning` | Resource usage is approaching a limit (>80%) |
| `Exceeded` | One or more limits have been breached |

### Print Columns

| Column | Field | Description |
|--------|-------|-------------|
| Display Name | `spec.displayName` | Human-readable name |
| Phase | `status.phase` | Current lifecycle phase |
| Namespace | `status.namespace` | Team namespace |
| Clusters | `status.clusterCount` | Number of clusters |
| Quota | `status.quotaStatus` | Quota status (OK/Warning/Exceeded) |
| Age | `metadata.creationTimestamp` | Resource age |

## Resource Usage Calculation

The Team controller computes resource usage by aggregating data from all TenantClusters in the team namespace:

- **CPU**: Sum of `machineTemplate.CPU * workers.replicas` across all TenantClusters
- **Memory**: Sum of `machineTemplate.Memory * workers.replicas` across all TenantClusters
- **Storage**: Sum of `machineTemplate.DiskSize * workers.replicas` across all TenantClusters
- **Utilization percentages**: Computed as `(current usage / limit) * 100`, only when the corresponding limit is set

When any utilization exceeds 100%, the controller sets the `QuotaExceeded` condition to `True` and updates `quotaStatus` to `Exceeded`.

## Quota Enforcement

Resource limits are enforced at two levels:

### Webhook Enforcement (TenantCluster Admission)

A validating webhook intercepts TenantCluster create and update requests and checks them against the owning Team's resource limits:

- **CPU quota**: Sums `machineTemplate.CPU * replicas` across all existing TenantClusters in the team (excluding self on update), adds the proposed cluster's CPU, and compares against `maxCPUCores`.
- **Memory quota**: Same calculation using `machineTemplate.Memory` against `maxMemory`.
- **Storage quota**: Same calculation using `machineTemplate.DiskSize` against `maxStorage`.
- **Cluster count**: Checks current cluster count against `maxClusters`.
- **Nodes per cluster**: Checks proposed worker replicas against `maxNodesPerCluster`.
- **Total nodes**: Sums worker replicas across all clusters against `maxTotalNodes`.

The webhook also enforces ProviderConfig-level limits (`maxClustersPerTeam`, `maxNodesPerTeam`) when present.

If any limit would be exceeded, the webhook rejects the request with a descriptive error message.

### Controller Enforcement (Continuous Monitoring)

The Team controller continuously monitors resource usage and updates status conditions. This catches drift caused by external changes and provides visibility into quota health via the `QuotaExceeded` condition and `quotaStatus` field.

## Group Sync

Teams can reference identity provider groups to automatically grant access to group members. When a user authenticates via SSO, Butler resolves their group memberships and matches them against team group entries.

### How Group Matching Works

1. The user authenticates via an IdentityProvider (OIDC/SSO).
2. Butler retrieves the user's group memberships from the identity provider:
   - **Google Workspace**: Groups fetched via Admin SDK (not included in OIDC token).
   - **Microsoft Entra ID / Okta**: Groups come from the `groups` claim in the OIDC token.
3. Group names are normalized before matching:
   - Email-style groups: `engineers@company.com` is normalized to `engineers`.
   - LDAP DN format: `CN=Engineers,OU=Groups,DC=company` is normalized to `engineers`.
4. The normalized group name is compared against `spec.access.groups[].name`.
5. If the group entry specifies `identityProvider`, it only matches groups from that specific IdentityProvider CRD. Otherwise, it matches groups from any configured provider.
6. Matched users receive the role specified on the group entry.

### Group Priority

If a user is matched by both a direct `users` entry and one or more `groups` entries, the highest-privilege role wins. Role precedence: `admin` > `operator` > `viewer`.

## Finalizer

`butler.butlerlabs.dev/team`

The Team finalizer ensures proper cleanup when a Team is deleted:

1. All TenantClusters in the team namespace are deleted (and their finalizers run).
2. RBAC resources (Roles, RoleBindings) are removed.
3. The team namespace (`team-{name}`) is deleted.
4. The finalizer is removed and the Team resource is deleted.

## Examples

### Basic Team

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
```

### Team with Full Resource Quotas

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: development
spec:
  displayName: Development Team
  description: Shared development environment with resource limits
  access:
    users:
      - name: lead@example.com
        role: admin
    groups:
      - name: developers
        role: operator
        identityProvider: google-workspace
  resourceLimits:
    # Cluster limits
    maxClusters: 5
    maxNodesPerCluster: 10
    maxTotalNodes: 30
    # Compute limits
    maxCPUCores: "120"
    maxMemory: "480Gi"
    maxStorage: "2Ti"
    # Per-cluster defaults
    defaultNodeCount: 3
    defaultCPUPerNode: "4"
    defaultMemoryPerNode: "16Gi"
  providerConfigRef:
    name: harvester-dev
  clusterDefaults:
    kubernetesVersion: "1.30.4"
    workerCount: 3
    workerCPU: "4"
    workerMemoryGi: 16
    workerDiskGi: 100
    defaultAddons:
      - cilium
      - metallb
      - cert-manager
```

### Team with Feature Restrictions

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: sandbox
spec:
  displayName: Sandbox Team
  description: Restricted sandbox for experimentation
  access:
    users:
      - name: intern@example.com
        role: operator
    groups:
      - name: interns
        role: viewer
  resourceLimits:
    maxClusters: 2
    maxNodesPerCluster: 3
    maxTotalNodes: 6
    maxCPUCores: "24"
    maxMemory: "96Gi"
    maxStorage: "500Gi"
    # Feature restrictions
    allowedKubernetesVersions:
      - "1.30.4"
    allowedProviders:
      - harvester
    allowedAddons:
      - cilium
      - metallb
    deniedAddons:
      - longhorn
      - gpu-operator
  providerConfigRef:
    name: harvester-sandbox
```

## See Also

- [Multi-Tenancy Architecture](../../architecture/multi-tenancy.md)
- [TenantCluster](./tenantcluster.md) - Clusters owned by teams
- [ProviderConfig](./providerconfig.md) - Infrastructure provider configuration
- [Multi-Tenancy](../../concepts/multi-tenancy.md) -- SSO and group sync
