# ADR 0012: POS Tickets, Ordering and Service Workflow

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

A production `POSTED` path remains blocked until ADR 0009 (tax formula and rounding), ADR 0010 (invoice/fiscal adapters and payment fiscal-code mapping), and ADR 0011 (Viva Android contract) have their implementation blockers closed.

This ADR does not authorize a kitchen display protocol, table-reservation product, or POS application code.

Amended 2026-08-15: seating session owned by ADR 0013.

## Date

2026-08-15

## Context

ADR 0006 owns the commercial POS document: freeze-on-add-line, one location/storage/currency, and “only `POSTED` creates `SALE`”. Its commercial states are `DRAFT → POSTED → REVERSED` and `DRAFT → CANCELLED`. ADR 0007 owns modifiers. ADR 0010 owns the outgoing Invoice and forbids splitting a posted ticket. ADR 0011 requires `PAYMENT_IN_PROGRESS` and a frozen payload hash before a card charge, and allows a `BANK_TRANSFER` invoice with a `PaymentInstruction`.

Without this ADR, sending food to the kitchen would post a `SALE`, a waiter would overwrite a sent “no onion” line, two devices would send the same beer twice, a split would let parent and child both post the same quantity, one cancelled starter would void the whole table, a prepared void would leave write-off evidence for an undefined later job, and a zero-total or bank invoice would invent a fake `CAPTURED` Payment.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the operational Ticket domain **before** POS and KDS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
Ticket records an operational hospitality order.

Ticket is not an Invoice, Payment, stock document,
kitchen production record or table reservation.
```

The Ticket links those domains. It does not replace them.

```text
A Ticket is a mutable operational order until finalization begins.

Kitchen and service progress are tracked separately per item
and per append-only ProductionInstruction.

ProductionInstruction content is immutable.
Production progress is an append-only event stream.
Current status is derived.

Entering PAYMENT_IN_PROGRESS freezes the complete commercial payload.
It means finalization in progress, even when no provider charge runs.

POSTED creates the final Sale, stock effects, Invoice,
Payment Allocations and required outboxes in one local DB transaction.

A posted Ticket is never edited, cancelled or split.
Every later correction uses explicit reversal and credit-document workflows.
```

ADR 0006 freeze-on-add-line is not a contradiction:

```text
Freeze-on-add-line freezes the selected catalog/commercial snapshot
for that TicketLine version.

It does not make the entire Ticket immutable.
```

Changing product, recipe, tax class, or price context **before send** does not silently overwrite the historical snapshot. Create a new line version, or replace the unsent line. A **sent** snapshot never changes.

A non-sale stock disposition on `VOIDED`, or on a partial cancel of prepared quantity at `POSTED`, is **not** a `SALE`. It does not reopen ADR 0006.

## Decision

### 1. Ticket responsibilities

A Ticket holds: tenant, location, sales channel, service context (table / takeaway), responsible operator, guest or business recipient if any, lines/qty/modifiers, pre-finalization prices and discounts, kitchen/bar notes, operational state, monotonic `version`, and links to Payments, Sale, Invoice, production batches, commercial-ownership transfers, and lineage (split/merge/transfer).

A Ticket is **not** authority for: final tax calculation, fiscal invoice number, JIR/ZKI, money actually received, final stock, provider settlement, or an accounting journal.

### 2. Ticket lifecycle

```text
DRAFT → OPEN → PAYMENT_IN_PROGRESS → POSTED
DRAFT → CANCELLED
OPEN → VOIDED
```

```text
DRAFT
→ explicit Send order
→ OPEN
```

- First successful send creates `OPEN`.
- Later sends create new production batches on the **same** Ticket.
- A Ticket with no kitchen/bar items uses an explicit `Accept order`. That also transitions to `OPEN` and **must not** create an empty production batch.
- `DRAFT` may accumulate lines, including ADR 0007 `incomplete`.
- `CANCELLED` remains only for a Ticket that never executed (no send, no accept, no stock, no invoice).

Other states:

- `OPEN` — order accepted; more batches may be sent. **Does not post `SALE`.**
- `PAYMENT_IN_PROGRESS` — commercial payload frozen; **finalization in progress**. Not only a provider charge.
- `POSTED` — atomic commercial close. Never from mutable `OPEN`.
- `VOIDED` — entire pre-post Ticket closed with no quantity remaining for commercial posting. Not invoice storno.

ADR 0006 `REVERSED` remains the commercial/stock reversal of a posted ticket. It is not a 0012 operational state.

Do not put `COOKING`, `READY`, or `SERVED` on the Ticket itself. One ticket may have a drink served, a main in preparation, and a dessert not sent.

### 3. ProductionInstruction content is immutable

Each send creates an immutable batch. Do not store a mutable `status` on the instruction.

```text
ProductionInstruction
---------------------
ticket_id
batch_sequence
destination
created_by
created_at
idempotency_key

