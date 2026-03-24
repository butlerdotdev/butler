---
title: Roadmap
sidebar_position: 1
---

# Butler Roadmap

This document outlines planned development for Butler. Plans are subject to change based on community feedback and priorities.

## Current Status

**Version**: 0.9.x (Beta)

Butler is in beta with a comprehensive feature set:

- Management cluster bootstrap across 6 providers (Harvester, Nutanix, AWS, GCP, Azure, with Proxmox planned)
- Tenant cluster provisioning with hosted control planes via Steward
- Multi-OS worker nodes (Talos, Rocky, Flatcar, Bottlerocket, Kairos)
- Three control plane exposure modes (LoadBalancer, Ingress, Gateway)
- Enterprise IPAM with NetworkPool and IPAllocation
- Addon management (Cilium, MetalLB, Longhorn, cert-manager, Traefik, FluxCD)
- CLI tools (butleradm, butlerctl) with cloud bootstrap support
- Web Console with terminal access, addon management, and ImageSync UI
- Team-based multi-tenancy with resource quotas
- SSO integration (Google, Microsoft Entra ID, Okta)
- Butler Image Factory for OS image building and syncing
- Configurable control plane resource limits (prevents etcd starvation at scale)

---

## 2026 Q2

### Butler Image Factory

- [ ] **Phase 2: Full image lifecycle** - Automated image builds for all OS types
- [ ] **Provider-native image registration** - Auto-register images on Nutanix, Proxmox
- [ ] **Image garbage collection** - Clean up unused images on providers

### Provider Expansion

- [ ] **Proxmox VE provider** - butler-provider-proxmox
- [ ] **VMware vSphere provider** - butler-provider-vsphere

### Platform Stability

- [ ] **Comprehensive E2E testing** - Automated integration tests across all providers
- [ ] **Performance benchmarks** - Published scaling limits (target: 100+ tenants per management cluster)

### Console Improvements

- [ ] **Butler Portal (Backstage)** - Workspace federation across clusters
- [ ] **Provider management UI** - CRUD for ProviderConfigs

---

## 2026 H2

### Enterprise Features

- [ ] **Cost tracking** - Resource usage per team
- [ ] **Compliance policies** - Enforce security standards (pod security, network policies)
- [ ] **API tokens** - Service account authentication for CI/CD
- [ ] **Audit logging** - Track all cluster operations

### Cluster Operations

- [ ] **Cluster auto-scaling** - Scale workers based on workload pressure
- [ ] **Cluster templates** - Reusable cluster configurations beyond ButlerConfig defaults
- [ ] **In-place Kubernetes upgrades** - Upgrade tenant cluster K8s version without rebuild

### Platform Maturity

- [ ] **v1.0 release** - Stable API graduation from v1alpha1 to v1
- [ ] **High availability documentation** - Disaster recovery and backup guides
- [ ] **Steward CNCF Sandbox submission** - Target 2027

---

## Future Vision

### Butler Portal (Internal Developer Platform)

Backstage-based IDP that integrates with Butler:

- Workspace federation across multiple Butler management clusters
- Service catalog with Butler cluster creation
- Developer self-service portal
- Template-driven application onboarding

### Ecosystem Expansion

- **Crossplane integration** - Managed cloud resources alongside tenant clusters
- **Terraform provider** - IaC integration for Butler resources
- **Pulumi provider** - Modern IaC option

---

## Completed Milestones

### 2024 Q4

- Initial butler-api with core CRDs
- butler-bootstrap controller
- butler-controller for TenantCluster
- Harvester provider implementation
- Nutanix provider implementation
- butler-cli with butleradm and butlerctl
- Helm chart packaging and OCI registry publishing

### 2025 Q1-Q2

- Butler Console (React + TypeScript)
- butler-server API backend
- Google SSO integration
- Team-based multi-tenancy with resource quotas
- ManagementAddon controller
- Single-node topology support
- Microsoft Entra ID and Okta SSO

### 2025 H2

- Steward (community-governed Kamaji fork) stabilization
- CAPI-Steward control plane provider
- Talos worker support with talosconfig generation
- Control plane exposure modes (Ingress, Gateway)
- Enterprise IPAM with NetworkPool/IPAllocation
- Configurable control plane resource limits

### 2026 Q1

- Cloud bootstrap (AWS, GCP, Azure) with butleradm
- Cloud provider controllers (butler-provider-aws/gcp/azure)
- Multi-OS worker provisioning (Talos, Rocky, Flatcar, Bottlerocket, Kairos)
- Butler Image Factory (Phase 1)
- ImageSync CRD and butlerctl image commands
- LoadBalancerRequest CRD for cloud HA control planes
- Butler Console ImageSync UI
- Butler Portal v0.1.0 (Backstage workspaces plugin)

---

## Contributing to the Roadmap

We welcome community input on priorities:

1. **Feature Requests**: Open a [GitHub Issue](https://github.com/butlerdotdev/butler/issues/new?template=feature_request.yml)
2. **Discussions**: Join [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions)
3. **Design Proposals**: Submit a [design proposal](../../design/proposals/)

## See Also

- [COMPONENTS.md](../../COMPONENTS.md) - Component status
