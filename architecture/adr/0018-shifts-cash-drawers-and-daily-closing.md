# ADR 0018: Shifts, Cash Drawers and Daily Closing

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a time-clock product, cash-drawer hardware integration, payroll engine, or POS application code.

## Date

2026-08-15

## Context

ADR 0011 already treats cash-drawer movements as append-only and forbids a count variance from rewriting sales invoices. ADR 0012 owns the Ticket. ADR 0017 owns who may act and reserved `closing.perform`. ADR 0009–0011 own tax, invoice, and payment formulas.

Without this ADR, clock-out would close the till, one drawer close would close the restaurant, a waiter wallet would be invisible, the safe would be force-closed every night, opening float would look like turnover, and two devices would close the same day twice.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the shift, cash-accountability, and business-day domain **before** POS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
StaffShift                 = when the employee works
CashAccountableUnit        = drawer | staff wallet | safe
CashAccountableUnitSession = day-scoped responsibility for DRAWER or STAFF_WALLET
SAFE ledger                = persistent reconstructed balance across days
BusinessDay                = the location’s operating day
CashMovement               = immutable cash event
CashTransfer               = two-sided intra-location cash move
SafeCount                  = periodic safe verification, not a daily close
```

Clock-out does not close a drawer. Clock-out **must not** leave a staff wallet open. Closing one drawer does not close the business day. An active safe does **not** have to close with the day.

## Decision

### 1. BusinessDay

```text
BusinessDay
-----------
tenant_id
location_id
business_date
opened_at
closing_started_at
closed_at
status          # OPEN | CLOSING | CLOSED
version
```

- At most one `OPEN` or `CLOSING` day per location. A new day cannot open until the previous day is controlled-closed.
- `business_date` is not necessarily today’s calendar date. All records use the **location timezone**. Late-night sales after midnight may still belong to the previous business date.
- Location setting `business_day_cutoff` (example `04:00`). A sale on 16 August at 01:30 belongs to business date 15 August when cutoff is 04:00.
- Cutoff and timezone are **snapshotted on open**. A later setting change does not rewrite an open or closed day.

Ticket, Payment, SeatingSession, StaffShift, and day-scoped `CashAccountableUnitSession` (drawer / wallet) stamp `business_day_id` at **create** against the location’s current `OPEN` day. They do not hop at midnight. `CLOSING` / `CLOSED` reject new documents for that day.

`SAFE` movements stamp the day they occurred for reporting. They do not open or close a day-scoped safe session.

### 2. StaffShift

```text
StaffShift
----------
membership_episode_id
location_id
business_day_id
started_at
ended_at
started_by
ended_by
status          # OPEN | CLOSED | CORRECTED
```

Actions: Clock in, Clock out, controlled manager correction (`shift.correct`). Clock in/out is the actor’s own membership unless it is a correction.

- One `OPEN` shift per membership episode at a time (any location). Overlap rejected.
- Bound to an ADR 0017 `ACTIVE` episode at start. Ending membership does not delete shifts.
- Correction keeps original values, reason, and manager. A shift change does not rewrite Tickets or financial authors.
- Payroll, leave, and statutory time-and-attendance stay out of this ADR.

An open shift is **not** authorization. The backend still runs the ADR 0017 checks. Location setting:

```text
operator_requires_open_staff_shift = true | false
```

If true, POS commands require an `OPEN` StaffShift at that location. The shift does not replace roles or permissions.

Clock-out is rejected while the employee has a non-`CLOSED` `STAFF_WALLET` session, unless that session is first closed or transferred by a completed `CashTransfer` / wallet handover.

### 3. CashAccountableUnit

This ADR does **not** model only a physical till.

```text
CashAccountableUnit
-------------------
type            # DRAWER | STAFF_WALLET | SAFE
location_id
currency
status          # ACTIVE | OUT_OF_SERVICE | RETIRED
```

A unit is not a POS device, employee, business day, tender, or invoice.

- **DRAWER** — physical till. Operational, day-scoped session. One POS device may be configured for one drawer. Several devices may share a drawer only if the location explicitly allows it. Device registration stays ADR 0019. `drawer.open_no_sale` (ADR 0017) is hardware pop without a sale.
- **STAFF_WALLET** — cash a specific waiter is responsible for while working. Operational, day-scoped session.
- **SAFE** — location safe. **Not** a day-scoped operational session. Holds cash across business days. Persistent reconstructed balance, audited transfers, and periodic `SafeCount`. Must not be artificially closed and reopened every day.

### 4. Operational session versus persistent safe

`DRAWER` and `STAFF_WALLET` use `CashAccountableUnitSession` bound to a `BusinessDay` and **must** be `CLOSED` before `Close business day`.

`SAFE` does **not** use a daily session. It has a continuous movement ledger. Expected safe cash is reconstructed from all movements. A day close **snapshots** the safe balance at the closing high-water and **does not reset** that balance. The next day continues from the same reconstructed amount. No opening-float / closing-removal pair is invented for the safe at day close.

An active safe does **not** block `Close business day`.

A safe **does** block day close when there is an incomplete transfer to or from the safe, an unresolved `SafeCount`, or an unknown / unresolved safe variance.

### 5. CashAccountableUnitSession

Every open of a **DRAWER** or **STAFF_WALLET** creates a session. Each session has its own opening float, movements, expected cash, count, variance, close, and responsible person. This session type is **not** used for `SAFE`.

```text
CashAccountableUnitSession
--------------------------
accountable_unit_id
business_day_id
membership_episode_id   # required for STAFF_WALLET
opened_by
opening_float
opened_at
closing_started_at
closed_at
ownership_mode snapshot # DRAWER only
status                  # OPEN | CLOSING | CLOSED
version
```

- At most one non-`CLOSED` session per drawer or wallet unit.
- Open requires: `OPEN` BusinessDay, `ACTIVE` unit, the matching open permission, opening float, unit currency (location currency, ADR 0006), operator + audit.
- Opening float is **not** turnover. It must not enter sales totals.

`StaffWalletSession` is the `STAFF_WALLET` case:

- Always `SINGLE_OPERATOR`.
- Bound to one `MembershipEpisode`.
- At most one active wallet per employee per location per currency.
- Another waiter must not post cash into that wallet.
- In `STAFF_WALLETS` and `HYBRID`, a cash collection must be attributed to the **actual operator’s** open wallet.
- Clock-out cannot leave it open.

### 6. Location cash operation mode

```text
cash_operation_mode:
- CENTRAL_DRAWER
- STAFF_WALLETS
- HYBRID
```

Snapshotted on each session open. Changing the setting does not rewrite an `OPEN` session.

- **CENTRAL_DRAWER** — cash sale, refund, and `TIP_IN` go only through a `DRAWER` session. Opening a `STAFF_WALLET` for collection is rejected.
- **STAFF_WALLETS** — cash sale, refund, and `TIP_IN` go only through the operator’s `STAFF_WALLET`. A `DRAWER` or `SAFE` may still exist as float source / end-of-day destination via `CashTransfer`.
- **HYBRID** — waiters collect into wallets and transfer to a main drawer or safe during or at end of shift.

### 7. Drawer ownership mode

```text
drawer_ownership_mode: SINGLE_OPERATOR | SHARED | HANDOVER
```

Applies only to `DRAWER`. Snapshotted on session open. May change only when that drawer has no non-`CLOSED` session.

- **SINGLE_OPERATOR** — one person is responsible for the whole session. Another operator must not create cash movements on that session. Card / non-cash tenders remain allowed.
- **SHARED** — several authorized operators use the till. Each movement stores the real operator. Variance is at **session** level, not per person.
- **HANDOVER** — till stays open; responsibility moves by audited handover: previous responsible, new responsible, time, expected amount, optional counted amount, difference, confirmation by **both** memberships **or** manager override. Does not close the physical drawer.

`STAFF_WALLET` handover is a `CashTransfer` (wallet → wallet), not `drawer_ownership_mode`.

### 8. CashTransfer

Supported audited transfers:

```text
Drawer → Staff wallet        opening float / issue
Staff wallet → Drawer        turn-in
Staff wallet → Safe          direct safe drop
Staff wallet → Staff wallet  controlled handover only
Drawer → Safe                safe drop from till
```

A transfer that keeps cash **inside the location** must have two linked movements in **one transaction**:

- `TRANSFER_OUT` on the source (drawer/wallet session, or the SAFE ledger)
- `TRANSFER_IN` on the destination (drawer/wallet session, or the SAFE ledger)
- same `cash_transfer_id`
- same amount and currency
- recipient confirmation **or** manager override

A lone `CASH_OUT` / `TRANSFER_OUT` without the matching inbound movement is rejected when the cash remains on site.

ADR 0011 `SAFE_DROP` is this transfer into the persistent `SAFE` ledger. It is not a sales reduction and not a daily safe close.

### 9. CashMovement

This ADR owns unit/session lifecycle, transfers, expected cash, and close. ADR 0011 owns Payment facts and “`CASH_SALE` references a Payment”. Movements are not edited or deleted.

```text
OPENING_FLOAT
CASH_SALE
CASH_REFUND
CASH_IN
CASH_OUT
TIP_IN
TRANSFER_OUT
TRANSFER_IN
SAFE_DROP
HANDOVER_ADJUSTMENT
CLOSING_REMOVAL
VARIANCE_ADJUSTMENT
```

- `CASH_SALE` from a successful ADR 0011 cash allocation / `CAPTURED` cash Payment. Idempotent: one Payment → at most one `CASH_SALE` on **one** session (the mode-required drawer or the operator’s wallet).
- `CASH_REFUND` references the matching outbound Payment / reversal and is written atomically with it (ADR 0011), on the same unit type the sale used when still open, otherwise via controlled transfer or exception.
- Standalone `CASH_IN` / `CASH_OUT` require a reason and are only for cash that **leaves or enters the location**. Intra-location movement is `CashTransfer`.
- `TIP_IN` is physical tip cash in the accountable unit (ADR 0011 Tip). Not a `SALE`.
- `SAFE_DROP` is a transfer into `SAFE`, not turnover.
- `CLOSING_REMOVAL` is ADR 0011 `CLOSING_DEPOSIT`.
- `VARIANCE_ADJUSTMENT` is a compensating movement only. It is not the close variance result. ADR 0011 `COUNT_ADJUSTMENT` is not the close variance.
- A fix is a compensating movement.

```text
expected_cash =
  opening_float + cash_sales + cash_in + tip_in + transfer_in
  - cash_refunds - cash_out - transfer_out - safe_drops
  ± compensating_adjustments
