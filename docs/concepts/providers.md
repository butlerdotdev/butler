---
title: Infrastructure Providers
sidebar_position: 5
---

# Infrastructure Providers

Butler uses infrastructure providers to provision VMs for management and tenant cluster nodes. Each provider adapts Butler to a specific infrastructure platform.

## Supported Providers

### On-Prem

| Provider | Platform | Status |
|----------|----------|--------|
| [Harvester](../getting-started/harvester.md) | Harvester HCI (KubeVirt) | Stable |
| [Nutanix](../getting-started/nutanix.md) | Nutanix AHV | Stable |
| Proxmox | Proxmox VE | Planned |

### Cloud

| Provider | Platform | Status |
|----------|----------|--------|
| [AWS](../getting-started/aws.md) | Amazon Web Services | Stable |
| [GCP](../getting-started/gcp.md) | Google Cloud Platform | Stable |
| [Azure](../getting-started/azure.md) | Microsoft Azure | Stable |

## Two-Layer Architecture

Providers operate in two layers:

**Bootstrap layer** -- Thin provider controllers (`butler-provider-harvester`, `butler-provider-aws`, etc.) that run during management cluster creation. Each controller watches MachineRequest CRDs, calls the provider SDK to launch VMs, polls for IPs, and reports status. The same pattern applies across all supported providers.

**Steady-state layer** -- Official Cluster API infrastructure providers (CAPI) manage tenant cluster worker VM lifecycle after the management cluster is running. Butler generates the correct CAPI resources (AWSMachineTemplate, HarvesterMachineTemplate, etc.) based on the provider type in the ProviderConfig.

## ProviderConfig

Each provider requires a ProviderConfig resource that stores credentials and provider-specific settings. ProviderConfigs can be scoped to the entire platform or restricted to a specific team.

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: harvester-prod
  namespace: butler-system
spec:
  provider: harvester
  credentialsRef:
    name: harvester-kubeconfig
  harvester:
    namespace: default
    networkName: default/workloads
```

See [ProviderConfig Reference](../reference/crds/providerconfig.md) for all provider-specific fields.

## Networking Modes

Providers support two networking modes:

- **IPAM mode** (on-prem) -- Butler allocates IP ranges from NetworkPool resources. Used with Harvester, Nutanix, and Proxmox.
- **Cloud mode** -- The cloud provider handles networking (VPCs, subnets, security groups). Used with AWS, GCP, and Azure.

Set the mode in `ProviderConfig.spec.network.mode`.

## See Also

- [Getting Started](../getting-started/) -- Bootstrap guides for each provider
- [Networking and IPAM](./networking.md) -- IP address management for on-prem providers
- [Contributing > Adding Providers](../contributing/adding-providers.md) -- How to implement a new provider
