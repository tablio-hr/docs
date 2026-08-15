# ADR 0015: Reservations, Waitlist and Guest Seating

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a reservation product, channel adapters, deposit collection, guest profiles, or POS application code.

## Date

2026-08-15

## Context

ADR 0013 owns `ServiceArea`, `ServiceTable`, and versioned `SeatingSession`. Occupancy is derived from active membership. ADR 0012 owns the Ticket. A future Reservation may supply expected covers; the session stores actual `guest_count`. Deposits and no-show charges belong to ADR 0031.

Without this ADR, booking a table at 20:00 would create a live session, two confirmed parties could share one table, a walk-in would invent a fake reservation, cancelling a booking would void a Ticket, and seating the same arrived party from reservation and waitlist would open two sessions.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the guest-arrival domain **before** booking implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
Reservation   = expected future arrival
WaitlistEntry = guest waiting for capacity
SeatingSession = guest is actually seated
Ticket        = ordering and payment
```

A reservation must not become an active table session in advance. `SeatingSession` is created only when staff run **Seat party**.

```text
A planned table does not occupy the table now.
A confirmed planned table does reserve that future interval
against other confirmed reservations.
```

```text
A linked Reservation and WaitlistEntry are one arrival.
They may produce at most one SeatingSession.
```

## Decision

### 1. Reservation and allowed transitions

A Reservation belongs to one tenant and one location. It holds at least: local arrival date and time, expected duration, party size, guest name and contact, guest note, staff note, source, status, a stable public identifier, and `version`.

```text
PENDING → CONFIRMED → ARRIVED → SEATED → COMPLETED
PENDING / CONFIRMED / ARRIVED → CANCELLED
CONFIRMED / ARRIVED → NO_SHOW
```

- `SEATED` does not go back to `CONFIRMED`.
- `CANCELLED`, `NO_SHOW`, and `COMPLETED` are terminal.
- A correction of a terminal decision is a controlled compensation with audit, not a silent status overwrite.

`PENDING` is for locations that require manual confirm. Auto-confirm versus manual confirm is a **location** setting, not tenant-wide.

Walk-in is **not** a Reservation source. Sources: `STAFF`, `WEB`, `PHONE`, `CHANNEL`, `IMPORT`, `API`.

External systems store `external_source` and `external_reservation_id`, unique per tenant and source. Upsert is idempotent. A stale external message must not overwrite a newer local change.

### 2. Planned tables are not occupancy

```text
ReservationTableAssignment
```

is not ADR 0013 `SeatingSessionTable`.

- The plan does **not** block a current `SeatingSession`.
- The plan **does** block another **confirmed** reservation on the same table in an overlapping interval.
- Changing the plan does not move seated guests.
- The same physical table may have several non-overlapping future reservations.
- Real assignment is rechecked at **Seat party**. Staff may seat on a different free table, with an audited change from the plan.

Interval:

```text
[start_at, expected_end_at)
```

18:00–20:00 does not overlap 20:00–22:00. The location may add turnaround time between reservations.

Capacity hold:

```text
PENDING     does not hold capacity
CONFIRMED   holds reserved capacity
ARRIVED     still holds reserved capacity
SEATED      reservation hold is atomically replaced
            by SeatingSession occupancy
