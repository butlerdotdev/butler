# Architectural Decision Records

This directory contains Architectural Decision Records (ADRs) for the Butler project.

## What is an ADR?

An ADR captures an important architectural decision made along with its context and consequences. ADRs are immutable once accepted. If a decision changes, a new ADR supersedes the old one.

## ADR Index

| Number | Title | Status |
|--------|-------|--------|
| [ADR-0001](ADR-0001-template.md) | ADR Template | Template |
| [ADR-0002](ADR-0002-dual-cli.md) | Dual CLI Pattern (butleradm/butlerctl) | Accepted |
| [ADR-0003](ADR-0003-crds-as-api.md) | CRDs as the API Contract | Accepted |
| [ADR-0004](ADR-0004-multi-repo.md) | Multi-Repository Architecture | Accepted |
| [ADR-0005](ADR-0005-helm-distribution.md) | Helm Chart Distribution via OCI | Accepted |
| [ADR-0006](ADR-0006-hosted-control-planes.md) | Hosted Control Planes by Default | Accepted |

## When to Write an ADR

Write an ADR when making decisions that:

- Affect multiple Butler components
- Change the API or CRD structure
- Impact how users interact with Butler
- Affect the build, test, or release process
- Are difficult to reverse
- Require significant development effort

## ADR Process

1. **Copy the template**: `cp ADR-0001-template.md ADR-NNNN-title.md`
2. **Fill in sections**: Complete all sections of the template
3. **Open a PR**: Submit for review with status "Proposed"
4. **Discuss**: Address feedback in PR comments or meetings
5. **Merge**: Update status to "Accepted" or "Rejected"

## ADR Status

| Status | Meaning |
|--------|---------|
| Proposed | Under discussion, not yet decided |
| Accepted | Decision made, will be implemented |
| Rejected | Considered but not adopted |
| Deprecated | No longer applies, superseded or obsolete |
| Superseded | Replaced by a newer ADR (link to it) |

## References

- [Michael Nygard's ADR article](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [ADR GitHub organization](https://adr.github.io/)
- [Kubernetes Enhancement Proposals (KEPs)](https://github.com/kubernetes/enhancements)