ProductionInstructionLine
-------------------------
ticket_line_id
quantity
product_snapshot
modifier_snapshot
production_note_snapshot
```

```text
ProductionInstruction content is immutable.

Production progress is an append-only event stream.
Current status is derived.
```

```text
ProductionProgressEvent
-----------------------
instruction_line_id
quantity
event_type
occurred_at
recorded_at
actor/device
idempotency_key
```

Event types:

```text
ACKNOWLEDGED
IN_PREPARATION
READY
SERVED
CANCEL_REQUESTED
CANCEL_ACCEPTED
CANCEL_REJECTED
```

Current prep status (`SENT`, `ACKNOWLEDGED`, `IN_PREPARATION`, `READY`, `SERVED`, and cancellation outcomes) is a **projection**. It is not the only proof of history.

- A sent batch is never edited or deleted.
- Historical progress events are never edited or deleted.
- A double-click with the same idempotency key returns the same batch.
- `batch_sequence` is monotonic within the Ticket.
- Kitchen and bar may receive separate batches / destinations.
- Unsent notes may change; a sent note remains the snapshot on that batch.
- Print or KDS retry delivers the existing instruction. It does **not** create a new business instruction.
- A late KDS event must not regress a projection (for example `SERVED` → `IN_PREPARATION`).
- The same provider/device event is processed idempotently.
- A progress event must not cover more than the active quantity.
- `ProductionCancellation.outcome` is produced by a progress event (`CANCEL_ACCEPTED` / `CANCEL_REJECTED`), not a silent overwrite.

### 4. Quantity model

Status on a row is not enough. One line `3 × beer` may be `2 sent`, `1 not sent`, `1 served`, `1 cancelled`.

```text
ordered_qty =
  not_sent_qty
  + active_sent_qty
  + cancelled_qty
```

Preparation and served quantities are derived from active sent quantity and the progress event stream. No derived quantity may become negative. Decimal quantities use the product’s canonical measure precision (ADR 0002 / ADR 0003).

- A new send covers only positive `not_sent_qty`.
- The same quantity unit must not appear in two batches.
- Partial send is allowed (`2/3`, then remainder → second batch, total `3/3`).
- Partial cancel does not cancel the whole line.
- Send under the Ticket lock **recomputes** not-sent quantity. It does not trust a stale client total.

### 5. Sent portion is immutable

```text
NOT_SENT portion
→ may be edited or removed