CANCELLED / NO_SHOW  release reserved capacity
```

### 3. Location `reservation_capacity_mode`

```text
reservation_capacity_mode:
- TABLE_BASED
- VENUE_CAPACITY
- COMBINED
```

- `TABLE_BASED` — must fit a valid stored combination of available tables, including no overlap with other confirmed assignments.
- `VENUE_CAPACITY` — only the total allowed guests in the interval.
- `COMBINED` — both checks; either failure rejects automatic confirm.

Example in one tenant:

```text
Restoran Vodice   COMBINED
Beach bar Srima   VENUE_CAPACITY
Konoba Šibenik    TABLE_BASED
```

**Unassigned tables**

- `TABLE_BASED` and `COMBINED` **must persist a concrete valid `ReservationTableAssignment` combination at confirm**.
- A reservation without a planned table may be **automatically** confirmed only in `VENUE_CAPACITY`.
- Manual confirm without a table in `TABLE_BASED` or `COMBINED` is **audited overbooking**.
- Do not introduce a re-packing optimizer that re-proves all confirmed reservations after every change.

Config is **versioned**. Confirm snapshots: mode, max venue capacity, duration and turnaround rules, and the table-config version used. A later location change does not void an already confirmed reservation. Changing time, party size, or location **re-checks against the current active config** and stores a new snapshot.

Silent overbooking is forbidden. Manual overbooking is a **separate permission and audited decision**, not a fourth mode.

Other location settings: default duration, duration by party size or service type, table turnaround, whether unassigned-table reservations are allowed (`VENUE_CAPACITY` only for auto-confirm), grace / overdue / no-show policy.

### 4. Venue and table checks after seating

- `Seat party` atomically converts the future hold into real occupancy. There is **no moment** between releasing the hold and creating the session in which capacity looks free.
- A venue-capacity check counts relevant `CONFIRMED` / `ARRIVED` reservations **and** active `SeatingSession`s, **without double-counting** a linked party.
- A table-based check after seating sees active ADR 0013 membership (`left_at = null`).

### 5. Deterministic location capacity lock

Confirm, same-location change, and cancel use a deterministic **location capacity lock** for the affected time interval.

- Two requests for the last table must not both confirm.
- Two requests for the last venue covers must not both confirm.
- `COMBINED` always locks resources in the same order: location interval, then tables by ID.
- Reservation change and cancel on one location use that same lock protocol.

**Location change locks both locations.**

- Lock the old and new location in deterministic ID order.
- In one transaction, release old capacity and take new capacity.
- If the new location has no capacity, the old reservation stays **entirely unchanged**.

### 6. Arrival, waitlist, and Seat party

**Mark arrived** sets `ARRIVED`. It does not occupy a table.

Then either **Seat party** or **Add to waitlist**.

```text
A linked Reservation and WaitlistEntry are the same arrival.
They may produce at most one SeatingSession.

