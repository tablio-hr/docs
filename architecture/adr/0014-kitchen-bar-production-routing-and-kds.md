# ADR 0014: Kitchen, Bar Production Routing and KDS

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a KDS wire protocol, printer drivers, or POS application code.

## Date

2026-08-15

## Context

ADR 0012 owns the operational Ticket, immutable `ProductionInstruction` content, quantity tracking, and append-only progress events. ADR 0005 already uses **Production Order** for warehouse make-to-stock (`PRODUCTION_OUT` / `PRODUCTION_IN`). ADR 0013 owns seating. Kitchen send must not post `SALE` (ADR 0006).

Without this ADR, KDS would read live Ticket lines, a waiter would pick stations, one missing bar route would still send pizza, two equally specific routes would fail only at service, an offline screen would create a second dispatch, and `ALL_DESTINATIONS` would be confused with per-line readiness.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks kitchen and bar routing **before** KDS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
The waiter Ticket is not the kitchen document.

The waiter has one send action: Send order for the whole Ticket.
The waiter never chooses destinations.

KDS displays immutable ProductionInstructions created at send,
not the live Ticket.

A later catalog, recipe or routing change
must not rewrite what the kitchen already received.
```

```text
The entire Ticket must classify successfully before anything is sent.
If any production item has no valid route, send nothing
and show the waiter which line has no production station.
```

```text
Ticket #104  →  Send order
dispatch_id D17
  Kitchen        2 × pizza
  Bar            2 × Coca-Cola
  Coffee bar     1 × coffee
  Pastry         1 × cake
