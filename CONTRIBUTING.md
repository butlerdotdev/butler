# Contributing to Butler

Thank you for your interest in contributing to Butler! This document provides guidelines and information for contributors.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Where to Contribute](#where-to-contribute)
- [Development Setup](#development-setup)
- [Making Changes](#making-changes)
- [Pull Request Process](#pull-request-process)
- [Design Proposals](#design-proposals)
- [Community](#community)

---

## Code of Conduct

Butler follows the [CNCF Code of Conduct](https://github.com/cncf/foundation/blob/main/code-of-conduct.md). By participating, you are expected to uphold this code. Please report unacceptable behavior to conduct@butlerlabs.dev.

---

## Getting Started

1. **Understand Butler**: Read the [README](README.md) and [Architecture docs](docs/architecture/)
2. **Find something to work on**: Browse issues labeled `good first issue` or `help wanted`
3. **Set up your environment**: Follow the [Development Setup](docs/contributing/development-setup.md) guide
4. **Make your change**: Follow the contribution workflow below
5. **Submit a PR**: Open a pull request following our guidelines

---

## Where to Contribute

Butler is a multi-repository project. Use this guide to find the right place for your contribution:

### CRD Types or API Changes
**Repository**: [butler-api](https://github.com/butlerdotdev/butler-api)
- Adding or modifying CRD fields
- API versioning changes
- Shared type definitions
- Generated client code

### Controller Logic
**Repository**: Depends on the resource being reconciled

| Resource | Repository |
|----------|------------|
| TenantCluster, Team, TenantAddon, ManagementAddon | [butler-controller](https://github.com/butlerdotdev/butler-controller) |
| ClusterBootstrap, MachineRequest | [butler-bootstrap](https://github.com/butlerdotdev/butler-bootstrap) |
| Provider-specific VM provisioning | butler-provider-{harvester,nutanix,...} |

### CLI Changes
**Repository**: [butler-cli](https://github.com/butlerdotdev/butler-cli)
- New commands for butleradm or butlerctl
- Output formatting
- Configuration handling
- Shell completions

### Console/UI Changes
**Repositories**:
- Frontend (React): [butler-console](https://github.com/butlerdotdev/butler-console)
- Backend API (Go): [butler-server](https://github.com/butlerdotdev/butler-server)

### Helm Chart Changes
**Repository**: [butler-charts](https://github.com/butlerdotdev/butler-charts)
- Chart values and defaults
- Deployment templates
- RBAC configurations
- CRD distribution

### Documentation Changes
**Repository**: This repo ([butler](https://github.com/butlerdotdev/butler))
- Architecture documentation
- Getting started guides
- Operations guides
- Governance documents

### Cross-Cutting Features
**Action**: Open an issue in this repo first
- Features spanning multiple repositories
- Architectural changes
- New component proposals

---

## Development Setup

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Go | 1.22+ | Controller and CLI development |
| Node.js | 20+ | Console frontend development |
| Docker | Latest | Container builds, KIND clusters |
| kubectl | 1.28+ | Kubernetes interaction |
| kind | Latest | Local test clusters |
| helm | 3.x | Chart development |

### Clone and Build

```bash
# Clone the component you want to work on
git clone https://github.com/butlerdotdev/butler-controller
cd butler-controller

# Install dependencies
make deps

# Build
make build

# Run tests
make test

# Run locally against a cluster
make run
```

See [docs/contributing/development-setup.md](docs/contributing/development-setup.md) for detailed instructions.

---

## Making Changes

### Branch Naming

Use descriptive branch names:
- `feature/add-cluster-autoscaling`
- `fix/tenant-kubeconfig-format`
- `docs/update-bootstrap-guide`

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

Examples:
```
feat(controller): add cluster auto-healing support

Implements automatic node replacement when health checks fail.
Includes configurable thresholds and cooldown periods.

Closes #123
```

### Sign Your Commits (DCO)

All commits must be signed off to indicate agreement with the [Developer Certificate of Origin](DCO.md):

```bash
git commit -s -m "feat(api): add autoHealing field to TenantCluster"
```

This adds a `Signed-off-by` line to your commit message.

### Code Style

- **Go**: Follow [Effective Go](https://go.dev/doc/effective_go) and run `gofmt`
- **TypeScript**: Follow the project's ESLint configuration
- **Markdown**: Use standard formatting, run link checks

Each repository has linting configured in CI.

---

## Pull Request Process

### Before Opening a PR

1. **Ensure tests pass**: `make test`
2. **Lint your code**: `make lint`
3. **Update documentation** if needed
4. **Sign all commits** with DCO

### PR Template

Use the repository's PR template. Include:
- Description of changes
- Related issue(s)
- Testing performed
- Breaking changes (if any)

### Review Process

1. **Automated checks** run first (CI, DCO verification)
2. **Maintainer review** within 1-2 business days
3. **Address feedback** by pushing additional commits
4. **Squash and merge** once approved

### Cross-Repository Changes

For changes spanning multiple repos:

1. **Open a tracking issue** in this umbrella repo
2. **Split into per-repo PRs** linked to the tracking issue
3. **Coordinate merge order**:
   - `butler-api` first (shared types)
   - Controllers second (implementation)
   - CLI/Console last (user-facing)

Example:
```
butler#123: [Tracking] Add cluster auto-healing support

PRs:
1. butler-api#45: Add autoHealing field to TenantCluster spec
2. butler-controller#67: Implement auto-healing reconciliation
3. butler-charts#23: Add auto-healing config to values.yaml
4. butler-cli#34: Add --auto-healing flag to cluster create
5. butler#123: Update architecture docs
```

---

## Design Proposals

Significant changes require a design proposal before implementation.

### When to Write a Proposal

- New CRD or major CRD changes
- Architectural changes affecting multiple components
- New component addition
- Breaking changes
- Build/release process changes

### Process

1. **Copy the template**: `design/proposals/NNNN-template.md`
2. **Fill in sections**: Problem, Proposal, Alternatives, etc.
3. **Open a PR** for review
4. **Discuss** in PR comments or community meeting
5. **Merge** with status `Accepted` or `Rejected`
6. **Implement** once accepted

See [design/proposals/README.md](design/proposals/README.md) for the full process.

### ADRs (Architectural Decision Records)

For architectural decisions, use the ADR format in `design/adr/`. ADRs document the context, decision, and consequences of significant technical choices.

---

## Community

### Getting Help

- **GitHub Discussions**: Questions, ideas, general discussion
- **Discord**: [discord community](https://discord.gg/cAzWG9qz3K)
- **Issue Tracker**: Bug reports, feature requests
- **Design Proposals**: Significant changes

### Meetings

Community meetings are held bi-weekly. See [community/meetings/](community/meetings/) for schedule and notes.

### Recognition

Contributors are recognized in:
- Release notes
- CONTRIBUTORS file (if you'd like to be listed)
- Community shoutouts

---

## Thank You!

Your contributions make Butler better for everyone. We appreciate your time and effort!

If you have questions, don't hesitate to ask in GitHub Discussions or open an issue.
