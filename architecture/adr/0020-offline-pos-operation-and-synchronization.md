# ADR 0020: Offline POS Operation and Synchronization

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize an offline POS product, a local edge server, or POS application code.

## Date

2026-08-15

## Context

ADR 0019 owns device identity, credentials, `REASSIGNING`, and `EffectiveDeviceConfig`. ADR 0018 owns cash units and business-day close. ADR 0014 owns kitchen send and `dispatch_id`. ADR 0011 owns Payment and `UNKNOWN`. ADR 0010 owns invoice numbers and fiscalization. ADR 0017 owns who may act.

Without this ADR, a disconnected till would become a second backend, two handhelds would edit the same Ticket, a waiter would see “sent to kitchen” when the kitchen never received it, local cash would look fiscalized, and a found device would rewrite a closed day.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks offline authority **before** POS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
The central server is the only canonical authority.
Devices work offline only inside pre-issued, signed,
time-boxed leases.
```

```text
Server state
  → last confirmed snapshot
POS device
  → local commands
Encrypted outbox
  → reconnect
Server validation and replay
  → ACCEPTED | REJECTED | CONFLICT | MANUAL_REVIEW | SECURITY_QUARANTINE
```

v1: no local edge server, no peer-to-peer, no CRDT, no generic merge.

## Decision

### 1. OfflineLease — one per device

A registered device does **not** go offline by default. While online it receives a signed lease bound to tenant, location, `device_id`, `assignment_generation`, `credential_generation`, `effective_config_id`, `business_day_id`, `allowed_command_types`, validity, and optional limits.

Every new lease also carries server-confirmed:

```text
sequence_start
previous_command_hash
lease_generation
```

- At most **one active lease** per device and business day. Two overlapping leases for the same device are rejected.
- Renewal continues the **same** sequence and hash chain. Sequence **never** resets on the same `device_id`.
- A new lease is not issued if the previous lease still has an unresolved outbox.
- Reinstall gets a new `device_id` (ADR 0019), not a new sequence on the old identity.
- The server permanently stores the last accepted sequence and hash.
- Short-lived. The device cannot extend a lease locally.
- Revoke of future offline work takes effect at the next online check.
- Without a valid lease the device is read-only or fully blocked, per location config.

```text
offline_operation_mode:
- DISABLED
- ORDERING_ONLY
- ORDERING_AND_CASH
- CONTROLLED_FULL
```

`CONTROLLED_FULL` is still an allow-list, not “everything works offline”.

### 2. Local store and OfflineCommand

One operator action is one local transaction: validate snapshot → create command → append outbox → update read-model → then show success. UI success without a durable outbox row is rejected.

The local DB is encrypted, bound to the registered device, clone/restore resistant, crash-safe. Purge only after a confirmed server checkpoint. Uninstall before sync can lose events and needs a warning or admin recovery.

```text
OfflineCommand
--------------
command_id
device_id
device_sequence
offline_lease_id
command_type
aggregate_id
base_version
payload
payload_hash
dependency_ids
operator_snapshot
business_day_id
effective_config_id
offline_data_snapshot hash / versions
device_created_at
previous_command_hash
device_signature
```

`command_id` is a device-born UUID/ULID, globally unique, **never rewritten** on sync. `device_sequence` is strictly increasing.

Server sequence rules: duplicate → same result; next → process; gap → stop; older unknown → security anomaly; same id, different payload → compromise.

Hash chain plus device signature. Both required.

If Create Ticket fails, later add-line / send / cash must not become an unrelated sale. Dependents become `BLOCKED_BY_DEPENDENCY`.

```text
PENDING | SENDING | ACCEPTED | DUPLICATE | REJECTED
CONFLICT | MANUAL_REVIEW | BLOCKED_BY_DEPENDENCY | SECURITY_QUARANTINE
```

No last-write-wins.

### 3. Server-side fencing

`OfflineAggregateLease` and `OfflineTableLease` are **server fences**, not only client courtesy. While a lease is valid, the server rejects conflicting **online** commands from other devices.

Each lease has a `fencing_token` / generation. After generation 8 is issued, generation 7 is dead.

Ownership moves only after: sync + release, lease expiry, or an audited **forced takeover**. Late commands from the previous owner go to `MANUAL_REVIEW`.

An online device must not seat a table reserved by a valid offline table lease. A Ticket created offline belongs to its creator until first sync. Without a table lease, offline seating is off.

### 4. OfflineDataSnapshot

`EffectiveDeviceConfig` is not enough. The server also issues a signed business snapshot:

```text
OfflineDataSnapshot
-------------------
catalog_version
sale_action_versions
price_list_version
tax_rule_version
modifier_version
production_route_version
table_map_version
issued_at
valid_until
canonical_hash
server_signature
```

The device must not invent a price, tax, sale action, or production route. On sync the server checks the command against the signed snapshot that was valid at `device_created_at`. A price not in that snapshot is rejected.

### 5. SyncSession

Do not pull canonical changes and then accept commands against a moving server.

1. Open `SyncSession`
2. Server freezes / marks a canonical high-water
3. Device sends the command chain
4. Server processes against that canonical state and fencing tokens
5. Return results and changes up to the **new** high-water
6. Device atomically installs the new canonical snapshot
7. Re-apply the still-unconfirmed local overlay
8. Confirm checkpoint
9. Only then purge the confirmed outbox prefix

Any interrupt must be safe to retry. Full resync **must not** delete the unconfirmed outbox. It rebuilds the canonical base and reapplies the pending overlay.

Cursor: tenant + location scoped, opaque, monotonic, bound to device sync generation. Must not read another tenant or location. Stale/unsupported cursor → controlled full resync, no guessing.

### 6. Cash versus fiscal and POSTED

Receiving cash is **not** proof that an invoice was issued or fiscalized.

- A Ticket must not become fully `POSTED` from a local cash movement alone.
- Offline invoice finalization is allowed only if ADR 0010 has a pre-reserved number/block **and** an allowed offline fiscal flow.
- Without that, the cash payment is clearly marked `PENDING_DOCUMENT` / `PENDING_FISCALIZATION`.
- The guest must not be shown a successful fiscalization that did not happen.

Offline cash only in `ORDERING_AND_CASH` or `CONTROLLED_FULL`, with operator offline auth, an open ADR 0018 drawer/wallet session, same location, not in close, global ids and idempotency. Payment and movement are written together.

Card: the POS app must not declare a card paid. Offline card only if the terminal independently authorizes, returns a stable provider reference, and a verifiable final result. Otherwise `UNKNOWN`; later provider result binds the **same** Payment.

### 7. Production

ADR 0014 send rules stay: whole Ticket, one `dispatch_id`, fail-closed routing, immutable instructions, idempotent delivery.

```text
QUEUED_LOCALLY
DELIVERED_TO_STATION
ACKNOWLEDGED_BY_STATION
```

A local printer is not proof that the server or KDS received the send. Offline send must not show “sent to kitchen” as if the kitchen acknowledged it.

### 8. Approvals — safer v1

v1: offline allow only discounts or comps the operator may **self-apply** inside the signed snapshot. Any action that requires a second approver, maker-checker, or emergency override is **rejected** offline.

`OfflineApprovalBudget` is the locked shape **before** a later amendment may enable second-person offline approve. A budget is reserved server-side, exclusive to one device, consumed in the local transaction, one-time nonce and exact payload hash, re-checked on sync. Until such a budget is issued, the action stays blocked.

Forbidden offline: emergency override, role / permission / device / config change.

### 9. OfflineAuthorization

PIN cache only for people recently authenticated on **this** device and location. Encrypted, bound to the device key, `MembershipEpisode`, `authorization_generation`, short-lived, limited to the signed permission snapshot; wiped on expiry, location change, or confirmed revoke.

```text
OfflineAuthorization
--------------------
staff_membership_id
membership_episode_id
authorization_generation
device_id
location_id
permissions
monetary_limits
valid_until
issued_at
```

The device cannot widen permissions or limits. On reconnect the server checks the command was inside that then-valid scope. A later suspension does not silently delete a lawful historical command; policy may send it to `MANUAL_REVIEW`.

Clock: server-issued validity, last known server time, monotonic device clock if present, max skew, block new commands on clock rollback. After lease expiry: display and sync only; no new business commands.

### 10. Business-day close fence

Heartbeat is not proof the outbox is empty.

```text
OPEN → OFFLINE_DRAIN → CLOSING → CLOSED
```

`OFFLINE_DRAIN`: no new leases for that day; devices get a closing fence; each device submits a signed sequence high-water; close waits for lease return or proven expiry; financial `PENDING` / `UNKNOWN` / `CONFLICT` / `MANUAL_REVIEW` / `SECURITY_QUARANTINE` and sequence gaps block close.

A lost-device close exception (`offline.drain_exception`) after lease expiry **seals** the missing range:

```text
device_id
lease_id
last_accepted_sequence
declared / assumed high-water
missing sequence range
closed_business_day_id
approved_by
reason
```

Commands that later arrive from that sealed range must **not** mutate the closed day. They go to `MANUAL_REVIEW`. A financial correction is a late adjustment on the **current OPEN** day, with a reference to the original closed day.

### 11. Compromised credential

Already `ACCEPTED` commands stay valid. An idempotent retry of an accepted command returns the same result.

Every **previously unknown** command from a `COMPROMISED` credential generation goes to `SECURITY_QUARANTINE`. It is not applied automatically even if signature and lease look valid. Manual recovery decides compensation. Sync does **not** continue past the suspicious command.

### 12. Conflicts and recovery

No universal merge.

- Ticket change without a live fencing token → `CONFLICT` / reject
- Table taken against a live table fence → `REJECTED`
- Duplicate cash → existing result
- Cash after closing fence → `MANUAL_REVIEW`
- Same `dispatch_id` → duplicate, not a new instruction
- Add-line keeps the frozen snapshot price if the signed data snapshot allowed it
- Role revoked after a valid command → accept or review per signed policy; do not rewrite the author

Operator UI must distinguish `OFFLINE` / `PENDING SYNC` / `SYNCED` / `CONFLICT` / `ACTION REQUIRED`. A rejected financial command is not deleted; it opens a recovery task.

### 13. Permissions

ADR 0017 owns the catalog. This ADR adds:

```text
offline.lease_manage
offline.drain_exception
```

Forced takeover uses `offline.lease_manage` (or `device.security_manage` when the device is compromised).

### 14. Audit

For every offline command keep at least: the original signed payload; device and credential generation; lease and permission snapshot; `OfflineDataSnapshot` hash; device sequence and hash chain; operator and membership episode; device / config / app version; device and server time; sync attempts; server decision and reason; conflict or manual resolution; any compensation; sealed-range reference if the command arrived after a close exception.

### 15. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- No valid lease → no new business command. The device cannot extend a lease locally.
- A second overlapping lease for the same device is rejected.
- Lease renewal continues the same sequence and hash anchor; sequence does not reset on the same `device_id`.
- A new lease while the previous outbox is unresolved is rejected.
- UI success without a durable outbox row is impossible / rejected.
- Duplicate `command_id` / sequence returns the same result; a sequence gap stops processing; same id + different payload is a security stop.
- A broken hash chain stops sync; the chain is not skipped.
- A dependent cash command after a rejected Create Ticket is `BLOCKED_BY_DEPENDENCY`, not a new sale.
- An online device cannot mutate a Ticket or seat a table fenced by another device’s live lease.
- Forced takeover sends the previous owner’s late commands to `MANUAL_REVIEW`; the old fencing token is rejected.
- A command whose price / tax / route is not in the signed `OfflineDataSnapshot` is rejected.
- Full resync does not delete the pending outbox; the overlay is reapplied.
- A server mutation during sync does not skip the frozen high-water.
- Offline cash without a valid ADR 0010 offline fiscal flow does not become fully `POSTED`; guest UI does not show fiscal success.
- Offline send shows `QUEUED_LOCALLY`, not kitchen acknowledged.
- App-declared card capture without a terminal reference stays `UNKNOWN`; a later result updates the same Payment.
- Second-approver / emergency override offline is rejected in v1.
- The same `OfflineApprovalBudget` cannot be consumed on two devices.
- A PIN cache for a never-seen-on-this-device employee is rejected.
- After lease expiry, sync of already-stored commands is allowed; new commands are not.
- Interrupted sync is safe to retry; the outbox is not purged before checkpoint ack.
- A stale cursor triggers full resync, not guessed gaps.
- `Close business day` from `OPEN` without `OFFLINE_DRAIN` is rejected.
- A found device after a closing exception does not mutate the closed day; sealed-range commands go to `MANUAL_REVIEW` / late adjustment.
- After `COMPROMISED`, accepted commands stay; a new or unknown command from that credential generation goes to `SECURITY_QUARANTINE`; the chain does not continue past it.
- A rejected financial command opens recovery and is not deleted.
- A cursor from another tenant or location is rejected.

## Rejected alternatives

- An authoritative local database.
- Last-write-wins.
- Overlapping leases on one device.
- Resetting sequence on the same `device_id`.
- Client-only Ticket or table leases while online devices can still write.
- Inventing prices outside `OfflineDataSnapshot`.
- Pull-then-push sync against a moving high-water.
- Full resync that wipes the pending outbox.
- `POSTED` or guest-visible fiscalization from local cash alone.
- A close exception that later lets a found device rewrite the closed day.
- The same approval budget on two devices.
- Auto-applying unknown commands after `COMPROMISED`.
- UI success without an outbox row.
- Purge before checkpoint.
- An unbounded PIN cache.
- A device-extended lease.
- Cash or day close with active leases or an unproven outbox.
- Heartbeat as empty-outbox proof.
- App-declared card capture.
- Deleting a rejected financial command.
- Continuing sync after a broken hash chain.
- CRDT, peer-to-peer, or a local edge server in v1.
- Amending ADR 0001–0009, 0012–0013, or 0015–0016 in this change.

## Consequences

### Positive

- A disconnected till cannot become a second source of truth.
- Two devices cannot silently fight over one Ticket or one table.
- A found device cannot rewrite a closed business day.
- A compromised credential cannot auto-apply unknown commands.

### Negative

- Offline is narrower than “the restaurant keeps working as usual”.
- v1 cannot maker-check or emergency-override while disconnected.
- Day close waits for drain, lease expiry, or an audited sealed-range exception.

### Neutral

- Documentation can merge without an offline POS build or a crypto library.
- A later amendment may issue `OfflineApprovalBudget`; v1 does not.
- A local edge server would need its own ADR.

## Invariants

1. The server is the only canonical authority. The device is not a second backend.
2. At most one active `OfflineLease` per device and business day. Sequence never resets on the same `device_id`. Renewal continues the hash chain.
3. Aggregate and table leases are server fences with a `fencing_token`. Online writers are blocked. Forced takeover reviews late commands from the previous owner.
4. Every command is signed, hash-chained, and checked against `OfflineLease`, `OfflineAuthorization`, and `OfflineDataSnapshot`. The device does not invent price, tax, sale action, or route.
5. `SyncSession` freezes one high-water. Full resync keeps the pending outbox. Purge only after checkpoint.
6. Local cash is not `POSTED` and not fiscal success unless ADR 0010 has a reserved offline fiscal flow. Card capture is not declared by the POS app.
7. Offline send is `QUEUED_LOCALLY` until the station is delivered or acknowledged. Same `dispatch_id` is a duplicate.
8. v1 forbids second-approver and emergency override offline. An approval budget, if later issued, is exclusive to one device.
9. `OFFLINE_DRAIN` precedes day close. A lost-device exception seals the missing range. Later commands do not mutate the closed snapshot.
10. After `COMPROMISED`, accepted commands stay; unknown commands from that generation are `SECURITY_QUARANTINE`. The chain does not continue past them.
11. Tenant isolation. Ids alone do not authorize. No last-write-wins.

## Follow-up ADRs

```text
Customer Profiles, Consent and Loyalty
```

Do not implement a local edge server, CRDT, or a payment-terminal offline protocol from this ADR.

## See also

- [ADR 0010: Invoices and Fiscalization](0010-invoices-and-fiscalization.md)
- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0014: Kitchen, Bar Production Routing and KDS](0014-kitchen-bar-production-routing-and-kds.md)
- [ADR 0017: Staff Identity, Roles and Operator Authorization](0017-staff-identity-roles-and-operator-authorization.md)
- [ADR 0018: Shifts, Cash Drawers and Daily Closing](0018-shifts-cash-drawers-and-daily-closing.md)
- [ADR 0019: POS Devices, Registration and Configuration](0019-pos-devices-registration-and-configuration.md)

## Out of scope

This ADR does not define:

- peer-to-peer CRDT or a full on-site edge server
- accounting posting
- a concrete payment-terminal offline protocol
- a concrete crypto library or local DB engine
- conflict-center UX chrome
- exact lease TTL, PIN-cache TTL, or clock-skew threshold
- POS screen layout

## Amendment — 2026-08-15: Offline loyalty owned by ADR 0021

The original Decision that the server is the only canonical authority, and that the device writes signed commands rather than a second backend, remain in the original text.

ADR 0021 owns the loyalty ledger. An offline device may show the last confirmed balance as informational. Cached or locally estimated points must be visually distinct from the server-confirmed balance.

The device writes a signed `AccrueLoyaltyEarn` command. Local UI is `PENDING_SYNC` only. The device must not create a canonical `LoyaltyLedgerEntry`. After server accept, the server creates `EARN_PENDING`. A rejected offline sale creates no points.

v1 forbids offline redemption from a cached balance. Offline redemption would need an exclusive server-issued budget or lease and a valid `LoyaltyRedemptionAuthorization`. v1 issues neither.

This amendment does not change `OfflineLease`, fencing, `SyncSession`, or cash-versus-fiscal rules.

## Amendment — 2026-08-15: Channel webhooks are server-only

The original Decision that the server is the only canonical authority remain in the original text.

ADR 0022 owns provider inbox and outbox. Webhooks land only on the central server. An offline device must not ingest a provider webhook as canonical, must not accept a `ChannelOrder` the server already accepted, and must not emit a provider acknowledgement. After sync it receives the accepted Ticket or `ChannelOrder` snapshot. Local print is not platform acknowledgement.

This amendment does not change `OfflineLease`, fencing, `SyncSession`, or cash-versus-fiscal rules.

## Amendment — 2026-08-15: Incoming eInvoice and recipient fiscalization are server-only

The original Decision that the server is the only canonical authority remain in the original text.

ADR 0024 owns inbound eInvoice receive and recipient fiscalization. An offline POS device must not receive a canonical eInvoice, send recipient fiscalization, confirm manual evidence, or store intermediary credentials. After sync it may show an allowed AP status only.

This amendment does not change `OfflineLease`, fencing, `SyncSession`, or cash-versus-fiscal rules.

## Amendment — 2026-08-15: Accounting export is server-only

The original Decision that the server is the only canonical authority remain in the original text.

ADR 0025 owns accounting posting and export. An offline POS device must not prepare, download, or submit an accounting batch, and must not store racunai.hr credentials. Offline cash is not an export source until the server has accepted it.

This amendment does not change `OfflineLease`, fencing, `SyncSession`, or cash-versus-fiscal rules.

## Amendment — 2026-08-16: Late offline accept is an analytics restatement source

The original Decision that the server is the only canonical authority remain in the original text.

ADR 0026 owns analytics projections and snapshots. A late server accept of an offline command is a restatement source. It must not mutate a `PUBLISHED` `AS_RECORDED` snapshot. A POS device must not publish or restate analytics snapshots.

This amendment does not change `OfflineLease`, fencing, `SyncSession`, or cash-versus-fiscal rules.

## Amendment — 2026-08-16: Offline recovery must honor privacy fence and legal hold

The original Decision that the server is the only canonical authority remain in the original text.

ADR 0027 owns audit, retention, legal hold, and privacy execution. Emergency override and offline recovery must not bypass tenant isolation, a legal hold or retention stop, or `PrivacySubjectFence`. A late offline event after `ERASED` is quarantined and must not rematerialize PII.

This amendment does not change `OfflineLease`, fencing, `SyncSession`, or cash-versus-fiscal rules.
