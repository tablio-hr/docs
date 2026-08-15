# Architecture Decision Records

Accepted decisions that shape Tablio. An ADR locks a long-lived boundary so later PRs do not re-open it.

Tablio writes platform ADRs **before** the first implementation when the decision is a tenancy, security, or release boundary. Implementation details stay in the API README and implementation plans.

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-platform-deployment-and-tenancy-boundary.md) | Platform deployment and tenancy boundary | Accepted (2026-08-15) |

## Template

New ADRs use the next free number and this shape:

```markdown
# ADR 00XX: Title

## Status

Proposed | Accepted (YYYY-MM-DD) | Superseded by ADR 00YY

## Date

YYYY-MM-DD

## Context

Why a decision is needed.

## Decision

What we will do.

## Rejected alternatives

What we will not do, and why.

## Consequences

### Positive
### Negative
### Neutral

## Out of scope
```

File name: `00XX-short-kebab-title.md`.
