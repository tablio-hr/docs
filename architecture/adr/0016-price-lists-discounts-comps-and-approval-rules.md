# ADR 0016: Price Lists, Discounts, Comps and Approval Rules

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a pricing engine in application code, loyalty, coupons, gift cards, deposits, service charge, tips, or dynamic pricing.

## Date

2026-08-15

## Context

ADR 0006 freezes unit price, basis, and currency on add-line and forbids mixed ticket currencies. Price is not `Product.price`. A promotional line with price `0` still produces `SALE` if the action has a stock effect. ADR 0008 owns automatic line promotions and forbids stacking those promotions in v1. ADR 0009 and ADR 0010 own tax and fiscal treatment. ADR 0012 owns Ticket `version`, sent-line immutability, and payment freeze.

Without this ADR, a waiter would pick a list per row, a missing price would become `0.00`, a happy-hour edit would rewrite open tickets, five 5 % discounts would bypass a 20 % manager rule, and an approval for 10 EUR would still apply after a reprice to 15 EUR.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the pricing and approval domain **before** POS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
Product       = what is sold
PriceList     = starting catalog price
Discount      = controlled reduction
Comp          = full or partial complimentary
Approval      = authorization to deviate
TicketLine    = frozen financial result
```

Price is not a live link to the list. Add-line stores the calculation on the Ticket line. A later list change does not rewrite existing lines.

```text
One location may have several published lists.
The backend picks exactly one list per add-line.
The waiter does not pick a list per row.
```

## Decision

### 1. PriceList

```text
PriceList
---------
tenant_id
name
currency
valid_from
valid_until
status          # DRAFT | PUBLISHED | RETIRED
version
```

A published version is immutable. An edit creates a new version. List currency must match the Ticket (ADR 0006).

Validity is `[valid_from, valid_until)` in the **location timezone**. The timezone enters the snapshot. DST must resolve to one unambiguous instant.

Selection precedence, most specific wins:

```text
location + service area + service type
location + service type
location
tenant default
```

Equal specificity and overlapping validity are rejected at **publish**, not at add-line.

Happy hour, terrace, delivery, takeaway, event, wholesale, and internal lists are published lists plus selection rules.

### 2. PriceListEntry targets `sale_action_id`

```text
PriceListEntry
--------------
price_list_version_id
sale_action_id
unit_price
tax_inclusion_mode
```

- Price is bound to `sale_action_id`. Product, sale unit, and stock effect are already defined by ADR 0006.
- One published list version may have only one entry per `sale_action_id`.
- `unit_price` is non-negative and uses the allowed money decimal precision.
- Do not price an ambiguous `product_id` when the same product is sold as piece, glass, bottle, or another action.
- Missing price in the resolved list → the line cannot be added. No silent `0.00`.

### 3. TicketLine snapshot

On add-line the backend stores at least: catalog unit price, resolved list id and version, selection-rule snapshot, location timezone, base / discount / comp / final unit and line totals, currency, applied discounts, and approval ids.

The snapshot must later prove which list was used, which price was found, which rule selected the list, which discounts applied, who approved a deviation, and which final price entered the invoice.

### 4. Reprice ticket

**Reprice ticket** is an explicit action. It is all-or-nothing, under the Ticket lock, and applies only to unsent / not-in-payment lines.

- Compute new catalog prices for every included unsent line.
- If any included line has no new price, change **nothing**.
- Recompute automatic promotions (ADR 0008).
- Do **not** automatically carry a manual discount, Comp, or price override onto the new base.
- Mark unused approval requests `EXPIRED`.
- Require a new manual action and a new approval after a successful reprice.
- Write the new Ticket version only after the whole calculation succeeds.

An approval for a 10 EUR base must not remain valid after a reprice to 15 EUR. Sent, prepared, `PAYMENT_IN_PROGRESS`, or `POSTED` lines are not silently repriced.

### 5. Discount, stacking, Comp, and override

Discount types: `PERCENTAGE`, `FIXED_AMOUNT`, `FIXED_UNIT_PRICE`. Scope: one line, selected lines, or the whole Ticket.

A discount stores reason, source, value, author, time, approval status, application order, and a rule snapshot.

A ticket-level discount **allocates** to lines. Invoice and tax are per line. Decimal arithmetic. Deterministic remainder. No float. Allocated line discounts **sum exactly** to the ticket-level amount.

v1 stacking:

```text
base price
→ automatic discounts (including ADR 0008)
→ manual line discounts
→ ticket-level discount allocation
→ Comp
→ final price
```

- Each discount rule states whether it may combine.
- Comp is always last.
- Line total cannot go below zero.
- A discount cannot exceed the remaining chargeable amount.
- A discount cannot apply to cancelled quantity.
- Apply is idempotent.

`PRICE_OVERRIDE`, `DISCOUNT`, and `COMP` are distinct. A zero manual override is forbidden; that is a hidden Comp.

```text
Comp
----
ticket_line_id
quantity
amount
reason_code
staff_note
requested_by
approved_by
approved_at
```

Comp is not deleting the line and not an unaudited discount. Full or partial. The line stays on the Ticket, in production, stock, and audit. Tax and fiscal treatment of complimentary supply stay ADR 0009 / ADR 0010.

### 6. Approval

Location-scoped `ApprovalRule`: action type, threshold type and value, requester role, approver role, validity, version.

Thresholds may use percent, absolute amount, line value, ticket deviation, or the employee’s cumulative deviation in the shift.

```text
ApprovalRequest
---------------
ticket_id
ticket_version
affected lines and quantities
before totals
proposed after totals
reason
requester
required approval level
status
```

```text
PENDING | APPROVED | REJECTED | CANCELLED | EXPIRED | CONSUMED
```

The approval is valid only for the exact requested change. It is stale if Ticket version, quantities, prices, lines, discount amount, or tax result changed. One consume. The approver must belong to the same tenant and an allowed location.

**Consume is one transaction:**

- lock the Ticket
- re-check version and **cumulative** deviation
- apply discount / Comp / override
- mark the approval `CONSUMED`
- persist all of the above together

If apply fails, the approval stays unconsumed. Two devices must not consume the same approval.

**Thresholds use total effective deviation from the frozen base**, including all active manual discounts, price override, Comp, and previously approved deviations on the same line or Ticket. Five 5 % discounts must not bypass a 20 % manager rule.

Optional `requester_may_approve_own_request: false` (maker-checker). Emergency override is a separate right with reason and stronger audit.

Roles are referenced. ADR 0017 owns the staff identity catalog.

### 7. After send and after pay

Before send, price may change under these rules. After send, the production instruction does not change. A financial change must not erase the original line trace. Comp and discount may be allowed until payment starts, with approval. Quantity change remains ADR 0012 cancel / replacement.

After payment or fiscalization: no price mutation. Correction is reversal or credit (ADR 0010 / 0011 / 0012).

### 8. Money and audit

One currency per Ticket. Decimal only. No implicit mid-step rounding that loses the exact total. The frontend is never the authority for the final calculation.

Audit keeps original and previous price, applied discounts, comps, rejected and consumed approvals, reason, actor **and role at that time**, location, and timestamps. A later role change does not rewrite history. These records are not deleted.

### 9. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Two equally specific overlapping list rules → publish rejected.
- Missing `sale_action_id` in the resolved list → line not added; not `0.00`.
- Two entries for the same `sale_action_id` in one list version → publish rejected.
- Happy-hour list selected by location local time; timezone stored on the snapshot.
- Later list edit does not change an existing open line.
- Reprice with one unpriced included line → nothing changes.
- Reprice success drops manual discount/comp/override and expires unused approvals.
- Five 5 % manual discounts on one line → approval uses cumulative deviation, not the last 5 %.
- Two devices consume the same approval → only one apply succeeds; the other remains or conflicts.
- Failed apply leaves the approval unconsumed.
- Zero manual override → rejected; Comp required.
- Ticket-level discount allocations sum exactly to the ticket discount.
- Sent or `PAYMENT_IN_PROGRESS` line is not silently repriced.
- `POSTED` ticket cannot receive a new discount.

## Rejected alternatives

- A live price on the Product without a versioned list.
- Pricing by ambiguous `product_id` when several sale actions exist.
- Auto-repricing open lines when a list changes.
- Missing price as `0.00`.
- Hidden Comp via a zero manual override.
- Deleting a comped line.
- Frontend-only approval checks.
- An approval that survives Ticket change, reprice, or a second consume.
- Non-atomic consume (apply without `CONSUMED`, or `CONSUMED` without apply).
- Splitting one large discount into many small ones to dodge a threshold.
- Carrying manual discount, Comp, or override across Reprice.
- An approver from another tenant or a disallowed location.
- Float money arithmetic.
- Editing price after fiscalization.
- Amending ADR 0001–0005, 0007, or 0009–0015 in this change.

## Consequences

### Positive

- Terrace, delivery, and happy hour can publish different lists without a waiter picking a list.
- A catalog edit cannot rewrite what was already ordered.
- A manager threshold cannot be gamed with many small discounts.
- An approval cannot be spent twice or kept after a reprice.

### Negative

- Every sellable Sale Action must have a price in the resolved list before add-line.
- Reprice throws away unused approvals and manual deviations; staff must redo them.
- Maker-checker locations cannot self-approve a Comp.

### Neutral

- Documentation can merge without a pricing UI or role catalog.
- ADR 0008 automatic promotions still do not stack with each other.
- Loyalty, coupons, gift cards, deposits, service charge, tips, and dynamic pricing stay later ADRs.

## Invariants

1. Product ≠ PriceList ≠ Discount ≠ Comp ≠ Approval ≠ TicketLine. The add-line price is the resolved published list entry for that `sale_action_id`, frozen on the line.
2. One location may have several published lists. The backend selects exactly one by the locked precedence. Equal overlapping rules are rejected at publish. Validity uses `[valid_from, valid_until)` in the location timezone.
3. One list version has at most one entry per `sale_action_id`. `unit_price` is non-negative decimal money. Missing price fails closed. No silent `0.00`.
4. Reprice is explicit, all-or-nothing, and only for unsent / not-in-payment lines. Manual discount, Comp, and override are not carried. Unused approvals expire. A new Ticket version is written only after the whole calculation succeeds.
5. Application order is base → automatic (0008) → manual line → ticket allocation → Comp. Line total cannot be negative. Ticket-level allocations sum exactly to the ticket discount. Apply is idempotent.
6. `PRICE_OVERRIDE`, `DISCOUNT`, and `COMP` are distinct. Zero override is not Comp. Comp does not delete the line or invent tax treatment.
7. Approval consume locks the Ticket, re-checks version and cumulative deviation from the frozen base, applies the change, and marks `CONSUMED` in one transaction. One consume. Stale after Ticket or tax change.
8. After send, production instructions do not change. After pay or fiscalization, price does not mutate. Correction is reversal or credit.
9. One ticket currency. Decimal money. Frontend is not the pricing authority. Audit keeps prices, deviations, approvals, actor role at the time, and location.
10. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Staff Identity, Roles and Operator Authorization
Gift Cards, Vouchers and Stored Value
Deposits, Prepayments and No-show Charges
Customer Profiles, Consent and Loyalty
```

Do not implement loyalty, coupons, gift cards, deposits, service charge, tips, or dynamic pricing from this ADR.

## See also

- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)
- [ADR 0008: Bundles and Promotions](0008-bundles-and-promotions.md)
- [ADR 0009: Tax Model](0009-tax-model.md)
- [ADR 0010: Invoices and Fiscalization](0010-invoices-and-fiscalization.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)

## Out of scope

This ADR does not define:

- loyalty points or membership
- coupon campaigns or a general promotion engine beyond ADR 0008
- gift cards or stored value
- deposits or prepayments
- service charge
- tips
- dynamic or AI pricing
- staff identity and the role catalog
- tax formula or fiscal XML
- POS screen layout
