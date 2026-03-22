# Bootstrap Config Reference

Field reference for `butleradm bootstrap` configuration files.

The config is a plain YAML file passed to `butleradm bootstrap <provider> --config <path>`. The CLI parses it into Go structs defined in `butler-cli/internal/adm/bootstrap/orchestrator/config.go`.

---

## Top-Level Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `provider` | string | Yes | -- | Infrastructure provider: `harvester`, `nutanix`, `proxmox`, `aws`, `gcp`, `azure` |
| `cluster` | object | Yes | -- | Cluster specification |
| `network` | object | No | See defaults | Network configuration |
| `talos` | object | No | See defaults | Talos Linux configuration |
| `addons` | object | No | See defaults | Addon configuration |
| `providerConfig` | object | Yes | -- | Provider-specific settings (must match `provider` field) |

---

## cluster

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `cluster.name` | string | Yes | -- | Cluster name. Used for VM names, kubeconfig context, and resource naming. |
| `cluster.topology` | string | No | `ha` | Cluster topology: `single-node` or `ha`. Single-node forces `controlPlane.replicas=1` and skips worker nodes. |
| `cluster.controlPlane` | object | Yes | -- | Control plane node pool |
| `cluster.workers` | object | No | -- | Worker node pool. Ignored when topology is `single-node`. |

### Node Pool (controlPlane / workers)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `replicas` | int | Yes | -- | Number of nodes. Control plane must be 1 (single-node) or 3 (HA). |
| `cpu` | int | Yes | -- | vCPUs per node |
| `memoryMB` | int | Yes | -- | Memory per node in megabytes (e.g., 8192 = 8 GB) |
| `diskGB` | int | Yes | -- | Boot disk size in gigabytes |
| `extraDisks` | array | No | -- | Additional disks (for Longhorn storage) |
| `extraDisks[].sizeGB` | int | Yes | -- | Disk size in gigabytes |
| `extraDisks[].storageClass` | string | No | -- | Optional storage class |

---

## network

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `network.podCIDR` | string | No | `10.244.0.0/16` | Pod network CIDR |
| `network.serviceCIDR` | string | No | `10.96.0.0/12` | Service network CIDR |
| `network.vip` | string | On-prem only | -- | Control plane VIP address. Used by kube-vip. Not applicable for cloud providers. |
| `network.loadBalancerPool` | object | On-prem only | -- | MetalLB IP address range. Not applicable for cloud providers. |
| `network.loadBalancerPool.start` | string | Yes (if pool set) | -- | First IP in the pool (inclusive) |
| `network.loadBalancerPool.end` | string | Yes (if pool set) | -- | Last IP in the pool (inclusive) |

### On-Prem vs Cloud

- **On-prem** (Harvester, Nutanix, Proxmox): Set `vip` and `loadBalancerPool`. kube-vip provides control plane HA, MetalLB provides LoadBalancer services.
- **Cloud** (AWS, GCP, Azure): Do not set `vip` or `loadBalancerPool`. The provider creates a cloud load balancer for control plane HA. The CCM handles LoadBalancer services natively.

---

## talos

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `talos.version` | string | No | `v1.12.1` | Talos Linux version |
| `talos.schematic` | string | No | -- | Talos Image Factory schematic ID. Determines which extensions are included in the image (e.g., iscsi-tools, qemu-guest-agent). |

---

## addons

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `addons.cni.type` | string | No | `cilium` | CNI plugin type |
| `addons.storage.type` | string | No | `longhorn` | Storage provider type |
| `addons.loadBalancer.type` | string | No | `metallb` | Load balancer type (on-prem only) |
| `addons.loadBalancer.addressPool` | string | No | -- | **Deprecated.** Use `network.loadBalancerPool` instead. Legacy IP range string (e.g., `10.40.0.200-10.40.0.250`). |
| `addons.gitOps.type` | string | No | -- | GitOps controller type (e.g., `flux`). Not installed during bootstrap by default. |
| `addons.capi.enabled` | bool | No | `false` | Install Cluster API |
| `addons.capi.version` | string | No | -- | CAPI version |
| `addons.butlerController.enabled` | bool | No | `false` | Install Butler controller |
| `addons.butlerController.version` | string | No | -- | Butler controller version |
| `addons.butlerController.image` | string | No | -- | Butler controller container image |
| `addons.console.enabled` | bool | No | `false` | Install Butler Console |
| `addons.console.version` | string | No | `latest` | Console chart/image version |
| `addons.console.ingress.enabled` | bool | No | `false` | Create Ingress for console (on-prem) |
| `addons.console.ingress.host` | string | No | `butler.<cluster>.local` | Ingress hostname |
| `addons.console.ingress.className` | string | No | -- | Ingress class (e.g., `traefik`) |
| `addons.console.ingress.tls` | bool | No | `false` | Enable TLS on ingress |
| `addons.console.ingress.tlsSecretName` | string | No | -- | TLS secret name (auto-generated if TLS enabled) |
| `addons.console.auth.adminPassword` | string | No | `admin` | Initial admin password |
| `addons.console.auth.jwtSecret` | string | No | (random) | JWT signing secret |

