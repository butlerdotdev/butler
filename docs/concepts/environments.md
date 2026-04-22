---
title: Environments
sidebar_position: 7
---

# Environments

Environments are logical subdivisions within a [Team](./multi-tenancy.md). Use them to separate dev from prod, carve out per-user sandboxes, or group shared utility clusters without creating a new Team for each slice.

An environment lives in `Team.spec.environments[]` as a named slot with optional quotas, access overrides, and default cluster shape. TenantClusters opt into an environment by carrying the label `butler.butlerlabs.dev/environment: <env-name>`. The label is stamped automatically by the console and CLI when an environment is selected at cluster-create time.

Environments are not a separate tenancy boundary. Team roles still apply; environment access can only elevate a member's role within that env, never reduce it. Team admins remain admins everywhere in the team.

## When to use environments

Reach for environments when:

- The team runs workloads at different risk tiers (dev, staging, prod) and you want per-tier quotas.
- Individual engineers need sandbox clusters capped at one per person.
- You want different default cluster shapes per tier (small dev clusters, larger prod clusters).
- Only a subset of team members should be able to create production clusters.

If the team operates a single homogeneous workload with no tier distinctions, leaving `spec.environments` empty is fine. The label requirement only kicks in once at least one env is defined.

## The four fields

Each environment in `spec.environments[]` has four configurable fields. Only `name` is required.

### name

Operator-chosen identifier. Used as the value of the `butler.butlerlabs.dev/environment` label on TenantClusters. Must match Kubernetes label-value syntax (alphanumeric plus `-`, `_`, `.`; alphanumeric anchors; up to 63 characters). Immutable after creation; rename by delete-and-recreate.

### description

Free-text blurb shown in the console env list and the environment context in the cluster create form. Not interpreted by the controller or webhook; purely operator documentation.

### limits

Per-env quota caps on top of the team's total ceiling:

- `maxClusters`: maximum TenantClusters allowed in this environment.
- `maxClustersPerMember`: per-individual cap. Prevents any single user from consuming an environment's cluster count on their own.

Both are optional. An unset cap means no environment-level limit; the team-wide `resourceLimits.maxClusters` still applies.

### access

Additive-only RBAC overrides. Lists users or groups with an env-specific role that overrides their team role **within this environment only**. The role can only be equal to or higher than the team role; env access cannot reduce a team admin to operator.

### clusterDefaults

Default values applied when a TenantCluster is created in this env without specifying the field itself. Env defaults override team-level `spec.clusterDefaults` on conflicts. Supported fields match the team cluster defaults (`kubernetesVersion`, `workerCount`, `workerCPU`, `workerMemoryGi`, `workerDiskGi`).

## Per-member cap semantics

`maxClustersPerMember` is identity-based, not role-based. The admission webhook counts TenantClusters in the target env owned by the requesting user (via the `butler.butlerlabs.dev/owner` annotation the controller promotes from the `butler.butlerlabs.dev/creator-email` annotation at create time). A user over their personal cap gets rejected regardless of whether they are team admin, operator, or viewer.

"Member" here means any authenticated identity that creates clusters. Platform admins are subject to the cap too if they create on behalf of themselves; the cap does not exempt any role.

The cap only enforces when the creator email is known. Console and CLI creates stamp the annotation automatically (the console via server impersonation, the CLI via direct identity forwarding). TenantClusters created by raw `kubectl apply` must carry the annotation explicitly or the admission webhook rejects them when the target env has a per-member cap set.

The rejection message on a breach reads:

```
user "carol@example.com" already owns 2 cluster(s) in environment "dev"; env limits to 2 per member
```

## Additive-only access inheritance

Team roles provide a baseline. An env's access block can elevate a specific user or group for that env only:

```
Team access:
  alice@example.com: operator
  bob@example.com:   admin

Environment prod.access:
  alice@example.com: admin
```

Effective roles for alice:

| Environment | Role |
|---|---|
| dev (no env override) | operator |
| prod (env elevates her) | admin |

Bob stays admin everywhere because team roles apply wherever env access does not elevate further. Reducing Bob to operator in an env has no effect; the session layer takes the higher of the two roles.

Env access entries must reference users or groups who already appear in `Team.spec.access`. Introducing a new identity through an env access block is rejected by the admission webhook; use the team access list to grant initial team membership, then elevate through an env if needed.

## Cluster defaults hierarchy

When a TenantCluster is created, each field resolves through three layers:

1. Explicit value in the TenantCluster spec, if set.
2. Environment's `clusterDefaults[field]`, if the TC is in an env and the env sets it.
3. Team's `spec.clusterDefaults[field]`, if the team sets it.