```

```text
0012/0014 ProductionInstruction  = guest-order kitchen/bar work
0005 Production Order            = warehouse batch that moves stock
```

## Decision

### 1. Waiter Send is the whole current Ticket

The waiter does not pick lines or stations. One click:

1. Lock the Ticket version and recompute `not_sent_qty` (ADR 0012).
2. For every not-sent production line, resolve the active `ProductionRoute` version.
3. Split lines by production station.
4. Create a separate `ProductionInstruction` per station.
5. Link them with the same `dispatch_id`.

If **any** production line has no valid route, commit nothing. Tell the waiter which line is missing a station. No kitchen-only partial success.

A later Send, after new lines are added, is a **new** `dispatch_id` for the remaining `not_sent_qty`. That is ADR 0012’s later batch, not a waiter line-picker. The waiter cannot send “drinks only” while food stays unsent on purpose.

`Accept order` for a Ticket with no kitchen or bar items stays ADR 0012: `OPEN`, no empty instruction.

ADR 0012’s quantity leftover after a partial `2/3` send is **not** waiter UX here. The waiter always sends all current `not_sent_qty`.

### 2. Send commit is not device delivery

The transaction ends when instructions are durably written. KDS or printer then pick them up with retry.

- Offline KDS must not create a second dispatch.
- `delivery_pending` is not “not sent”.
- The device acknowledges receipt idempotently.
- The same KDS event or ack must not change status twice or notify the waiter twice.

Print or KDS retry delivers the existing instruction. It does not create a new business instruction (ADR 0012).

### 3. ProductionStation

A stable station at a location (`KITCHEN`, `BAR`, `COFFEE_BAR`, `PASTRY`, …). Not a ServiceArea, not warehouse storage, not a fiscal premises.

A route to a deleted, inactive, or other-location station is **not** a valid route.

- Station must belong to the same tenant and location.
- Station must be active at send time.
- Deactivating a station does not change old instructions.
- No automatic fallback to “main kitchen”.

### 4. ProductionRoute — deterministic precedence

Not a single hardcoded place on the product for every location.

Publish-time specificity, most specific wins:

```text
location + service area
location + service type
location
tenant / product default
```

- If two published routes have the same specificity for the same key, **reject the route version at publish**. Do not wait for waiter send.
- Send freezes the resolved route version and `route_completion_role` on each created line.
- A later route edit does not change already-sent work.
- One Ticket line may fan out to several stations when the frozen rule says so.
- Ambiguous or missing resolution is fail-closed. No silent “all kitchen”.
- The waiter never overrides the resolved station.

```text
route_completion_role:
- REQUIRED
- INFORMATIONAL
```

Kitchen work is typically `REQUIRED`. A pass-screen copy may be `INFORMATIONAL`. Only `REQUIRED` tasks block `ALL_DESTINATIONS`.

### 5. `dispatch_id`

All `ProductionInstruction`s from one Send share one `dispatch_id`. Retry with the same idempotency key returns the same dispatch.

Keep ADR 0012 names. Destination groups are `ProductionInstruction`s, not a second `ProductionItem` master, and not an ADR 0005 Production Order.

### 6. Location `production_readiness_mode`

This is a **location** setting inside the tenant, not a tenant-wide switch.

```text
production_readiness_mode:
- ALL_DESTINATIONS
- PER_DESTINATION
```

**`ALL_DESTINATIONS` is dispatch-scoped, not line-scoped.**

- Readiness is computed for the whole `dispatch_id`.
- The waiter is notified only when every `REQUIRED` task on every instruction in that dispatch is finished, or no longer blocking after a confirmed cancel.
- One slow `REQUIRED` task keeps the **entire dispatch** “in preparation”.

**`PER_DESTINATION`** — each station marks its own part ready and the waiter is notified immediately, for example “Drinks ready at the bar”, while the kitchen is still working.

In **both** modes every station keeps its own progress (ADR 0012 events). The setting only decides **when the waiter sees readiness and is notified**.

Example in one tenant:

```text
Restoran Vodice   ALL_DESTINATIONS
Beach bar Srima   PER_DESTINATION
Hotel Zagreb      ALL_DESTINATIONS
```

The current location value is **snapshotted onto the dispatch / ProductionInstructions at send**. A later change of the location setting must not change an already-sent order.

### 7. Cancel and replacement on KDS

ADR 0012 already locks cancel + replacement. This ADR locks the kitchen effect:

- A sent task is never deleted.
- A `CANCELLED` progress event is visible on the **same** station.
- The replacement gets a new instruction and a **new** `dispatch_id`.
- The old instruction stays as evidence.
- A `CANCELLED` task no longer blocks readiness, but the cancel must be station-acknowledged or at least reliably delivered. An undelivered cancel still blocks `ALL_DESTINATIONS`.

### 8. KDS read-model

```text
KDS does not read the current Ticket.
KDS displays ProductionInstruction snapshots created at send.
```

Wire protocol and screen chrome stay later.

### 9. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Send with one unrouted line → nothing is written; waiter sees that line.
- One Send with pizza, cola, coffee, cake → four instructions, one `dispatch_id`.
- Two equally specific published routes → route version rejected at publish.
- Route to an inactive or other-location station → send blocked; no main-kitchen fallback.
- Offline KDS after commit → `delivery_pending`; retry does not create a second dispatch.
- Duplicate KDS ack → status and waiter notification change once.
- `ALL_DESTINATIONS`: bar ready, kitchen still working → no waiter “ready” for the dispatch.
- `INFORMATIONAL` pass screen unfinished → does not block `ALL_DESTINATIONS`.
- Cancel after send → same-station `CANCELLED` event; replacement uses a new `dispatch_id`; old instruction remains.
- Undelivered cancel under `ALL_DESTINATIONS` → dispatch stays blocked.
- Location later switches readiness mode → already-sent dispatch keeps the frozen mode.

## Rejected alternatives

- KDS querying live Ticket lines.
- Renaming kitchen work to `ProductionOrder` (collides with ADR 0005).
- Waiter choosing stations or sending a subset of lines.
- Sending kitchen items while a bar item silently has no route.
- Treating `ALL_DESTINATIONS` as per-line readiness.
- Letting an `INFORMATIONAL` pass screen block dispatch readiness.
- Resolving equal-specificity routes only when the waiter sends.
- Rolling back a committed send because KDS is offline.
- Treating `delivery_pending` as “not sent”.
- Deleting a sent KDS task on cancel.
- Automatic fallback to “main kitchen”.
- A single product default with no location or service-type override.
- One tenant-wide readiness mode for every location.
- Changing an already-sent dispatch when the location later switches mode.
- Posting `SALE` or warehouse `PRODUCTION_*` on kitchen send.
- Amending ADR 0001–0011 or 0013 in this change.

## Consequences

### Positive

- The kitchen works from what was sent, not from a later catalog edit.
- A missing bar route cannot silently drop drinks while food prints.
- Beach bar and restaurant can notify waiters differently without sharing one tenant switch.
- Offline KDS cannot double-dispatch.
- Cancel stays visible on the station that received the original work.

### Negative

- Operators cannot send drinks first while food stays unsent on purpose.
- A location must publish non-overlapping routes before service.
- `ALL_DESTINATIONS` waits for the slowest `REQUIRED` station in the dispatch.
- An undelivered cancel keeps the dispatch blocked.

### Neutral

- Documentation can merge without a KDS protocol or printer driver.
- Course / fire / hold remain later hooks.
- ADR 0012 instruction content and progress events stay unchanged.

## Invariants

1. Ticket ≠ ProductionInstruction ≠ ADR 0005 Production Order ≠ KDS device. Kitchen send does not post `SALE` or `PRODUCTION_*`.
2. KDS reads sent instruction snapshots, not the live Ticket, catalog, recipe, or current route.
3. Waiter Send covers all current `not_sent_qty`. No destination picker. No line picker. Classify all, then persist per-station instructions linked by `dispatch_id`, or persist nothing.
4. Send commit is durable write. Delivery is after commit. `delivery_pending` is not “not sent”. Offline KDS and retry must not create a second dispatch. Device ack and progress events are idempotent.
5. Route precedence is `location + service area`, then `location + service type`, then `location`, then tenant/product default. Equal specificity is rejected at publish. Resolved route version and `REQUIRED` / `INFORMATIONAL` are frozen at send.
6. A valid station is active and belongs to the same tenant and location. No main-kitchen fallback. Deactivation does not rewrite old instructions.
7. `ALL_DESTINATIONS` readiness is the whole `dispatch_id`. Only unfinished `REQUIRED` tasks, including undelivered cancels, block it. `PER_DESTINATION` notifies per station. Both modes keep per-station progress. The location mode is frozen at send.
8. A sent task is never deleted. Cancel is visible on the same station. Replacement is a new instruction and a new `dispatch_id`.
9. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Reservations, Waitlist and Guest Seating
Courses / fire / hold
POS layout
```

The next domain ADR should define **Reservations, Waitlist and Guest Seating**. Do not implement booking from this ADR. KDS wire protocol stays an adapter behind these semantics.

## See also

- [ADR 0005: Recipes and Production](0005-recipes-and-production.md)
- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)
- [ADR 0013: Tables, Service Areas and Seating](0013-tables-service-areas-and-seating.md)

## Out of scope

This ADR does not define:

- KDS wire protocol or printer drivers
- course / fire / hold UX
- expo choreography beyond `INFORMATIONAL` routes
- table reservation or waitlist
- POS screen layout
- staff identity and operator authorization
- offline POS synchronization
- payments, tax, or fiscal XML

## Amendment — 2026-08-15: Offline send is QUEUED_LOCALLY

The original Decision that Waiter Send is all-or-nothing, instructions share one `dispatch_id`, and a retry with the same idempotency key returns the same dispatch, remain in the original text.

ADR 0020 may queue a send while disconnected. That state is `QUEUED_LOCALLY`. It is not kitchen receipt. `DELIVERED_TO_STATION` and `ACKNOWLEDGED_BY_STATION` stay this ADR. A local printer is not proof that the server or KDS received the send.

This amendment does not change destination resolution, fail-closed routing, or instruction immutability.
