---
title: ProviderConfig
sidebar_position: 3
---

# ProviderConfig

A ProviderConfig defines connection credentials, infrastructure settings, network configuration, and resource limits for an infrastructure provider.

## API Version

`butler.butlerlabs.dev/v1alpha1`

## Scope

Namespaced

## Short Name

`pc`

## Finalizer

`butler.butlerlabs.dev/providerconfig`

## Print Columns

| Column | Field |
|--------|-------|
| Provider | `spec.provider` |
| Scope | `spec.scope.type` |
| Ready | `status.ready` |
| Validated | `status.validated` |
| Age | `metadata.creationTimestamp` |

## Description

ProviderConfig stores the credentials, provider-specific settings, network configuration, and resource limits needed to provision tenant cluster infrastructure. Butler supports six provider types: three on-premises providers (Harvester, Nutanix, Proxmox) and three cloud providers (Azure, AWS, GCP). Each provider type has its own configuration section with provider-specific fields.

A ProviderConfig can be scoped to the entire platform or to a specific team, controlling which teams can use it for cluster provisioning. Network configuration supports both cloud-native networking and IPAM-managed address pools with static or elastic allocation modes.

## Spec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider` | `ProviderType` | Yes | Provider type. One of: `harvester`, `nutanix`, `proxmox`, `azure`, `aws`, `gcp` |
| `credentialsRef` | [`SecretReference`](#secretreference) | Yes | Reference to a Secret containing provider credentials |
| `harvester` | [`HarvesterConfig`](#harvesterconfig) | No | Harvester-specific configuration |
| `nutanix` | [`NutanixConfig`](#nutanixconfig) | No | Nutanix-specific configuration |
| `proxmox` | [`ProxmoxConfig`](#proxmoxconfig) | No | Proxmox-specific configuration |
| `azure` | [`AzureConfig`](#azureconfig) | No | Azure-specific configuration |
| `aws` | [`AWSConfig`](#awsconfig) | No | AWS-specific configuration |
| `gcp` | [`GCPConfig`](#gcpconfig) | No | GCP-specific configuration |
| `scope` | [`Scope`](#scope) | No | Controls which teams can use this provider |
| `network` | [`NetworkConfig`](#network-configuration) | No | Network and IPAM configuration |
| `limits` | [`Limits`](#limits) | No | Resource limits for teams using this provider |

### SecretReference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the Secret |
| `namespace` | string | No | Namespace of the Secret. Defaults to the ProviderConfig namespace |
| `key` | string | No | Specific key within the Secret data |

---

## Provider Configuration

Each provider type requires its own configuration section matching the `spec.provider` value. Only the section corresponding to the chosen provider type is used; the others are ignored.

### HarvesterConfig

Configuration for [Harvester](https://harvesterhci.io/) HCI clusters.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | No | | Harvester API endpoint URL |
| `namespace` | string | No | `"default"` | Harvester namespace for VM resources |
| `networkName` | string | Yes | | VM network in `namespace/name` format (e.g., `default/vlan40-workloads`) |
| `imageName` | string | No | | VM image name |
| `storageClassName` | string | No | | StorageClass to use for VM volumes |

### NutanixConfig

Configuration for [Nutanix](https://www.nutanix.com/) AHV clusters via Prism Central.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | | Prism Central endpoint URL. Must use `https://` scheme |
| `port` | int32 | No | `9440` | Prism Central API port |
| `insecure` | bool | No | `false` | Skip TLS certificate verification |
| `clusterUUID` | string | Yes | | Target AHV cluster UUID |
| `subnetUUID` | string | Yes | | Network subnet UUID for VM NICs |
| `imageUUID` | string | No | | VM image UUID |
| `storageContainerUUID` | string | No | | Storage container UUID for VM disks |

### ProxmoxConfig

Configuration for [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment) clusters.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | | Proxmox API endpoint URL. Must use `https://` scheme |
| `insecure` | bool | No | `false` | Skip TLS certificate verification |
| `nodes` | []string | Yes | | Target Proxmox node names. Minimum 1 entry |
| `storage` | string | Yes | | Storage identifier for VM disks |
| `templateID` | int32 | No | | VM template ID to clone from |
| `vmidRange` | [`VMIDRange`](#vmidrange) | No | | Range of VM IDs to allocate |

#### VMIDRange

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `start` | int32 | No | | Start of VM ID range. Minimum value: `100` |
| `end` | int32 | No | | End of VM ID range. Minimum value: `100` |

### AzureConfig

Configuration for [Microsoft Azure](https://azure.microsoft.com/).

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `subscriptionID` | string | Yes | | Azure subscription ID |
| `resourceGroup` | string | Yes | | Azure resource group name |
| `location` | string | No | | Azure region (e.g., `eastus`, `westeurope`) |
| `vnetName` | string | No | | Virtual network name |
| `subnetName` | string | No | | Subnet name within the VNet |

### AWSConfig

Configuration for [Amazon Web Services](https://aws.amazon.com/).

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `region` | string | Yes | | AWS region (e.g., `us-east-1`, `eu-west-1`) |
| `vpcID` | string | No | | VPC ID. If omitted, the default VPC is used |
| `subnetIDs` | []string | No | | Subnet IDs for VM placement |
| `securityGroupIDs` | []string | No | | Security group IDs to attach to VMs |

### GCPConfig

Configuration for [Google Cloud Platform](https://cloud.google.com/).

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `projectID` | string | Yes | | GCP project ID |
| `region` | string | Yes | | GCP region (e.g., `us-central1`, `europe-west1`) |
| `network` | string | No | | VPC network name |
| `subnetwork` | string | No | | Subnetwork name within the VPC |

---

## Scope

The `spec.scope` field controls which teams can use the ProviderConfig for cluster provisioning.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | No | `"platform"` | Scope type. One of: `platform`, `team` |
| `teamRef` | LocalObjectReference | Conditional | | Reference to the Team. Required when `type` is `team` |

### Platform Scope

When `scope.type` is `platform` (or `scope` is omitted entirely), any team in the Butler platform can reference this ProviderConfig for cluster provisioning. Platform-scoped ProviderConfigs are typically created by platform administrators.

### Team Scope

When `scope.type` is `team`, only the team referenced by `scope.teamRef.name` can use this ProviderConfig. Team-scoped ProviderConfigs allow teams to bring their own infrastructure credentials and isolate their provider access from other teams.

---

## Network Configuration

The `spec.network` section controls IP address management and load balancer allocation for tenant clusters provisioned with this provider.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `mode` | string | No | `"cloud"` | Network mode. One of: `ipam`, `cloud` |
| `poolRefs` | [`[]PoolRef`](#poolref) | No | | References to IP address pools. Required when `mode` is `ipam` |
| `subnet` | string | No | | Subnet CIDR for node addressing |
| `gateway` | string | No | | Default gateway IP address |
| `dnsServers` | []string | No | | DNS server addresses |
| `loadBalancer` | [`LoadBalancerConfig`](#loadbalancerconfig) | No | | Load balancer IP allocation settings |
| `quotaPerTenant` | [`QuotaPerTenant`](#quotapertenant) | No | | Per-tenant IP address quotas |

### Network Modes

**`cloud` mode** (default): Relies on the cloud provider's native networking. IP addresses are assigned by the cloud provider's DHCP or instance metadata service. This is the default mode and is appropriate for AWS, Azure, and GCP providers.

**`ipam` mode**: Butler manages IP address allocation from configured address pools. Required for on-premises providers (Harvester, Nutanix, Proxmox) where no cloud DHCP service is available. When `ipam` mode is selected, `poolRefs` must reference at least one IP address pool.

:::warning
IPAM mode (`ipam`) is required for on-premises providers and is forbidden for cloud providers (`azure`, `aws`, `gcp`). The webhook validates this constraint on create and update.
:::

### PoolRef

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | | Name of the IP address pool resource |
| `priority` | int32 | No | `0` | Pool selection priority. Higher values indicate higher priority |

### LoadBalancerConfig

Controls how load balancer IP addresses are allocated to tenant clusters.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `defaultPoolSize` | *int32 | No | `8` | Default number of LB IPs allocated per tenant. Minimum: `1` |
| `allocationMode` | string | No | `"static"` | Allocation strategy. One of: `static`, `elastic` |
| `initialPoolSize` | *int32 | No | `2` | Initial pool size when using `elastic` mode. Minimum: `1` |
| `growthIncrement` | *int32 | No | `2` | Number of IPs to add when the elastic pool grows. Minimum: `1` |

#### Static Allocation

In `static` mode, each tenant cluster receives a fixed block of `defaultPoolSize` load balancer IPs at provisioning time. The block size does not change during the cluster lifecycle.

#### Elastic Allocation

In `elastic` mode, each tenant cluster starts with `initialPoolSize` load balancer IPs. As demand increases, Butler allocates additional IPs in increments of `growthIncrement`, up to `defaultPoolSize`. This conserves IP address space when tenants have variable load balancer requirements.

### QuotaPerTenant

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `maxNodeIPs` | *int32 | No | | Maximum node IP addresses per tenant. Minimum: `1` |
| `maxLoadBalancerIPs` | *int32 | No | | Maximum load balancer IP addresses per tenant. Minimum: `1` |

---

## Limits

The `spec.limits` section constrains resource consumption by teams using this ProviderConfig.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `maxClustersPerTeam` | *int32 | No | | Maximum number of clusters a team can provision. Minimum: `1` |
| `maxNodesPerTeam` | *int32 | No | | Maximum total nodes across all clusters for a team. Minimum: `1` |

When limits are set, the Butler controller checks them before allowing a TenantCluster to be created or scaled. If a limit is not set, no restriction is enforced for that dimension.

---

## Status

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Standard Kubernetes conditions. Includes `Ready`, `Validated` |
| `validated` | bool | Whether the provider credentials and configuration have been validated |
| `lastValidationTime` | *Time | Timestamp of the last successful validation |
| `providerVersion` | string | Detected version of the provider (e.g., Harvester version, Nutanix AOS version) |
| `ready` | bool | Whether the provider is reachable and ready to accept MachineRequests |
| `lastProbeTime` | *Time | Timestamp of the last health probe |
| `capacity` | [`Capacity`](#capacity) | Estimated available capacity |

### Capacity

| Field | Type | Description |
|-------|------|-------------|
| `availableIPs` | int32 | Number of IP addresses available in the configured pools |
| `estimatedTenants` | int32 | Estimated number of additional tenant clusters that can be provisioned |

---

## Credential Secret Format

Each provider type expects specific keys in the referenced Secret. Create the Secret before creating the ProviderConfig.

### Harvester

| Key | Description |
|-----|-------------|
| `kubeconfig` | Harvester cluster kubeconfig with permissions to manage VMs |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harvester-credentials
  namespace: butler-system
type: Opaque
stringData:
  kubeconfig: |
    apiVersion: v1
    kind: Config
    clusters:
    - cluster:
        server: https://harvester.example.com:6443
        certificate-authority-data: <base64>
      name: harvester
    # ...
```

### Nutanix

| Key | Description |
|-----|-------------|
| `username` | Prism Central username |
| `password` | Prism Central password |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nutanix-credentials
  namespace: butler-system
type: Opaque
stringData:
  username: admin
  password: <prism-central-password>
```

### Proxmox

Proxmox supports either username/password or API token authentication.

| Key | Description |
|-----|-------------|
| `username` | Proxmox username (e.g., `root@pam`) |
| `password` | Proxmox password |
| `token` | Proxmox API token (alternative to username/password) |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: proxmox-credentials
  namespace: butler-system
type: Opaque
stringData:
  username: root@pam
  password: <proxmox-password>
```

### Azure

| Key | Description |
|-----|-------------|
| `tenantId` | Azure Active Directory tenant ID |
| `clientId` | Service principal application (client) ID |
| `clientSecret` | Service principal client secret |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azure-credentials
  namespace: butler-system
type: Opaque
stringData:
  tenantId: <azure-tenant-id>
  clientId: <service-principal-client-id>
  clientSecret: <service-principal-secret>
```

### AWS

| Key | Description |
|-----|-------------|
| `accessKeyId` | IAM access key ID |
| `secretAccessKey` | IAM secret access key |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: butler-system
type: Opaque
stringData:
  accessKeyId: <aws-access-key-id>
  secretAccessKey: <aws-secret-access-key>
```

### GCP

| Key | Description |
|-----|-------------|
| `serviceAccount` | GCP service account key in JSON format |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gcp-credentials
  namespace: butler-system
type: Opaque
stringData:
  serviceAccount: |
    {
      "type": "service_account",
      "project_id": "my-project",
      "private_key_id": "...",
      "private_key": "-----BEGIN PRIVATE KEY-----\n...",
      "client_email": "butler@my-project.iam.gserviceaccount.com",
      ...
    }
```

---

## Webhook Validation

The ProviderConfig admission webhook enforces the following rules.

### Create and Update

- The provider-specific configuration section must be present and valid for the chosen `spec.provider` type.
- Provider-specific required fields are validated:
  - **Azure**: `subscriptionID` and `resourceGroup` must be set.
  - **AWS**: `region` must be set.
  - **GCP**: `projectID` and `region` must be set.
- If `spec.scope.type` is `team`, `spec.scope.teamRef.name` must be set.
- If `spec.network.mode` is `ipam`, `spec.network.poolRefs` must contain at least one entry.
- IPAM mode is forbidden for cloud providers (`azure`, `aws`, `gcp`).

### Delete

- Deletion is blocked if any TenantCluster references the ProviderConfig via `providerConfigRef`. Remove or migrate all dependent clusters before deleting.

---

## Examples

### On-Premises Harvester with IPAM

A platform-scoped Harvester provider using IPAM for IP address management with static load balancer allocation.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: harvester-prod
  namespace: butler-system
spec:
  provider: harvester
  credentialsRef:
    name: harvester-credentials
    namespace: butler-system
  harvester:
    endpoint: https://harvester.example.com:6443
    namespace: default
    networkName: default/vlan40-workloads
    imageName: default/talos-1.9.0
    storageClassName: longhorn
  network:
    mode: ipam
    poolRefs:
      - name: prod-ip-pool
        priority: 10
      - name: overflow-ip-pool
        priority: 5
    subnet: "10.40.0.0/24"
    gateway: "10.40.0.1"
    dnsServers:
      - "10.40.0.2"
      - "10.40.0.3"
    loadBalancer:
      defaultPoolSize: 8
      allocationMode: static
    quotaPerTenant:
      maxNodeIPs: 20
      maxLoadBalancerIPs: 8
  limits:
    maxClustersPerTeam: 5
    maxNodesPerTeam: 50
```

### Nutanix with Static IPAM

A platform-scoped Nutanix provider with IPAM and resource limits.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: nutanix-datacenter
  namespace: butler-system
spec:
  provider: nutanix
  credentialsRef:
    name: nutanix-credentials
  nutanix:
    endpoint: https://prism.example.com
    port: 9440
    insecure: false
    clusterUUID: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    subnetUUID: "f0e1d2c3-b4a5-6789-0fed-cba987654321"
    imageUUID: "11223344-5566-7788-99aa-bbccddeeff00"
    storageContainerUUID: "aabbccdd-eeff-0011-2233-445566778899"
  network:
    mode: ipam
    poolRefs:
      - name: nutanix-node-pool
        priority: 10
    subnet: "10.50.0.0/24"
    gateway: "10.50.0.1"
    dnsServers:
      - "10.50.0.10"
    loadBalancer:
      defaultPoolSize: 4
      allocationMode: static
  limits:
    maxClustersPerTeam: 3
    maxNodesPerTeam: 30
```

### Cloud AWS Provider

An AWS provider using cloud-native networking. No IPAM configuration is needed.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: aws-us-east
  namespace: butler-system
spec:
  provider: aws
  credentialsRef:
    name: aws-credentials
  aws:
    region: us-east-1
    vpcID: vpc-0abc123def456789
    subnetIDs:
      - subnet-0aaa111bbb222ccc
      - subnet-0ddd333eee444fff
    securityGroupIDs:
      - sg-0123456789abcdef0
  limits:
    maxClustersPerTeam: 10
    maxNodesPerTeam: 100
```

### Team-Scoped Azure Provider

An Azure provider scoped to a specific team. Only the `platform-team` can use this ProviderConfig.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: azure-platform-team
  namespace: team-platform-team
spec:
  provider: azure
  credentialsRef:
    name: azure-credentials
    namespace: team-platform-team
  azure:
    subscriptionID: "12345678-abcd-efgh-ijkl-1234567890ab"
    resourceGroup: platform-team-rg
    location: eastus
    vnetName: platform-vnet
    subnetName: tenant-subnet
  scope:
    type: team
    teamRef:
      name: platform-team
  limits:
    maxClustersPerTeam: 5
    maxNodesPerTeam: 40
```

### Elastic IPAM Configuration

A Proxmox provider with elastic load balancer IP allocation. Tenant clusters start with a small pool that grows on demand.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: proxmox-elastic
  namespace: butler-system
spec:
  provider: proxmox
  credentialsRef:
    name: proxmox-credentials
  proxmox:
    endpoint: https://proxmox.example.com:8006
    insecure: false
    nodes:
      - pve-node-01
      - pve-node-02
      - pve-node-03
    storage: local-lvm
    templateID: 9000
    vmidRange:
      start: 200
      end: 999
  network:
    mode: ipam
    poolRefs:
      - name: proxmox-main-pool
        priority: 10
    subnet: "192.168.10.0/24"
    gateway: "192.168.10.1"
    dnsServers:
      - "192.168.10.2"
    loadBalancer:
      defaultPoolSize: 16
      allocationMode: elastic
      initialPoolSize: 2
      growthIncrement: 4
    quotaPerTenant:
      maxNodeIPs: 10
      maxLoadBalancerIPs: 16
  limits:
    maxClustersPerTeam: 8
    maxNodesPerTeam: 40
```

---

## See Also

- [TenantCluster](./tenantcluster.md) -- Uses ProviderConfig for infrastructure provisioning
- [MachineRequest](./machinerequest.md) -- VM lifecycle requests dispatched to provider controllers
- [Team](./team.md) -- Multi-tenancy unit; team-scoped ProviderConfigs restrict access
- [Provider Guides](../../guides/providers.md) -- Provider-specific setup and configuration guides