The console cluster-create form reflects this layering: when you select an environment, fields the env overrides flip to the env value with a "from env default" hint; fields only the team sets show a "from team default" hint. Typing over a field opts out of the hierarchy for that field only.

## Mutation authority

Environments have a split edit model:

| Field | Who can change it |
|---|---|
| `Team.spec.resourceLimits` | Platform admins only |
| `Team.spec.environments[].limits` | Team admins of the team, and platform admins |
| All other env fields | Team admins of the team, and platform admins |

The admission webhook enforces this. Team admins cannot raise their own team ceiling (only a platform admin can edit `resourceLimits`), but they can adjust per-env caps within that ceiling on their own.

When a team admin or platform admin attempts a resourceLimits mutation they are not authorized for, the webhook replies verbatim with:

```
spec.resourceLimits may only be modified by platform admins; user "alice@example.com" is not a platform admin
```

An operator on a team who attempts to edit env limits receives:

```
spec.environments[].limits may only be modified by team admins of "payments" or platform admins; user "bob@example.com" is neither
```

butler-server surfaces the message verbatim as the `message` field of a structured 403 (`reason: "webhook-denied"`); the console renders it inline on the form that triggered the denial.

## Migration for existing clusters

Clusters that existed before the team had any environments defined remain without the env label. They continue to work and count against the team's total `maxClusters`, but are not accounted under any env's cap.

Moving an existing cluster into an environment uses the `butler.butlerlabs.dev/migration-operation` annotation. The console's **Change Environment** action (per-cluster) and **Migrate clusters...** action (bulk) both set this annotation automatically. The CLI's `butleradm env migrate` command does the same. Direct `kubectl` edits to the env label are rejected without the annotation; the annotation is what tells the webhook this is an intentional migration rather than a stray label change.

The webhook message on a missing annotation is:

```
env label changes require the "butler.butlerlabs.dev/migration-operation" annotation set to "true"; use `butleradm env migrate`
```

## Worked example

A team `payments` runs its workloads across two environments:

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: payments
spec:
  access:
    users:
      - name: alice@example.com
        role: admin
      - name: bob@example.com
        role: operator
      - name: carol@example.com
        role: operator
  resourceLimits:
    maxClusters: 20
  clusterDefaults:
    kubernetesVersion: v1.31.0
    workerCount: 2
    workerCPU: 2
    workerMemoryGi: 4
  environments:
    - name: dev
      description: Developer sandboxes and integration testing
      limits:
        maxClustersPerMember: 2
      clusterDefaults:
        workerMemoryGi: 2
    - name: prod
      description: Customer-facing payments processing
      limits:
        maxClusters: 6
        maxClustersPerMember: 1
      access:
        users:
          - name: bob@example.com
            role: admin
      clusterDefaults:
        workerCount: 3
        workerCPU: 4
        workerMemoryGi: 8
```

### Effective access

| User | dev role | prod role | Why |
|---|---|---|---|
| alice | admin | admin | Team admin; env access block not needed |
| bob | operator | admin | Env prod elevates him |
| carol | operator | operator | No env override |

### Effective cluster defaults in dev

| Field | Value | Source |
|---|---|---|
| kubernetesVersion | v1.31.0 | team |
| workerCount | 2 | team |
| workerCPU | 2 | team |
| workerMemoryGi | 2 | env (env overrides team) |

### Effective cluster defaults in prod

| Field | Value | Source |
|---|---|---|
| kubernetesVersion | v1.31.0 | team |
| workerCount | 3 | env |
| workerCPU | 4 | env |
| workerMemoryGi | 8 | env |

### Quota math

- Team ceiling: 20 clusters.
- Env prod cap: 6; env dev cap: unset (only a per-member cap).
- Per-member cap: 2 in dev, 1 in prod.

Scenarios:

- Carol has 2 clusters in dev, wants a 3rd in dev: rejected, hits her per-member dev cap.
- Carol has 2 clusters in dev, wants 1 in prod: allowed, different env scope.
- Bob has 1 cluster in prod, wants a 2nd in prod: rejected, hits his per-member prod cap.
- Team has 6 prod clusters across all members: next prod create rejected for everyone until one is deleted, regardless of per-member cap.
- Team has 20 clusters total across dev and prod: next create in any env rejected, team ceiling reached.

## See also

- [Team CRD Reference](../reference/crds/team.md): full schema including `EnvironmentSpec` and `EnvironmentLimits`. Env access reuses the team-level `TeamAccess` type.
- [butleradm env](../reference/cli/butleradm.md#butleradm-env): CLI for env lifecycle.
- [butlerctl cluster --environment](../reference/cli/butlerctl.md#butlerctl-cluster-create): create a cluster in a specific env.
- [Multi-Tenancy](./multi-tenancy.md): the team-level tenancy boundary envs subdivide.