SENT / PREPARED portion
→ immutable
→ cancellation instruction
→ optional replacement line
```

Changing “no onion” to “with onion” after send is **not** an edit of the existing batch. It is:

```text
cancel old production quantity
+ create replacement Ticket line
+ send new production instruction
```

Kitchen audit stays accurate. The sent snapshot never changes.

### 6. ProductionCancellation is a separate workflow

`CANCELLED` as a production status is the result of events, not a manual overwrite.

```text
ProductionCancellation
----------------------
original_instruction_line_id
quantity
reason_code
requested_by
requested_at
acknowledged_by
acknowledged_at
outcome
```

`outcome` is set only by `CANCEL_ACCEPTED` or `CANCEL_REJECTED` on the progress stream.

Minimum reason codes:

```text
CUSTOMER_CHANGE
ENTRY_ERROR
OUT_OF_STOCK
KITCHEN_REJECTED
QUALITY_REMAKE
MANAGER_VOID
```

- Cancellation does not delete the original instruction.
- Cannot cancel more than `active_sent_qty`.
- Parallel cancellations lock the same line.
- Already prepared or served goods may require manager approval.
- Operational cancel before `POSTED` does **not** automatically create a refund.
- Reason and operator remain audited.
- Prepared/served cancelled quantity must receive a non-sale stock disposition in the same closing transaction that posts or voids the Ticket. Evidence must not wait for an undefined later job.

### 7. `VOIDED` is whole-ticket only

A Ticket with one cancelled line must **not** become `VOIDED`.

```text
VOIDED =
entire pre-post Ticket intentionally closed with no quantity remaining
for commercial posting
```

All of the following are required:

- All not-sent quantities are cancelled.
- All sent quantities have a matching cancellation / void outcome.
- No active or `UNKNOWN` PaymentIntent.
- No irreversible Payment waiting for resolution.
- No Invoice or `SALE` has been created.
- Reason code and operator are mandatory.
- `VOIDED` is terminal and never returns to `OPEN`.

`OPEN → VOIDED` is a full operational close, not “one line was cancelled”.

### 8. Prepared/served void has an immediate stock consequence

`VOIDED` is terminal. Proof of consumed goods must not remain for an undefined later job.

```text
If prepared or served quantity is excluded from commercial posting,
VOIDED must atomically create the required non-sale stock disposition
or a mandatory outbox that deterministically creates it.
```

- Unsent and unprepared quantity has no stock effect.
- Prepared/consumed quantity must get an explicit reason, for example `WASTE`, `STAFF_ERROR`, `CUSTOMER_CANCELLATION`, or `COMPLIMENTARY`.
- That movement is **not** a `SALE` and does not violate ADR 0006.
- There is no automatic write-off merely because an item was `SENT`. There must be proof of a preparation stage, or an authorized decision.
- The whole `VOIDED` transition and the matching stock disposition are atomic.
- On a **partial** cancel, the remainder of the Ticket may still become `POSTED`, but cancelled consumed quantity must receive the matching disposition in that **same** closing transaction.
- The same quantity must not land in both `SALE` and a write-off.

This is the kitchen write-off guardrail.

### 9. Concurrency and optimistic version

The same table may be open on multiple POS devices.

- Ticket has a monotonic `version`.
- Every mutation sends the expected version.
- A stale version returns conflict and refresh.
- Send, under a DB lock, recomputes not-sent quantities.
- `PAYMENT_IN_PROGRESS`, send, split, merge, transfer, and void use the **same** Ticket lock.
- One active payment-freeze claim per Ticket.
- Idempotency is scoped to tenant, device, and business operation.

### 10. `POSTED` is an atomic local close

```text
Ticket POSTED
+ final Sale
+ stock effects
+ Invoice
+ Payment Allocations
+ required outbox records
+ any required non-sale disposition for cancelled prepared quantity
```

must be created in **one local DB transaction**, following ADR 0006, ADR 0010, and ADR 0011.

External HTTP stays outside the transaction. A repeated post with the same key returns the same result: no second Sale, Invoice, or stock posting.

`POSTED` never goes directly from mutable `OPEN`. Every close path enters `PAYMENT_IN_PROGRESS` first.

### 11. Split, merge, and transfer — commercial ownership

Detailed UX stays out of scope. These invariants belong in this ADR.

Production audit may stay on the original Ticket. It must still be unambiguous **which Ticket commercially posts that quantity**. Otherwise parent and child can both post the same product, or neither can.

```text
TicketLineTransfer
------------------
source_ticket_id
destination_ticket_id
source_ticket_line_id
quantity
commercial_ownership
created_by
created_at
idempotency_key
```

- Every not-yet-posted quantity has exactly one current commercial owner.
- Split atomically subtracts quantity from the source Ticket and assigns it to the child Ticket.
- The production batch stays immutably bound to the original Ticket and instruction.
- Transferred quantity on the child Ticket keeps a reference to the original production instruction.
- Only the current commercial owner may include that quantity in `POSTED`.
- The sum of quantities across parent and children equals the quantity before the split.
- Split must not duplicate a discount, tax snapshot, or modifier.
- A split retry returns the same lineage result.
- The same ownership principle applies to merge.

Other transfer invariants:

- Only `DRAFT` and `OPEN` Tickets may transfer to another table.
- Tenant, location, and currency stay the same.
- `PAYMENT_IN_PROGRESS` and `POSTED` cannot be transferred, merged, or split.
- Split moves only not-yet-posted commercial quantities.
- Split must have an explicit parent/child lineage.
- Merge must not lose production batches, discounts, operator, history, or commercial-ownership records.
- Two parallel operations lock all involved Tickets in deterministic order.
- After split, each final Ticket gets at most one original Invoice.

### 12. `PAYMENT_IN_PROGRESS` is finalization in progress

Do not skip the freeze. ADR 0011 already allows a `BANK_TRANSFER` invoice with a `PaymentInstruction`. A `0,00 EUR` invoice or a fully complimentary ticket may also exist.

```text
OPEN
→ final validation
→ frozen payload hash
→ PAYMENT_IN_PROGRESS
→ POSTED
```

`PAYMENT_IN_PROGRESS` means **finalization in progress**, even when no provider charge starts.

```text
CASH/CARD
→ confirmed tender facts

