# ADR 0013: Tables, Service Areas and Seating

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a floor-plan product, reservation product, KDS protocol, or POS application code.

Amended 2026-08-15: reservations and Seat party owned by ADR 0015.

## Date

2026-08-15

## Context

ADR 0006 owns one Ticket location, storage, and currency. ADR 0012 owns the operational Ticket: explicit send, append-only production, quantity tracking, frozen finalization, and “transfer to another table” for `DRAFT` and `OPEN` Tickets. ADR 0015 will own Reservation. ADR 0010 owns the Invoice.

Without this ADR, a table would be treated as a Ticket, occupancy would be a durable `OCCUPIED` flag, two devices would seat the same free table twice, joining T8 would look like merging two parties, moving guests would rewrite Ticket history, one payment would close the whole stay, and a session opened in error after a kitchen send would be labelled `CANCELLED`.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the seating domain **before** POS floor-plan implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
ServiceArea defines where service happens.
Table represents a stable physical service point.
SeatingSession represents the temporary occupancy of one or more tables.
Ticket records what is ordered during that occupancy.
Reservation represents a future intent to occupy capacity.
```

```text
A physical Table is a stable location resource.

Temporary occupancy is represented by a versioned SeatingSession.
Tickets belong to the SeatingSession, not permanently to a table.

Moving or joining a free table changes the session’s table membership.
Merging two active sessions is a separate audited operation.

