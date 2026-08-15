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
| [0010](0010-invoices-and-fiscalization.md) | Invoices and Fiscalization | Proposed |
| [0011](0011-payments-and-settlement.md) | Payments and Settlement | Proposed |
| [0012](0012-pos-tickets-ordering-and-service-workflow.md) | POS Tickets, Ordering and Service Workflow | Proposed |
| [0013](0013-tables-service-areas-and-seating.md) | Tables, Service Areas and Seating | Proposed |
| [0014](0014-kitchen-bar-production-routing-and-kds.md) | Kitchen, Bar Production Routing and KDS | Proposed |
| [0015](0015-reservations-waitlist-and-guest-seating.md) | Reservations, Waitlist and Guest Seating | Proposed |
| [0016](0016-price-lists-discounts-comps-and-approval-rules.md) | Price Lists, Discounts, Comps and Approval Rules | Proposed |
| [0017](0017-staff-identity-roles-and-operator-authorization.md) | Staff Identity, Roles and Operator Authorization | Proposed |
| [0018](0018-shifts-cash-drawers-and-daily-closing.md) | Shifts, Cash Drawers and Daily Closing | Proposed |
| [0019](0019-pos-devices-registration-and-configuration.md) | POS Devices, Registration and Configuration | Proposed |
| [0020](0020-offline-pos-operation-and-synchronization.md) | Offline POS Operation and Synchronization | Proposed |

## ADR Roadmap

Reserved numbers for planned architectural decisions. A roadmap entry is not
an accepted decision and does not authorize implementation.

`Planned` is not `Proposed`. A reserved number is not a decision. There is no
file link until the ADR file exists. When the document is created, its row
moves from Roadmap to Index.

Roadmap titles and ordering may be refined before an ADR file exists. Any
roadmap change is made through a normal docs PR. Once an ADR file exists, its
number and identity are permanent. A removed planned topic leaves no tombstone
because it was never an ADR.

A roadmap topic becomes an ADR only when it requires a long-lived,
cross-component boundary or an irreversible architectural decision.
Feature UX and ordinary implementation details remain outside ADRs.

| ADR | Planned title | Status |
|-----|---------------|--------|
| 0021 | Customer Profiles, Consent and Loyalty | Planned |
| 0022 | Ordering Channels, Delivery and External Platforms | Planned |
| 0023 | Supplier Invoices and Accounts Payable | Planned |
| 0024 | Incoming eInvoices and Recipient Fiscalization | Planned |
| 0025 | Accounting Posting and Export | Planned |
| 0026 | Reporting, Analytics and Historical Snapshots | Planned |
| 0027 | Audit Trail, Data Retention and Privacy | Planned |
| 0028 | Public API, Webhooks and Integration Idempotency | Planned |
| 0029 | Menu Publishing, Availability and Dayparts | Planned |
| 0030 | Gift Cards, Vouchers and Stored Value | Planned |
| 0031 | Deposits, Prepayments and No-show Charges | Planned |
| 0032 | Tenant Plans, Entitlements and SaaS Billing | Planned |

## Template

New ADRs take the lowest available reserved number from the ADR Roadmap.
When the document is created, its row moves from Roadmap to Index.
`Planned` appears only in the Roadmap table, never as an Index status.

New ADRs use this shape:

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