---

## providerConfig

Only one provider block should be set, matching the top-level `provider` field.

### providerConfig.harvester

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kubeconfigPath` | string | Yes | Path to the Harvester kubeconfig file. Supports `~` expansion. |
| `namespace` | string | Yes | Harvester namespace for VMs (e.g., `default`) |
| `networkName` | string | Yes | VM network in `namespace/name` format (e.g., `default/vlan40-workloads`) |
| `imageName` | string | Yes | Talos image in `namespace/name` format (e.g., `default/talos-v1-12-1`) |

### providerConfig.nutanix

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | -- | Prism Central URL (e.g., `https://prism.example.com`) |
| `port` | int | No | `9440` | Prism Central API port |
| `insecure` | bool | No | `false` | Allow insecure TLS (self-signed certs) |
| `username` | string | Yes | -- | Prism Central username |
| `password` | string | Yes | -- | Prism Central password |
| `clusterUUID` | string | Yes | -- | Target Nutanix cluster UUID |
| `subnetUUID` | string | Yes | -- | VM network subnet UUID |
| `imageUUID` | string | Yes | -- | Talos image UUID in Prism Central |
| `storageContainerUUID` | string | No | -- | Storage container for VM disks |
| `hostAliases` | array | No | -- | `/etc/hosts` entries for the KIND node (e.g., `["10.0.0.1 prism.internal"]`). Useful for corporate DNS/Zscaler. |

### providerConfig.proxmox

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | -- | Proxmox API URL |
| `insecure` | bool | No | `false` | Allow insecure TLS |
| `username` | string | Yes | -- | Proxmox username |
| `password` | string | Yes | -- | Proxmox password |
| `nodes` | array | Yes | -- | List of Proxmox node names for VM placement |
| `storage` | string | Yes | -- | Storage location for VM disks |
| `templateID` | int | No | -- | VM template ID to clone |
| `vmidStart` | int | No | -- | Start of VM ID range |
| `vmidEnd` | int | No | -- | End of VM ID range |
| `hostAliases` | array | No | -- | `/etc/hosts` entries for KIND node |

### providerConfig.aws

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `accessKeyID` | string | Yes | -- | IAM access key ID |
| `secretAccessKey` | string | Yes | -- | IAM secret access key |
| `region` | string | Yes | -- | AWS region (e.g., `us-east-1`) |
| `vpcID` | string | No | -- | VPC ID |
| `subnetID` | string | No | -- | Subnet ID for VM placement |
| `securityGroupID` | string | No | -- | Security group ID |
| `instanceType` | string | No | `m5.xlarge` | EC2 instance type |
| `ami` | string | No | (per-region default) | Custom AMI ID. If not set, uses built-in Talos AMI for the region. |

### providerConfig.gcp

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `serviceAccountKeyPath` | string | Yes | -- | Path to GCP service account key JSON file. Supports `~` expansion. |
| `projectID` | string | Yes | -- | GCP project ID |
| `region` | string | Yes | -- | GCP region (e.g., `us-central1`) |
| `zone` | string | No | `{region}-a` | GCP zone |
| `network` | string | Yes | -- | VPC network name |
| `subnetwork` | string | No | -- | Subnet name |
| `machineType` | string | No | -- | GCE machine type (e.g., `n2-standard-4`) |
| `imageProject` | string | No | -- | GCP project containing the Talos image |
| `imageFamily` | string | No | -- | Image family (e.g., `talos-v1-12`). Used if `image` is not set. |
| `image` | string | No | -- | Specific GCE image name. Takes precedence over `imageFamily`. |

### providerConfig.azure

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `clientID` | string | Yes | -- | Service principal app ID |
| `clientSecret` | string | Yes | -- | Service principal password |
| `tenantID` | string | Yes | -- | Azure AD tenant ID |
| `subscriptionID` | string | Yes | -- | Azure subscription ID |
| `resourceGroup` | string | Yes | -- | Pre-existing resource group name |
| `location` | string | Yes | -- | Azure region (e.g., `eastus`) |
| `vnetName` | string | No | -- | Pre-existing VNet name |
| `subnetName` | string | No | -- | Subnet within the VNet |
| `securityGroupName` | string | No | -- | Pre-existing NSG name. **Required for CCM to create LB NSG rules.** |
| `vmSize` | string | No | -- | Azure VM size (e.g., `Standard_D4s_v3`) |
| `imageURN` | string | No | -- | Full ARM resource ID for the Talos image (managed image or gallery image version) |

---

## Defaults

When a field is omitted, these defaults are applied:

| Field | Default Value |
|-------|---------------|
| `cluster.topology` | `ha` |
| `network.podCIDR` | `10.244.0.0/16` |
| `network.serviceCIDR` | `10.96.0.0/12` |
| `talos.version` | `v1.12.1` |
| `addons.cni.type` | `cilium` |
| `addons.storage.type` | `longhorn` |
| `addons.loadBalancer.type` | `metallb` |
| `providerConfig.nutanix.port` | `9440` |
| `providerConfig.gcp.zone` | `{region}-a` |