BANK_TRANSFER
→ finalized PaymentInstruction

ZERO_TOTAL
→ no Payment, explicit zero-total reason
```

Then products, qty, modifiers, prices, discounts, tax context, table/channel/location, customer/recipient, and the tender set already being collected cannot change.

- `POSTED` never goes directly from mutable `OPEN`.
- Bank transfer does not require a fake `CAPTURED` Payment.
- A zero-total invoice does not create a fake Payment.
- Zero-total requires an allowed reason and an audited operator.
- All paths use the same frozen payload hash and channel validator.
- An amount greater than zero without a captured tender or an allowed `PaymentInstruction` cannot become `POSTED`.

Clear `FAILED` / `CANCELLED` payment may unlock under control. Card `UNKNOWN` stays frozen and blocks send, edit, split, merge, transfer, and void.

### 13. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Double-click first send → one batch and one transition to `OPEN`.
- Two devices send the same line at once → quantity sent only once.
- Partial send `2/3`, then remainder → two batches, total `3/3`.
- Change an already-sent modifier → cancellation + new line.
- Partial cancel does not void the whole Ticket.
- `UNKNOWN` card blocks send, edit, split, merge, transfer, and void.
- Retry post → one Sale, one Invoice, one stock effect.
- KDS/print fail after the batch exists → retry delivery, not a new batch.
- Split and send race → only one operation succeeds.
- Split of already-sent quantity → production audit stays on the source; only the child commercially posts the quantity.
- Late `IN_PREPARATION` after `SERVED` → no projection regression.
- Prepared quantity + full void → one write-off, no `SALE`, no double stock effect.

## Rejected alternatives

- Treating Ticket as the Invoice or as a stock document.
- `COOKING` as a Ticket status.
- Posting `SALE` when sending to kitchen.
- Auto-`OPEN` on first line, or `OPEN` when seated.
- Editing or deleting a sent production batch.
- Mutating `ProductionInstruction.status` as the history of record.
- Silent in-place edit of a sent modifier or note.
- Treating one cancelled line as Ticket `VOIDED`.
- Using `VOIDED` to storno a posted Invoice.
- Manual overwrite of production status instead of progress events.
- Deferring write-off evidence to an undefined later job after `VOIDED`.
- Automatic write-off merely because an item was `SENT`.
- Letting parent and child both post, or neither post, the same split quantity.
- Going `OPEN → POSTED` without freeze / `PAYMENT_IN_PROGRESS`.
- Creating a fake `CAPTURED` Payment for bank transfer or zero-total.
- Editing, cancelling, or splitting a `POSTED` ticket.
- Unlocking to ordinary `OPEN` after card `UNKNOWN`.
- Reopening ADR 0006’s “only `POSTED` creates `SALE`”.
- Interpreting freeze-on-add-line as whole-Ticket immutability.
- Amending ADR 0002–0005 or 0007–0011 in this change.

## Consequences

### Positive

- Kitchen send is an operational commitment, not a commercial post.
- A sent modifier cannot be silently rewritten; kitchen audit stays accurate.
- Two devices cannot send the same quantity twice.
- Split and merge cannot double-post or drop a quantity.
- A cancelled starter does not void the table.
- Prepared food that is voided leaves an immediate write-off, not a missing later job.
- Bank transfer and zero-total close through the same freeze as cash and card.

### Negative

- Operators must explicitly send or accept before the Ticket is `OPEN`.
- Changing a sent item requires cancel + replacement + a new instruction.
- Full void of prepared food requires an explicit non-sale disposition reason.
- Stale POS clients must refresh on version conflict.

### Neutral

- Documentation can merge without POS, KDS, or warehouse-disposition adapters.
- Course / fire / hold, stock reservation while `OPEN`, and table reservation remain later hooks.
- ADR 0006 commercial `POSTED` / `REVERSED` / `CANCELLED` stay unchanged.

## Invariants

1. Ticket ≠ Invoice ≠ Payment ≠ stock ≠ KDS ≠ reservation. The Ticket links those domains and does not replace them.
2. `DRAFT → OPEN` only via explicit `Send order` or `Accept order`. `Accept order` must not create an empty production batch. Later sends create new batches on the same Ticket.
3. ProductionInstruction **content** is immutable. Progress is an append-only event stream. Current status is a derived projection. A late event must not regress the projection. Print/KDS retry does not create a new instruction.
4. `ordered_qty = not_sent_qty + active_sent_qty + cancelled_qty`. No derived quantity may become negative. The same quantity unit must not appear in two batches.
5. A sent or prepared portion is immutable. A change is cancel + replacement line + new instruction. `ProductionCancellation.outcome` comes from the progress stream. Cancellation does not delete the original instruction and does not automatically refund.
6. `VOIDED` is whole-ticket only, terminal, and not invoice storno. One cancelled line is not `VOIDED`.
7. Prepared or served quantity excluded from commercial posting gets an atomic non-sale stock disposition, or a mandatory outbox that deterministically creates it. There is no automatic write-off merely because an item was `SENT`. The same quantity cannot be both `SALE` and write-off.
8. Ticket has a monotonic `version`. Send, finalization, split, merge, transfer, and void share the same Ticket lock. One active payment-freeze claim per Ticket. Idempotency is scoped to tenant, device, and business operation.
9. `POSTED` is one local DB transaction: Ticket + Sale + stock + Invoice + allocations + required outboxes + any cancel disposition. External HTTP stays outside. Retry returns the same result. `POSTED` never goes from mutable `OPEN`.
10. Every not-yet-posted quantity has exactly one commercial owner (`TicketLineTransfer`). Only that owner may include the quantity in `POSTED`. Production audit stays on the original instruction. Parent + children quantities equal the pre-split quantity.
11. `PAYMENT_IN_PROGRESS` is finalization in progress: confirmed cash/card tenders, a finalized `PaymentInstruction`, or an audited `ZERO_TOTAL`. No fake `CAPTURED` Payment. Card `UNKNOWN` stays frozen and blocks send, edit, split, merge, transfer, and void.
12. A posted Ticket is never edited, cancelled, or split. Later correction uses reversal and credit-document workflows (ADR 0003 / 0010 / 0011).
13. Tenant isolation. Ids alone do not authorize. This ADR stays `Proposed` and does not authorize a production `POSTED` path until ADR 0009, 0010, and 0011 blockers are closed.

## Follow-up ADRs

```text
Table reservation / waitlist
KDS protocol
Courses / fire / hold
Stock reservation while OPEN
Partial return / refund
POS layout
```

The next domain ADR should define **table reservation and waitlist**, or **KDS protocol**, depending on product order. Do not implement kitchen delivery or table booking from this ADR alone.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)
- [ADR 0005: Recipes and Production](0005-recipes-and-production.md)
- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)
- [ADR 0007: POS Modifiers](0007-pos-modifiers.md)
- [ADR 0008: Bundles and Promotions](0008-bundles-and-promotions.md)
- [ADR 0009: Tax Model](0009-tax-model.md)
- [ADR 0010: Invoices and Fiscalization](0010-invoices-and-fiscalization.md)
- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)

## Out of scope

This ADR does not define:

- KDS wire protocol or printer drivers
- table reservation, waitlist, or seating charts
- course / fire / hold UX
- stock reservation while `OPEN`
- detailed split/merge screens
- payments, FX, or Viva Android extras
- tax formula or rounding
- fiscal device protocol, JIR, or fiscal XML
- e-račun
- accounting journals
- partial return or refund documents
- POS screen layout

## Amendment — 2026-08-15: Seating session owned by ADR 0013

The original Decision 1 service context (table / takeaway) and Decision 11 “transfer to another table” remain in the original text. ADR 0013 owns the seating model.

```text
Ticket belongs to an optional seating_session_id,
not permanently to a table_id.
```

“Transfer table” means changing `SeatingSession` membership when the destination table is free, or an explicit `SeatingSessionMerge` when two active sessions are combined. Adding a free table is not merge.

Finalization freezes a `service_context_snapshot` of every membership that is active at freeze time. A frozen `PAYMENT_IN_PROGRESS` service-context snapshot still cannot change.

This amendment does not change Ticket lifecycle, production batches, quantity rules, `VOIDED`, atomic `POSTED`, commercial ownership after split, or “only `POSTED` creates `SALE`”.
