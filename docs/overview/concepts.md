# Butler Concepts & Terminology

This document defines the key terms and concepts used throughout Butler.

## Core Concepts

### Management Cluster

The Kubernetes cluster that runs Butler platform components. This is the "control plane" for your entire Butler deployment.

**Contains:**
- Butler controllers (butler-controller, butler-bootstrap)
- Steward (TenantControlPlane operator)
- Cluster API controllers and providers
- Butler Console (optional)
- Observability stack (optional)

**Key Points:**
- There is typically one management cluster per environment (dev, prod)
- Uses Talos Linux for immutable, secure node OS
- Can run in HA (3 control plane nodes) or single-node topology

### Tenant Cluster

A Kubernetes cluster provisioned by Butler for running workloads. Each tenant cluster is represented by a `TenantCluster` Custom Resource.

**Characteristics:**
- Fully isolated Kubernetes cluster
- Can have hosted or standalone control plane
- Worker nodes run on infrastructure provider (Harvester, Nutanix, etc.)
- Managed addons (CNI, storage, etc.)

### Hosted Control Plane

A control plane that runs as pods in the management cluster rather than as dedicated VMs. Butler uses [Steward](https://github.com/butlerdotdev/steward) for hosted control planes.

**Benefits:**
- Faster provisioning (seconds vs. minutes)
- Lower resource consumption
- Simplified operations
- Centralized etcd management

**Trade-offs:**
- Management cluster becomes more critical
- Requires sufficient management cluster resources

### Standalone Control Plane

Traditional control plane with dedicated VMs for control plane nodes. Each tenant cluster has its own etcd cluster.

**Benefits:**
- Maximum isolation
- No dependency on management cluster for workloads
- Traditional Kubernetes topology

**Trade-offs:**
- Higher resource consumption
- Slower provisioning
- More operational overhead

---

## Infrastructure Concepts

### Provider

An infrastructure adapter that provisions VMs for Butler clusters. Providers implement the interface between Butler and specific infrastructure platforms.

**Current Providers:**
| Provider | Platform | CAPI Provider |
|----------|----------|---------------|
| butler-provider-harvester | Harvester HCI | KubeVirt (capk) |
| butler-provider-nutanix | Nutanix AHV | CAPX |

### ProviderConfig

A CRD that stores infrastructure credentials and configuration for a specific provider.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: harvester-prod
spec:
  type: harvester
  harvester:
    credentialsRef:
      name: harvester-kubeconfig
    namespace: default
    networkName: default/workloads
    imageName: default/talos-1.9
```

### MachineRequest

A CRD used during bootstrap to request VM provisioning from a provider. Abstracted away during normal operation; users interact with TenantCluster instead.

---

## Platform Concepts

### Bootstrap

The process of creating a management cluster from scratch. Butler uses a temporary KIND cluster to orchestrate the bootstrap process.

**Phases:**
1. Create KIND cluster
2. Deploy bootstrap and provider controllers
3. Provision management cluster VMs
4. Configure Talos Linux
5. Install platform components
6. Clean up KIND cluster

### ClusterBootstrap

A CRD that defines a management cluster to be bootstrapped.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: my-cluster
spec:
  provider: harvester
  cluster:
    name: butler-mgmt
    controlPlaneEndpoint: 10.40.0.200
    kubernetesVersion: "v1.31.0"
    topology: ha  # or single-node
```

### ButlerConfig

A cluster-scoped singleton CRD that stores platform-wide configuration.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ButlerConfig
metadata:
  name: butler
spec:
  clusterDefaults:
    kubernetesVersion: "v1.30.0"
    controlPlane:
      type: hosted
  multiTenancy:
    mode: Enforced  # enforced, optional, or disabled
```

---

## Multi-Tenancy Concepts

### Team

A CRD representing a group of users who share access to Butler resources. Teams provide isolation boundaries.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: Team
metadata:
  name: platform-team
spec:
  displayName: "Platform Engineering"
  access:
    users:
      - name: alice@example.com
        role: admin
      - name: bob@example.com
        role: operator
    groups:
      - name: platform-engineers
        role: admin
```

### Team Roles

| Role | Permissions |
|------|-------------|
| admin | Full access to team resources, manage members |
| operator | Create/modify/delete clusters and addons |
| viewer | Read-only access to team resources |

### Multi-Tenancy Modes

| Mode | Description |
|------|-------------|
| enforced | All resources must belong to a team |
| optional | Teams available but not required |
| disabled | No team isolation |

### Platform Admin

A user with administrative access to the entire Butler platform, regardless of team membership. Platform admins can:

- Access all teams and clusters
- Manage platform configuration
- View management cluster resources

---

## Addon Concepts

### AddonDefinition

A cluster-scoped CRD that defines an addon available for installation. Represents the "catalog entry" for an addon.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: AddonDefinition
metadata:
  name: cilium
spec:
  displayName: Cilium
  description: eBPF-based CNI
  category: networking
  platform: true
  chart:
    repository: https://helm.cilium.io
    name: cilium
    defaultVersion: "1.17.0"
```

### TenantAddon

A CRD that represents an addon installed on a specific tenant cluster.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: TenantAddon
metadata:
  name: my-cluster-monitoring
  namespace: team-a
spec:
  clusterRef:
    name: my-cluster
  addon: prometheus
  version: "25.0.0"
```

### ManagementAddon

A CRD that represents an addon installed on the management cluster.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ManagementAddon
metadata:
  name: victoria-metrics
spec:
  addon: victoria-metrics
  version: "0.25.0"
```

### Platform Addons

Core addons required for cluster operation. Installed automatically during cluster creation and cannot be removed via UI.

Examples: Cilium (CNI), MetalLB (LoadBalancer), Longhorn (Storage)

---

## User Concepts

### User

A CRD representing a user account in Butler.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: User
metadata:
  name: user-alice
spec:
  email: alice@example.com
  displayName: "Alice Smith"
  authType: sso  # or local
  ssoProvider: google
  disabled: false
```

### IdentityProvider

A CRD that configures an SSO/OIDC provider for authentication.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: IdentityProvider
metadata:
  name: google
spec:
  type: oidc
  displayName: "Google"
  oidc:
    issuer: https://accounts.google.com
    clientID: xxx
    clientSecretRef:
      name: google-oidc-secret
```

---

## Lifecycle Concepts

### Cluster Phases

| Phase | Description |
|-------|-------------|
| Pending | CR created, waiting for reconciliation |
| Creating | Resources being created |
| Provisioning | Control plane being set up |
| Provisioning | Worker nodes being provisioned |
| Installing | Platform addons being installed |
| Running | Cluster operational |
| Updating | Cluster being updated (scale, version, etc.) |
| Deleting | Cluster being deleted |
| Failed | Error state |

### Addon Phases

| Phase | Description |
|-------|-------------|
| Pending | Addon requested |
| Installing | Helm release being installed |
| Installed | Addon running |
| Upgrading | New version being deployed |
| Uninstalling | Addon being removed |
| Failed | Installation error |

---

## See Also

- [Architecture](../architecture/) - How Butler components work together
- [Getting Started](../getting-started/) - Hands-on introduction