```

Expected cash for a drawer/wallet session is derived from that session’s movements. Expected cash for a `SAFE` is derived from the unit’s full movement ledger and is **not** reset at day close. A cache must reconstruct from movements. The frontend is not the authority.

### 10. Count, blind close, and variance

Applies to every drawer/wallet session.

`SAFE` uses periodic `SafeCount` (same count modes and variance rules), not a daily session close. An unresolved `SafeCount` or unknown safe variance blocks day close. A completed `SafeCount` does not reset the safe ledger.

```text
cash_count_mode: TOTAL_ONLY | DENOMINATION
drawer_close_mode: BLIND | EXPECTED_VISIBLE
```

If `DENOMINATION`, the backend derives the total from `CashCount` rows (denomination × pieces). The client must not send only a bare total.

`BLIND`: the operator enters counted cash **before** seeing expected; then the system shows variance. Manager and audit see both afterwards.

```text
variance = counted_cash - expected_cash
```

Variance is a **close result**. It does not change expected cash or invoices.

Location approval rules: tolerance without approval; manager threshold; block-close threshold; mandatory reason; mandatory recount. This is **not** an ADR 0016 Ticket `ApprovalRequest`. Consume still uses ADR 0017 identity / episode / location / session checks and `drawer.variance_approve` (wallets use the same approve permission). `SafeCount` uses `safe.count`.

### 11. Close drawer or wallet session

Under session lock:

```text
OPEN → CLOSING → CLOSED
```

Stop new cash movements → high-water → expected → counted → variance → approval if needed → closing snapshot → `CLOSED`.

This close path is **not** used for `SAFE`.

One operator must not close while another posts a cash sale on that session. A concurrent write after high-water is rejected. An idempotent retry returns the same close.

**Session close blockers:** cash Payment `UNKNOWN` attributed to this session; unallocated cash Payment on this session; started unfinished cash refund; cash movement or `CashTransfer` without a final result; unresolved variance approval; write after high-water.

An open Ticket **without cash activity on this session** does not by itself block **this** session close.

### 12. Close business day

Not a hand-typed total. A controlled snapshot of references and frozen summaries from ADR 0009–0011. This ADR does **not** own tax or settlement formulas.

The snapshot includes at least: all drawer and wallet sessions; **safe balance at the closing high-water** (no safe reset); cash sales/refunds; transfers; non-cash payments by method; invoices, voids, credits; tax summaries; discounts/comps; open/unresolved Tickets; `UNKNOWN` payments; open SeatingSessions (`OPEN` / `CLOSING`); open StaffShifts; exceptions and approvals; operator, membership episode, role versions; closing generation; source high-water marks.

```text
OPEN → CLOSING → CLOSED
```

Start close: lock `BusinessDay`, assign one closing claim/generation, reject new documents for that day, re-run all checks under that claim.

- Transient technical failure: stay `CLOSING`, retry the **same** claim. No second snapshot.
- Business blocker: return to `OPEN` with an audited reason.
- `CLOSED` is not reopened. Corrections: reversal, refund, compensating movement, late adjustment, or a new audited **correction snapshot** that does not mutate the original.

**Day-close blockers:** any non-`CLOSED` drawer or staff-wallet session; unresolved drawer/wallet variance; incomplete `CashTransfer` (including to or from the safe); unresolved `SafeCount` or unknown safe variance; active Ticket (`DRAFT` / `OPEN` / `PAYMENT_IN_PROGRESS`) or SeatingSession (`OPEN` / `CLOSING`) stamped to that day; Payment `UNKNOWN`; unallocated Payment; invoice/fiscal in unknown or in-flight state; unfinished refund/reversal; unpublished critical financial event; another active closing claim.

An open staff wallet blocks day close the same way an open drawer does. An active safe does **not**.

```text
open_shift_on_day_close: BLOCK | AUTO_CLOSE_WITH_EXCEPTION
```

`AUTO_CLOSE_WITH_EXCEPTION` requires `business_day.exception` or `shift.correct` and is audited. It does not rewrite historical Tickets. It **cannot** auto-close an open wallet; the wallet must be closed or transferred first.

### 13. Late events

After `CLOSED`, no silent write into that snapshot. A late external result links to the original document, is recorded as a late adjustment, posts to the **current OPEN** day (or a controlled adjustment period), and is visible on reports for **both** days. The closed snapshot stays immutable.

### 14. Permissions

ADR 0017 owns the catalog. Reserved `closing.perform` is `business_day.close`. `drawer.open_no_sale` stays. This ADR adds:

```text
shift.clock_in
shift.clock_out
shift.correct
drawer.open
drawer.use
drawer.handover
drawer.cash_in
drawer.cash_out
drawer.safe_drop
drawer.close
drawer.variance_approve
wallet.open
wallet.close
wallet.handover
cash.transfer
safe.count
business_day.open
business_day.close
business_day.exception
```

The backend checks the permission **and** the business rule. A role name is not enough.

### 15. Audit snapshot

Close keeps: business day + timezone/cutoff snapshot; drawer and wallet sessions; opening float; expected and counted; variance and approval; transfers; **safe high-water balance** (ledger not reset); payment totals by method; invoice/fiscal totals; refund/reversal totals; discount/comp totals; blockers and exceptions; operator, membership episode, role versions; closing generation; source high-waters.

A later change of role, timezone, cutoff, cash mode, or unit config does not rewrite a historical close. Business records reference `StaffMembership` (ADR 0017).

### 16. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Clock-out leaves a **drawer** `OPEN`; closing drawer A does not close the BusinessDay or drawer B.
- Clock-out with an open staff wallet is rejected until close or completed transfer/handover.
- Second `OPEN`/`CLOSING` BusinessDay at the same location is rejected.
- 16 August 01:30 with cutoff 04:00 stamps business date 15 August; after close, a new sale stamps the new `OPEN` day.
- Two overlapping `OPEN` StaffShifts for one episode are rejected.
- `operator_requires_open_staff_shift=true` without an open shift: POS command rejected; ADR 0017 permission alone is not enough.
- `CENTRAL_DRAWER`: opening a collection wallet is rejected; cash sale on a wallet is rejected.
- `STAFF_WALLETS` / `HYBRID`: cash sale is attributed to the actual operator’s open wallet; another waiter’s wallet is rejected.
- Second active wallet for the same episode + location + currency is rejected.
- `SINGLE_OPERATOR` drawer: second operator cash movement rejected; card tender allowed.
- `HANDOVER` drawer without both confirmations or manager override is rejected; session stays `OPEN`.
- Wallet → wallet transfer without both confirmations or manager override is rejected.
- Intra-location `TRANSFER_OUT` without matching `TRANSFER_IN` in the same transaction is rejected.
- Opening float is in expected cash and not in sales totals. Safe drop / transfer to SAFE reduces source expected cash, not sales.
- Duplicate `CASH_SALE` for the same Payment is rejected / idempotent.
- `BLIND`: expected hidden until counted cash is stored; then variance is shown.
- `DENOMINATION`: client-only total without rows is rejected; backend total wins.
- Variance does not change invoices or reconstructed expected cash.
- Unit close with cash `UNKNOWN` or unallocated cash Payment on that session is rejected.
- Concurrent cash sale during unit `CLOSING` / after high-water is rejected.
- Open Ticket without cash on that session does not block that session close.
- Open Ticket, SeatingSession, drawer session, or staff wallet stamped to the day blocks **day** close.
- An active safe with a reconstructed balance does **not** block day close; the snapshot stores the high-water safe balance and the next day continues from that balance with no invented open/close movements.
- Incomplete transfer to or from the safe, unresolved `SafeCount`, or unknown safe variance **does** block day close.
- `AUTO_CLOSE_WITH_EXCEPTION` on an open shift still fails if a wallet is open.
- Technical close failure stays `CLOSING`; retry same claim; no second snapshot.
- Business blocker returns day to `OPEN` with audit.
- `CLOSED` day reopen is rejected; late result is a late adjustment on the current OPEN day; original snapshot unchanged.
- After cutoff/timezone/role/cash-mode change, a historical close snapshot is unchanged.

## Rejected alternatives

- StaffShift = unit session.
- Unit session = business day.
- POS device = cash drawer.
- Modeling only a physical `CashDrawer`.
- Treating `SAFE` as a daily session that must close and reopen.
- Inventing safe opening-float / closing-removal pairs at day close.
- Resetting the safe balance in the daily snapshot.
- Blocking day close merely because the safe is active.
- Opening float as turnover.
- Safe drop as a sales reduction.
- Edit or delete of a CashMovement.
- One-sided `CASH_OUT` for cash that stays in the location.
- Clock-out that leaves a staff wallet open.
- Another waiter posting into a wallet.
- Operator sees expected before a blind count.
- Variance that rewrites sales or expected cash.
- Close with an `UNKNOWN` payment.
- Two concurrent closes.
- Automatic reopen of a `CLOSED` day.
- Silent late write into a closed snapshot.
- Frontend as expected-cash authority.
- A second closing snapshot for a retry of the same claim.
- `closing.perform` left as a second name beside `business_day.close`.
- Amending ADR 0001–0010 or 0013–0016 in this change.

## Consequences

### Positive

- A waiter can clock out without closing the till.
- Waiter wallets and a persistent safe are first-class, so cash responsibility is visible.
- Intra-location cash cannot disappear as a one-sided `CASH_OUT`.
- A daily close cannot invent safe movements or reopen yesterday’s snapshot.

### Negative

- Every location must choose a cash operation mode and keep drawer/wallet sessions closed before day close.
- Clock-out is blocked until the wallet is closed or transferred.
- Blind count and two-sided transfers add operational steps.

### Neutral

- Documentation can merge without a time-clock UI, drawer hardware, or payroll export.
- Device pairing stays ADR 0019. Offline sync stays ADR 0020.
- ADR 0011 still owns Payment facts. ADR 0017 still owns who may act.

## Invariants

1. StaffShift ≠ CashAccountableUnit ≠ CashAccountableUnitSession ≠ SAFE ledger ≠ BusinessDay ≠ CashMovement ≠ CashTransfer ≠ Payment.
2. At most one `OPEN` or `CLOSING` BusinessDay per location. Documents stamp `business_day_id` at create. `business_date` uses location timezone and snapshotted cutoff.
3. One `OPEN` StaffShift per membership episode. Shift is not authorization. Clock-out cannot leave a staff wallet open.
4. `DRAWER` and `STAFF_WALLET` sessions are day-scoped and must be `CLOSED` before day close. `SAFE` is a persistent reconstructed ledger and is not closed daily.
5. Opening float is not turnover. Safe drop / transfer to SAFE is not a sales reduction. Expected cash is derived. Frontend is not the authority.
6. Intra-location cash movement is a two-sided `CashTransfer` in one transaction. `CASH_SALE` is idempotent to one Payment and one mode-required session.
7. Variance is a close result. It does not rewrite invoices or expected cash. `SafeCount` does not reset the safe ledger.
8. Day close uses one claim: technical failure stays `CLOSING` and retries the same claim; a business blocker returns `OPEN`. `CLOSED` is not reopened. Late events do not mutate the snapshot.
9. An active safe does not block day close. An open drawer, open wallet, incomplete transfer, unresolved `SafeCount`, or unknown safe variance does.
10. Backend checks ADR 0017 permission and this ADR’s business rule. Role name is not enough. Historical close snapshots are immutable.

## Follow-up ADRs

```text
POS Devices, Registration and Configuration
Offline POS Operation and Synchronization
Accounting Posting and Export
Reporting, Analytics and Historical Snapshots
```

Do not implement device registration, offline sync, payroll, or accounting journals from this ADR.

## See also

- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)
- [ADR 0017: Staff Identity, Roles and Operator Authorization](0017-staff-identity-roles-and-operator-authorization.md)

## Out of scope

This ADR does not define:

- POS device registration, certificates, or kiosk provisioning (ADR 0019)
- offline synchronization (ADR 0020)
- payroll or statutory time-and-attendance
- bank settlement or provider reconciliation (ADR 0011 Settlement)
- accounting posting (ADR 0025)
- BI reports (ADR 0026)
- default cutoff time, denomination lists, or how often `SafeCount` must run
- POS screen layout

## Amendment — 2026-08-15: Device-to-drawer mapping owned by ADR 0019

The original Decision that a CashDrawer is not a POS device, and that several devices may share a drawer only if the location explicitly allows it, remain in the original text.

ADR 0019 owns `EffectiveDeviceConfig`, including device↔drawer pairing. This ADR still owns `cash_operation_mode` and whether the location allows a shared drawer.

A closing claim that depends on a device blocks that device from leaving `REASSIGNING`.

This amendment does not change StaffShift, BusinessDay close, two-sided `CashTransfer`, or the persistent `SAFE` ledger.
