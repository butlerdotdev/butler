# NNNN: Proposal Title

**Status**: Proposed | Accepted | Rejected | Implemented | Withdrawn

**Author(s)**: @username

**Created**: YYYY-MM-DD

**Last Updated**: YYYY-MM-DD

## Summary

One paragraph summary of the proposal.

## Motivation

### Problem Statement

What problem does this solve? Why is it needed?

### Goals

- Goal 1
- Goal 2

### Non-Goals

- Non-goal 1
- Non-goal 2

## Proposal

### Overview

High-level description of the proposed solution.

### Detailed Design

#### API Changes

```yaml
# Example CRD or API changes
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ExampleResource
spec:
  newField: value
```

#### Component Changes

Which components need to change and how?

| Component | Change |
|-----------|--------|
| butler-api | Add new field |
| butler-controller | Implement reconciliation |

#### User Experience

How will users interact with this feature?

```bash
# CLI example
butlerctl example-command --new-flag
```

### Migration / Upgrade Path

How do existing users migrate to this change?

## Alternatives Considered

### Alternative 1

Description and why it was rejected.

### Alternative 2

Description and why it was rejected.

## Implementation Plan

### Phase 1: Foundation

- [ ] Task 1
- [ ] Task 2

### Phase 2: Implementation

- [ ] Task 3
- [ ] Task 4

### Phase 3: Documentation

- [ ] Update docs
- [ ] Release notes

## Open Questions

- Question 1?
- Question 2?

## References

- [Related Issue](link)
- [External Documentation](link)
