# Butler Roadmap

This document outlines the planned development for Butler.

> **Note**: This roadmap represents current plans and is subject to change based on community feedback and priorities.

## Table of Contents

- [Current Status](#current-status)
- [2025 Q1](#2025-q1)
- [2025 Q2](#2025-q2)
- [2025 H2](#2025-h2)
- [Future Vision](#future-vision)
- [Completed Milestones](#completed-milestones)

---

## Current Status

**Version**: 0.1.x (Alpha)

Butler is currently in alpha with core functionality working:

- Management cluster bootstrap (Harvester, Nutanix)
- Tenant cluster provisioning with hosted control planes
- Basic addon management (Cilium, MetalLB, Longhorn)
- CLI tools (butleradm, butlerctl)
- Web Console (beta)
- Team-based multi-tenancy
- Google SSO integration  

---

## 2025 Q1

### Platform Stability

- [ ] **Comprehensive E2E testing** - Automated integration tests
- [ ] **Documentation completion** - Full operational guides
- [ ] **Error handling improvements** - Better failure messages and recovery

### Authentication & Authorization

- [ ] **Microsoft Entra ID SSO** - Enterprise AD integration
- [ ] **Okta SSO** - Additional IdP support  
- [ ] **RBAC enforcement** - API-level role enforcement (not just UI)

### Console Improvements

- [ ] **GitOps export workflow** - Full implementation
- [ ] **Provider management UI** - CRUD for ProviderConfigs
- [ ] **Delete confirmation** - Type cluster name to confirm

### CLI Enhancements

- [ ] **Apple code signing** - macOS notarization
- [ ] **Shell completions** - bash, zsh, fish
- [ ] **Interactive mode** - Guided cluster creation

---

## 2025 Q2

### New Providers

- [ ] **Proxmox VE provider** - butler-provider-proxmox
- [ ] **VMware vSphere provider** - butler-provider-vsphere

### Talos Tenant Clusters

- [ ] **Talos worker support** - Talos Linux for tenant workers
- [ ] **butler-trustd service** - Certificate signing for Steward integration
- [ ] **Full talosctl support** - Management of Talos tenant clusters

### Advanced Features

- [ ] **Cluster auto-scaling** - Scale based on workload
- [ ] **Cluster templates** - Reusable cluster configurations
- [ ] **Audit logging** - Track all cluster operations

### Addon Ecosystem

- [ ] **ArgoCD addon** - GitOps alternative to Flux
- [ ] **Velero addon** - Backup and disaster recovery
- [ ] **External Secrets addon** - Secret management integration

---

## 2025 H2

### Enterprise Features

- [ ] **Cost tracking** - Resource usage per team
- [ ] **Compliance policies** - Enforce security standards
- [ ] **Self-service onboarding** - User registration flow
- [ ] **API tokens** - Service account authentication

### Cloud Providers

- [ ] **AWS provider** - EKS-like experience on AWS
- [ ] **Azure provider** - AKS-like experience on Azure
- [ ] **GCP provider** - GKE-like experience on GCP

### Platform Maturity

- [ ] **v1.0 release** - Stable API, production-ready
- [ ] **High availability documentation** - Disaster recovery guides
- [ ] **Performance benchmarks** - Published scaling limits

---

## Future Vision

### Concierge (Internal Developer Platform)

Customized Backstage-based IDP that integrates with Butler:

- Service catalog with Butler cluster creation
- Developer self-service portal
- Template-driven application onboarding

### AI-Adjacent Services

- **Knowledge ingestion plugins** - RAG for platform documentation
- **Observability anomaly detection** - ML-powered alerting
- **Natural language cluster management** - "Create a production cluster for team X"

### Ecosystem Expansion

- **Crossplane integration** - Managed cloud resources
- **Terraform provider** - IaC integration
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
- Helm chart packaging
- OCI registry publishing

### 2025 Q1 (Partial)

- Butler Console (React + TypeScript)
- butler-server API backend
- Google SSO integration
- Team-based multi-tenancy
- ManagementAddon controller
- Single-node topology support  

---

## Contributing to the Roadmap

We welcome community input on priorities:

1. **Feature Requests**: Open a [GitHub Issue](https://github.com/butlerdotdev/butler/issues/new?template=feature_request.md)
2. **Discussions**: Join [GitHub Discussions](https://github.com/butlerdotdev/butler/discussions)
3. **Design Proposals**: Submit a [design proposal](../../design/proposals/)

## See Also

- [COMPONENTS.md](../../COMPONENTS.md) - Component status
- [Compatibility Matrix](../../releases/compatibility-matrix.md) - Version support
