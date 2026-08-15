# Architecture Decision Records

Accepted decisions that shape Tablio. An ADR locks a long-lived boundary so later PRs do not re-open it.

Tablio writes platform ADRs **before** the first implementation when the decision is a tenancy, security, or release boundary. Implementation details stay in the API README and implementation plans.

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-platform-deployment-and-tenancy-boundary.md) | Platform deployment and tenancy boundary | Accepted (2026-08-15) |
| [0002](0002-canonical-product-domain.md) | Canonical Product domain | Accepted (2026-08-15) |
| [0003](0003-warehouse-and-inventory.md) | Warehouse and Inventory | Accepted (2026-08-15) |
| [0004](0004-procurement-and-goods-receiving.md) | Procurement and Goods Receiving | Proposed |
| [0005](0005-recipes-and-production.md) | Recipes and Production | Proposed |
| [0006](0006-pos-sales-and-sale-actions.md) | POS Sales and Sale Actions | Proposed |
| [0007](0007-pos-modifiers.md) | POS Modifiers | Proposed |
| [0008](0008-bundles-and-promotions.md) | Bundles and Promotions | Proposed |
| [0009](0009-tax-model.md) | Tax Model | Proposed |

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
