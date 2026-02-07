# Butler Compatibility Matrix

This document tracks version compatibility between Butler components and dependencies.

## Current Release

**Butler Platform Version:** 0.1.x (Alpha)

## Component Versions

| Component | Version | Kubernetes | Status |
|-----------|---------|------------|--------|
| butler-api | v0.1.0 | 1.28-1.31 | Stable |
| butler-controller | v0.1.0 | 1.28-1.31 | Stable |
| butler-bootstrap | v0.1.0 | 1.28-1.31 | Stable |
| butler-cli | v0.1.0 | 1.28-1.31 | Stable |
| butler-console | v0.1.0-beta.1 | 1.28-1.31 | Beta |
| butler-server | v0.1.0-beta.1 | 1.28-1.31 | Beta |
| butler-charts | v0.1.0 | 1.28-1.31 | Stable |
| butler-provider-harvester | v0.1.0 | 1.28-1.31 | Stable |
| butler-provider-nutanix | v0.1.0 | 1.28-1.31 | Stable |

## Dependency Versions

### Platform Dependencies

| Dependency | Tested Version | Notes |
|------------|----------------|-------|
| Talos Linux | 1.9.x | Management cluster OS |
| Steward | 0.1.x | Hosted control planes |
| Cluster API | 1.7.x | Infrastructure management |
| Cilium | 1.17.x | CNI |
| MetalLB | 0.14.x | LoadBalancer |
| Longhorn | 1.7.x | Storage |
| cert-manager | 1.16.x | Certificates |
| Flux | 2.x | GitOps |

### Infrastructure Providers

| Provider | Required Version |
|----------|-----------------|
| Harvester | 1.3.x+ |
| Nutanix AHV | Prism Central 2023.x+ |
| CAPX (Nutanix) | 1.4.x |

## Component Dependencies

| Component | Requires |
|-----------|----------|
| butler-controller | butler-api v0.1.x |
| butler-bootstrap | butler-api v0.1.x |
| butler-cli | butler-api v0.1.x |
| butler-server | butler-api v0.1.x |
| butler-provider-* | butler-api v0.1.x |

## Helm Chart Versions

All charts are distributed via OCI at `oci://ghcr.io/butlerdotdev/charts/`.

| Chart | Version | App Version |
|-------|---------|-------------|
| butler-crds | 0.1.0 | - |
| butler-controller | 0.1.0 | v0.1.0 |
| butler-bootstrap | 0.1.0 | v0.1.0 |
| butler-console | 0.1.0 | v0.1.0-beta.1 |
| butler-addons | 0.1.0 | - |
| butler-provider-harvester | 0.1.0 | v0.1.0 |
| butler-provider-nutanix | 0.1.0 | v0.1.0 |

## Upgrade Compatibility

| From | To | Supported | Notes |
|------|-----|-----------|-------|
| v0.1.x | v0.2.x | TBD | May require CRD migration |

## Tenant Cluster Kubernetes Versions

Butler supports provisioning tenant clusters with:

| Version | Support Level |
|---------|---------------|
| 1.31.x | Full |
| 1.30.x | Full |
| 1.29.x | Full |
| 1.28.x | Full |
| < 1.28 | Not supported |

## Runtime Requirements

### Management Cluster

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| Control Plane Nodes | 1 (single-node) | 3 (HA) |
| Control Plane CPU | 4 cores | 8 cores |
| Control Plane RAM | 16 GB | 32 GB |
| Control Plane Disk | 100 GB | 200 GB |

### Per Tenant Cluster (Hosted CP)

| Resource | Approximate Usage |
|----------|-------------------|
| CPU | 0.5-1 cores per CP |
| RAM | 1-2 GB per CP |
| Storage | Depends on etcd size |

## Notes

- This matrix is updated with each release
- "Tested Version" means CI/CD validated
- Older versions may work but are not officially supported
- Always check release notes for breaking changes