One unlinked Reservation or WaitlistEntry
may also produce at most one SeatingSession.
```

Seat party, one transaction:

- Lock the **arrival claim** (the reservation and, if linked, its waitlist entry). Reject if already seated.
- Lock chosen `ServiceTable`s in deterministic ID order (ADR 0013).
- Require no active membership (`left_at = null`).
- Create one `SeatingSession` and add the tables.
- Mark **both** the waitlist entry and the reservation `SEATED` when they are linked.
- Atomically replace the reservation hold with session occupancy.
- The same idempotency key returns the same session.
- Two devices seating the same party — one via reservation, one via waitlist — must not create two sessions.
- Database uniqueness / claim protects the **whole linked arrival**, not each row alone.

Walk-in: create a `SeatingSession` or a `WaitlistEntry` directly. Do not invent a Reservation.

### 7. WaitlistEntry

```text
WAITING | NOTIFIED | ARRIVED | SEATED | LEFT | CANCELLED | EXPIRED
```

A waitlist entry holds: location, party size, name or label, optional contact, queued_at, estimated wait, seating preferences, optional `reservation_id`, order/priority, and an audit reason when staff skip the queue.

The queue is not strict FIFO. A 2-top may seat while an 8-top waits. Staff may pick a fitting party with a recorded reason.

**Notify guest** is not seated. Record call time, channel, attempt result, reply deadline, and an idempotency key. A failed SMS or WhatsApp must not become `NOTIFIED`. External send and local status have a distinct delivery result.

### 8. Overdue and `NO_SHOW`

Clock expiry marks **overdue** only. `NO_SHOW` is an explicit or controlled automated decision with audit. Time alone must not irreversibly close a reservation without a record.

Location settings: allowed lateness, when a reservation becomes a no-show candidate, auto versus staff confirm, and whether the planned table is held during grace. A grace hold is still not ADR 0013 active membership. It remains a confirmed future hold until `NO_SHOW`, `CANCELLED`, or `SEATED`.

### 9. Changes and cancel

Date, time, party size, or location change re-checks capacity under the lock protocol above.

- A `SEATED` reservation is not moved as a future booking.
- Changing a reservation does not automatically change an active `SeatingSession`.
- Cancelling a `SEATED` reservation does not close the session and does not void Tickets (ADR 0012).
- Cancel keeps reason, actor, and time.
- Contact and notes are not deleted from history without later privacy rules (ADR 0027).
- Terminal-status correction is compensation plus audit, not a silent overwrite.

`COMPLETED` when the linked session becomes ADR 0013 `CLOSED`, or an equivalent controlled close. Do not invent Ticket close from the reservation.

### 10. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Two confirmed reservations for the same table in an overlapping interval → second confirm rejected.
- 18:00–20:00 and 20:00–22:00 on the same table → both may confirm.
- `PENDING` does not block a later confirm of the same table/interval.
- `TABLE_BASED` auto-confirm without an assignment → rejected.
- `VENUE_CAPACITY` auto-confirm without an assignment → allowed when venue covers remain.
- Two devices confirm the last venue slot → only one succeeds.
- Location change to a full location → old reservation unchanged.
- Location change that succeeds → old capacity released and new capacity taken in one transaction.
- Seat party from a reservation and from its linked waitlist at once → one session; both rows `SEATED`.
- Seat party retry with the same idempotency key → same session.
- Seat party does not leave a gap where venue capacity looks free.
- Venue check after seating does not count the same party as both hold and session.
- `NO_SHOW` is not set by clock expiry alone; overdue is visible first.
- Failed notify delivery → waitlist stays not `NOTIFIED`.
- Cancel of a `SEATED` reservation → session and Tickets remain.
- Walk-in seating creates no Reservation.

## Rejected alternatives

- Creating a `SeatingSession` at booking time.
- Treating a planned table as current occupancy.
- Allowing two confirmed reservations on the same table in an overlapping interval.
- Auto-confirming `TABLE_BASED` or `COMBINED` without a stored assignment.
- Letting `PENDING` hold capacity.
- Seating a reservation and its linked waitlist into two sessions.
- Counting a seated party twice in venue capacity (hold plus session).
- Releasing a hold before the session exists.
- Location change that locks only the new location.
- Fake Reservation for every walk-in.
- Auto-closing a Ticket when a reservation is cancelled.
- Frontend-only capacity checks.
- An unspecified lock with no location or interval object.
- Tenant-wide reservation settings when locations differ.
- Silent capacity overrun.
- Silent rewrite of `CANCELLED`, `NO_SHOW`, or `COMPLETED`.
- Deleting a reservation instead of a terminal status and audit.
- A fourth hidden overbooking capacity mode.
- A confirm-time optimizer that re-packs all reservations.
- Amending ADR 0001–0012 or 0014 in this change.

## Consequences

### Positive

- Booking 20:00 does not lock tonight’s current guests off the table.
- Two confirmed parties cannot take the same table at the same time.
- A beach bar can use venue covers without assigning tables.
- Walk-in statistics stay honest.
- Cancelling a booking cannot void a sale.
- Reservation and waitlist cannot double-seat one arrival.

### Negative

- `TABLE_BASED` locations must assign tables before automatic confirm.
- Staff cannot silently confirm an unassigned table-based booking.
- Location moves are heavier: both locations lock.
- `NO_SHOW` needs an explicit or controlled decision, not only a clock.

### Neutral

- Documentation can merge without a booking UI, SMS provider, or deposit flow.
- Deposits stay ADR 0031. Guest profiles stay ADR 0021. Channel adapters stay ADR 0022. Privacy erasure stays ADR 0027.
- ADR 0013 occupancy and one-active-membership locks stay unchanged.

## Invariants

1. Reservation ≠ WaitlistEntry ≠ SeatingSession ≠ table occupancy ≠ Ticket. Seat party creates the session. Booking time does not.
2. `ReservationTableAssignment` is not ADR 0013 membership. It does not block current occupancy. A confirmed assignment blocks overlapping confirmed reservations on the same table.
3. `TABLE_BASED` and `COMBINED` confirm must store a concrete assignment. Unassigned auto-confirm is `VENUE_CAPACITY` only. Manual unassigned confirm in table-based modes is audited overbooking.
4. `PENDING` does not hold capacity. `CONFIRMED` and `ARRIVED` hold it. `SEATED` replaces the hold atomically with session occupancy. `CANCELLED` and `NO_SHOW` release it.
5. Venue checks count `CONFIRMED` / `ARRIVED` plus active sessions, without double-counting a linked arrival. Seat party leaves no gap where capacity looks free.
6. Same-location confirm, change, and cancel use a deterministic location-interval lock. Location change locks both locations by ID, releases and takes in one transaction, or leaves the old reservation unchanged.
7. Interval is `[start_at, expected_end_at)` plus location turnaround.
8. A linked Reservation and WaitlistEntry are one arrival and produce at most one `SeatingSession`. Seat party marks both `SEATED`. The database claim covers the arrival, not each row alone.
9. Only the listed transitions are allowed. `SEATED` does not return to `CONFIRMED`. Terminal states are not silently overwritten.
10. Walk-in is not a fake Reservation. Notify is not seated. Failed delivery is not `NOTIFIED`. Clock expiry is overdue, not `NO_SHOW`.
11. Cancelling a `SEATED` reservation does not close the session and does not void Tickets.
12. Tenant isolation. Reservation `version`. Ids alone do not authorize.

## Follow-up ADRs

```text
Deposits, Prepayments and No-show Charges
Customer Profiles, Consent and Loyalty
Ordering Channels, Delivery and External Platforms
Audit Trail, Data Retention and Privacy
Price Lists, Discounts, Comps and Approval Rules
```

Do not implement deposits, guest profiles, channel adapters, or privacy erasure from this ADR.

## See also

- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)
- [ADR 0013: Tables, Service Areas and Seating](0013-tables-service-areas-and-seating.md)

## Out of scope

This ADR does not define:

- deposits, prepayments, or no-show charges
- guest profiles, consent, or loyalty
- channel / OTA adapters beyond the external-id contract
- privacy erasure or retention schedules
- SMS or WhatsApp provider protocols
- floor-plan chrome or POS screen layout
- table-repacking optimizer
- staff identity and operator authorization
- KDS, payments, tax, or fiscal XML

## Amendment — 2026-08-15: Manual overbook permission owned by ADR 0017

The original Decision that silent overbooking is forbidden and that manual overbooking is a separate permission and audited decision remains in the original text.

ADR 0017 owns the permission catalog. Manual overbooking uses:

```text
reservation.overbook
```

Who may hold that permission, and how the operator session is proven, stay ADR 0017. This amendment does not change capacity modes, `ReservationTableAssignment`, or Seat party.

## Amendment — 2026-08-15: Guest snapshot is not a CustomerProfile

The original Decision that a Reservation holds guest name and contact, and that a waitlist entry may hold name or contact, remain in the original text.

ADR 0021 owns `CustomerProfile`. A reservation or waitlist guest snapshot is not a CRM row. Creating a profile is an explicit, audited action. A later profile edit must not rewrite the frozen snapshot. Reservation and waitlist notes stay on that process, not on a CRM note.

This amendment does not change capacity modes, `ReservationTableAssignment`, or Seat party.
