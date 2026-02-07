# Butler Project Governance

This document describes how the Butler project is governed.

## Table of Contents

- [Principles](#principles)
- [Roles](#roles)
- [Decision Making](#decision-making)
- [Contributions](#contributions)
- [Conflict Resolution](#conflict-resolution)
- [Changes to Governance](#changes-to-governance)

---

## Principles

Butler governance is guided by these principles:

1. **Open**: Everything happens in public (code, discussions, decisions)
2. **Welcoming**: We actively seek diverse contributors
3. **Transparent**: Decisions are documented with rationale
4. **Merit-based**: Influence is earned through contributions
5. **Consensus-seeking**: We prefer agreement over voting

---

## Roles

### Contributors

Anyone who contributes to Butler (code, docs, issues, reviews, etc.).

**Rights**:
- Open issues and discussions
- Submit pull requests
- Participate in design discussions

### Reviewers

Contributors with a track record of quality reviews.

**Rights**:
- All contributor rights
- Assigned as reviewers on PRs
- Listed in CODEOWNERS for specific areas

**Responsibilities**:
- Provide timely, constructive reviews
- Help new contributors

### Component Maintainers

Maintainers of specific Butler components.

**Rights**:
- All reviewer rights
- Merge PRs in their components
- Participate in release decisions for their components

**Responsibilities**:
- Triage issues
- Review and merge PRs
- Ensure CI/CD health
- Participate in maintainer meetings

### Core Maintainers

Overall project leadership.

**Rights**:
- All component maintainer rights
- Write access to all repositories
- Final say on architectural decisions
- Approve new maintainers

**Responsibilities**:
- Set project direction
- Coordinate releases
- Resolve disputes
- Manage governance
- Represent the project externally

---

## Decision Making

### Day-to-Day Decisions

Component maintainers make decisions for their areas:

- Merging PRs
- Issue triage
- Minor feature decisions
- Bug fix prioritization

**Process**: Maintainer judgment, can consult others

### Technical Decisions

Decisions affecting component architecture or user-facing behavior:

- New features
- API changes
- Dependency updates

**Process**: 
1. Open an issue or PR
2. Discuss with affected parties
3. Seek lazy consensus (no objections in 72 hours)
4. Document decision in commit/PR

### Architectural Decisions

Decisions affecting multiple components or project architecture:

- New CRDs
- Cross-component changes
- Build/release process
- Deprecations

**Process**:
1. Write an ADR (Architectural Decision Record) in `design/adr/`
2. Open PR for review
3. Minimum 1-week review period
4. Approval from 2+ core maintainers
5. No blocking objections
6. Merge with status `Accepted`

### New Component Proposals

Adding new repositories to the Butler ecosystem:

**Process**:
1. Write a design proposal in `design/proposals/`
2. Identify a core maintainer sponsor
3. Document use cases and maintenance commitment
4. 2-week review period
5. Core maintainer approval

### Governance Changes

Changes to this governance document:

**Process**:
1. Open PR with proposed changes
2. 2-week review period
3. Lazy consensus or 2/3 core maintainer vote

---

## Contributions

### Code Contributions

All code contributions follow the process in [CONTRIBUTING.md](../CONTRIBUTING.md):

1. Sign DCO
2. Follow coding standards
3. Submit PR
4. Address review feedback
5. Merge after approval

### Non-Code Contributions

We value all contributions:

- Documentation improvements
- Issue triage
- Answering questions
- Design reviews
- Testing and bug reports
- Evangelism and talks

---

## Conflict Resolution

### Technical Disputes

1. **Discussion**: Parties discuss in issue/PR
2. **Mediation**: Component maintainer(s) facilitate resolution
3. **Escalation**: Core maintainers make final decision

### Interpersonal Disputes

1. **Direct resolution**: Parties attempt to resolve directly
2. **Mediation**: Core maintainer mediates
3. **Code of Conduct**: If CoC violation, follow reporting process

### Maintainer Disputes

If maintainers cannot reach consensus:

1. Seek input from other maintainers
2. Core maintainers discuss privately
3. Vote if necessary (majority wins)
4. Document decision with rationale

---

## Changes to Governance

This governance model will evolve as Butler grows:

### Current Phase: Founding

- Small maintainer group
- Low ceremony
- Fast decisions
- Focus on building

### Future: Growth

When we have 4-10 maintainers:
- Formalize component ownership
- Regular sync meetings
- More structured decision process

### Future: Maturity

When we have 10+ maintainers:
- Special Interest Groups (SIGs)
- Working groups for cross-cutting concerns
- Steering committee
- Possibly seek CNCF sandbox

---

## References

This governance is inspired by:

- [Kubernetes Governance](https://github.com/kubernetes/community/blob/master/governance.md)
- [Flux Governance](https://github.com/fluxcd/community/blob/main/GOVERNANCE.md)
- [CNCF Project Governance](https://github.com/cncf/project-template/blob/main/GOVERNANCE.md)