Neither rewrites Payment or Invoice history.
A POSTED Ticket stays immutable.
```

Example:

```text
Terrace
→ Table T12
→ SeatingSession for 4 guests
→ Ticket for food
→ additional Ticket for bar
```

ADR 0012 “transfer to another table” is not a contradiction. This ADR owns seating. Guest move of a free table changes `SeatingSession` membership. Merge of two active sessions is `SeatingSessionMerge`. A frozen `PAYMENT_IN_PROGRESS` service-context snapshot still cannot change.

## Decision

### 1. ServiceArea

```text
ServiceArea
-----------
tenant_id
location_id
code
name
type
sort_order
status
```

Types include `MAIN_DINING`, `TERRACE`, `BAR`, `PRIVATE_ROOM`, `DELIVERY_PICKUP`.

- An area belongs to exactly one location.
- `code` is unique within the location.
- A deactivated area accepts no new seating.
- Renaming does not rewrite historical Tickets or frozen snapshots.
- An area is not warehouse storage and not a fiscal business premises.
- Moving a table between areas leaves an audit and is forbidden while the table has active membership.

### 2. ServiceTable

```text
ServiceTable
------------
tenant_id
location_id
service_area_id
code
display_name
capacity
status
sort_order
```

Physical status only:

```text
ACTIVE | OUT_OF_SERVICE | INACTIVE
```

There is no durable `OCCUPIED` on the table. Occupancy is derived from an active membership.

- Table code is unique within the location.
- Capacity is a positive integer.
- A table belongs to one active ServiceArea.
- A table with an active session cannot be deactivated or moved.
- A used table is never deleted; only deactivated.
- Renaming a table does not rewrite history or frozen snapshots.

### 3. Active membership is database-enforced

```text
A table may have at most one membership with left_at = null.
```

- Active membership has DB-enforced uniqueness where the database supports it.
- Opening a session and adding a table lock `ServiceTable`.
- Multiple tables are locked in deterministic ID order.
- Two POS devices cannot occupy the same free table at once.
- An idempotent retry returns the same session. It does not create a second one.
- `OUT_OF_SERVICE` and `INACTIVE` tables cannot receive new membership.

### 4. SeatingSession

```text
SeatingSession
--------------
tenant_id
location_id
status
guest_count
opened_at
opened_by
closing_started_at
closing_started_by
closed_at
closed_by
version
```

```text
OPEN → CLOSING → CLOSED
OPEN → CANCELLED
OPEN → MERGED
```

- `OPEN` — guests are seated; service is running.
- `CLOSING` — explicit Close seating claim. One Ticket in `PAYMENT_IN_PROGRESS` does **not** force the session to `CLOSING`.
- `CLOSED` — occupancy ended under the close rules.
- `CANCELLED` — opened in error, with no execution.
- `MERGED` — terminal derived state of a source session after `SeatingSessionMerge`.

Membership is append-only:

```text
SeatingSessionTable
-------------------
seating_session_id
table_id
joined_at
left_at
```

- Session and member tables stay on the same location (ADR 0006 / ADR 0012).
- A session may include tables from more than one ServiceArea at that location.
- Past membership rows are never rewritten.
- Session has a monotonic `version`. Stale mutations conflict.

### 5. Adding a free table is not merge

Two different cases:

```text
Session A + free table T8
→ add membership to Session A
```

```text
Session A on T4
+ Session B on T8
→ merge two active sessions
```

The second case is not an ordinary `joined_at`.

```text
SeatingSessionMerge
-------------------
source_session_id
destination_session_id
created_by
created_at
idempotency_key
reason
```

- Destination is the canonical survivor.
- Source becomes terminal `MERGED`.
- Only `DRAFT` and `OPEN` Tickets may change `seating_session_id`.
- `PAYMENT_IN_PROGRESS`, `UNKNOWN`, and `POSTED` Tickets keep the original session.
- Merge does not rewrite historical table membership.
- All Tickets, sessions, and tables lock in deterministic order.
- Tenant, location, and currency must be compatible.
- A merge retry returns the same result.
- Lineage must not cycle.

Join of a free table and merge of two active sessions remain different flows.

### 6. Leaving a table

- A table may be removed only under the Session lock.
- `left_at` is written. The membership row is not deleted.
- A session with active dine-in Tickets must not remain without an active table.
- The last table may leave only during a controlled `CLOSING` / `CLOSED` finish.
- Turning a dine-in session into takeaway is not an ordinary table move.
- `PAYMENT_IN_PROGRESS` or `UNKNOWN` blocks a membership change if that membership is part of the frozen service-context hash.
- Each table’s join and leave times stay auditable.

### 7. Ticket belongs to the stay

```text
Ticket → seating_session_id
```

not only `Ticket → table_id`.

Moving guests from T4 to free T8 keeps the same session and Tickets.

```text
One SeatingSession
→ one or more open Tickets
```

POS continues the current open Ticket by default. A new Ticket requires an explicit action.

Examples: a separate bar ticket, a per-guest ticket, another service stream, a prepared split before post.

- One session may have many Tickets.
- One Ticket belongs to at most one session.
- A takeaway or delivery Ticket has no required SeatingSession.
- Guest move of a free table changes session membership, not Ticket history.
- A `POSTED` Ticket stays immutable.
- Closing a session does not edit or delete Tickets.
- Each `POSTED` Ticket still gets at most one original Invoice (ADR 0010).
- Multiple Tickets do not create multiple sessions.
- Each Ticket has one current responsible operator.

### 8. `CLOSING` is an explicit session-level claim

```text
OPEN
→ Close seating
→ CLOSING
```

Entering `CLOSING`:

- Locks the session.
- Re-checks all linked Tickets.
- Blocks a new Ticket, a new send, and new table membership.
- Allows finishing existing Tickets.
- Stores `closing_started_by` and `closing_started_at`.

```text
CLOSING → CLOSED
```

only if all of:

- no `DRAFT`, `OPEN`, or `PAYMENT_IN_PROGRESS` Ticket
- no `UNKNOWN` PaymentIntent
- no unallocated captured Payment
- no unresolved production or cancellation outcome that blocks close
- every table membership has `left_at`

If close is not possible, the session stays `CLOSING` with explicit blockers. Controlled `CLOSING → OPEN` is allowed only before final close, with reason and audit.

### 9. `CANCELLED` is checkable

`CANCELLED` is allowed only when all of:

- no production instructions
- no PaymentIntent and no Payment
- no `POSTED` or `VOIDED` Ticket
- no active Ticket with commercial content
- any empty `DRAFT` Tickets are atomically marked `CANCELLED`
- a reason code is required
- every table membership gets `left_at`
- the session cannot be reopened

If anything executed, use the ADR 0012 ticket void or cancellation workflow. The session ends as `CLOSED`, not a fake `CANCELLED`.

### 10. Frozen service-context snapshot

A session may use several tables and areas during the stay. At finalization the Ticket freezes:

```text
service_context_snapshot
------------------------
seating_session_id
active_table_ids
table_codes
service_area_ids
service_area_codes
guest_count
responsible_operator
captured_at
```

- The snapshot contains every membership that is active at freeze time.
- Later rename of a table or area does not change the snapshot.
- A table used earlier, but no longer active, stays in session membership audit only.
- The Invoice need not print all of this. Audit keeps it.
- A move after freeze is blocked until finalization is resolved under control.

### 11. Guest count and capacity

- `guest_count` is a positive integer for an active dine-in session.
- Guest-count changes are audited.
- Exceeding the sum of table capacities is a **warning**, not an automatic reject.
- `guest_count` is not derived from the number of Tickets.
- A payment split does not change guest count.
- A future Reservation may supply an expected count. The SeatingSession stores the actual count.

### 12. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Two devices open a session on the same table → only one succeeds.
- Adding two free tables in opposite lock order → no deadlock.
- Removing the last table while an `OPEN` Ticket exists → rejected.
- `CLOSING` with one `UNKNOWN` Payment → cannot become `CLOSED`.
- Session cancel after a production send → rejected.
- Rename a table after post → Ticket snapshot stays the same.
- Join of a free table and merge of two active sessions use different flows.
- Capacity exceeded → warning, allowed, audited.

## Rejected alternatives

- Treating a table as a Ticket, Invoice, or stock location.
- Durable `OCCUPIED` on `ServiceTable`.
- Application-only “one active session per table” with no DB uniqueness.
- Treating merge of two active sessions as an ordinary `joined_at`.
- Leaving merge undefined while implying it works.
- Ticket permanently bound only to `table_id`.
- Inventing a new physical table to represent a join.
- Auto-opening a second Ticket without an explicit action.
- One session per Ticket when two checks share one stay.
- Forcing session `CLOSING` because one Ticket entered payment.
- Fake `CANCELLED` after production, payment, or void.
- Rewriting Ticket, Payment, or Invoice history on move.
- Hard-fail when guest count exceeds table capacity.
- Deleting a used table.
- Converting dine-in to takeaway as an ordinary table move.
- Starting Reservation or KDS in this ADR.
- Amending ADR 0001–0011 in this change.

## Consequences

### Positive

- Two devices cannot seat the same free table twice.
- Joining an empty table cannot silently swallow another party.
- Moving guests does not rewrite posted Tickets or invoices.
- One guest paying does not close the rest of the stay.
- A session opened in error cannot be labelled `CANCELLED` after kitchen work.
- Capacity is operational guidance, not a hard physical law.

### Negative

- Operators must explicitly open a second Ticket.
- Merge of two parties is a heavier, audited operation than adding a free table.
- Close seating can stay in `CLOSING` until every blocker is cleared.
- Stale POS clients must refresh on session version conflict.

### Neutral

- Documentation can merge without a floor-plan, reservation, or KDS product.
- Reservation expected covers stay in ADR 0015. This ADR stores actual `guest_count`.
- ADR 0012 commercial Ticket locks stay unchanged.

## Invariants

1. Table ≠ Ticket ≠ SeatingSession ≠ Reservation ≠ warehouse ≠ fiscal premises. Occupancy is derived. A table has no durable `OCCUPIED`.
2. A table may have at most one membership with `left_at = null`, DB-enforced. Opening a session and adding a table lock `ServiceTable` in deterministic ID order. `OUT_OF_SERVICE` and `INACTIVE` tables cannot receive new membership. An idempotent retry returns the same session.
3. Adding a free table is membership. Merging two active sessions is `SeatingSessionMerge`. Destination survives. Source becomes `MERGED`. Only `DRAFT` and `OPEN` Tickets may change `seating_session_id`. Lineage must not cycle.
4. Membership rows are never deleted. Leave writes `left_at`. A session with active dine-in Tickets must keep an active table. The last table leaves only during controlled `CLOSING` / `CLOSED`. Dine-in to takeaway is not an ordinary table move.
5. Tickets belong to an optional `seating_session_id`. One session may have many Tickets. POS continues the current Ticket by default. A new Ticket is an explicit action. Takeaway and delivery need no session. Closing a session does not edit or delete Tickets.
6. `CLOSING` is an explicit Close seating claim. It is not a side effect of one payment. `CLOSED` requires no open or in-progress Tickets, no `UNKNOWN` PaymentIntent, no unallocated captured Payment, no blocking production outcome, and `left_at` on every membership.
7. `CANCELLED` is allowed only when nothing executed. Otherwise the session ends as `CLOSED` through ADR 0012 ticket workflows. A cancelled session cannot be reopened.
8. Finalization freezes a `service_context_snapshot` of then-active memberships. Later rename does not change it. A move after freeze is blocked until finalization is resolved.
9. `guest_count` is the actual positive dine-in count, audited, and not derived from Tickets. Capacity overrun warns; it does not hard-fail.
10. A used table is deactivated, never deleted. Session has a monotonic `version`. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Reservations, Waitlist and Guest Seating
Kitchen, Bar Production Routing and KDS
Deposits, Prepayments and No-show Charges
POS layout / floor-plan UX
```

The next domain ADR should define **Reservations, Waitlist and Guest Seating**, or **KDS protocol**, depending on product order. Do not implement booking or kitchen routing from this ADR alone.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)
- [ADR 0010: Invoices and Fiscalization](0010-invoices-and-fiscalization.md)
- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)

## Out of scope

This ADR does not define:

- table reservation, waitlist, or guest seating charts
- deposits, prepayments, or no-show charges
- KDS wire protocol or printer drivers
- floor-plan chrome or POS screen layout
- staff identity and operator authorization
- offline POS synchronization
- payments, FX, or Viva Android extras
- tax formula, fiscal XML, or e-račun

## Amendment — 2026-08-15: Reservations and Seat party owned by ADR 0015

The original Decision 4 `SeatingSession` lifecycle, one active membership per table, and “Reservation may supply expected covers; the session stores actual `guest_count`” remain in the original text.

ADR 0015 owns Reservation, WaitlistEntry, and **Seat party**.

```text
Seat party creates the SeatingSession.
A reservation may link reservation_id and waitlist_entry_id.

ReservationTableAssignment is not active membership.
A confirmed assignment holds future table capacity
against other confirmed reservations.

Venue-capacity checks count active SeatingSessions
without double-counting a linked reservation.
```

This amendment does not change occupancy derivation, table `version` locking, `SeatingSessionMerge`, or Ticket belonging to the session.
