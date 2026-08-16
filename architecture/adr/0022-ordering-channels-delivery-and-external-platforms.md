# ADR 0022: Ordering Channels, Delivery and External Platforms

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a Wolt or Glovo adapter, a campaign engine, or POS application code.

## Date

2026-08-15

## Context

ADR 0001 isolates tenant data. ADR 0011 owns Payment and Settlement. ADR 0012 owns the Ticket. ADR 0014 owns kitchen send. ADR 0016 owns the internal price list and TicketLine snapshot. ADR 0017 owns the permission catalog. ADR 0020 makes the server the only canonical offline authority. ADR 0021 owns `CustomerProfile` and consent.

Without this ADR, a Wolt payload would become the Ticket, a webhook retry would open a second sale, a platform “paid” claim would look like bank settlement, commission would look like a discount, and a cancel webhook would delete kitchen work.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the external-ordering domain **before** channel adapters. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
OrderingChannel      = where the order came from
ExternalConnection   = authenticated link to a platform
ChannelOrder         = canonical received-order record
Ticket               = internal order and financial workflow
DeliveryFulfillment  = physical delivery or pickup
PlatformSettlement   = what the platform owes the restaurant
```

```text
External payload ≠ Ticket
ChannelOrder     ≠ Ticket
Ticket READY     ≠ delivered
Platform payment claim ≠ bank settlement
Platform commission ≠ TicketLine discount
Local accept ≠ provider acknowledgement
```

Wolt, Glovo, or a future platform is an adapter, not a domain model. Every adapter implements the same `ChannelOrder` contract.

## Decision

### 1. ChannelOrder first

Every **external** order becomes a `ChannelOrder` first. An external payload never directly creates or stands in for an internal Ticket.

Walk-in POS and staff-entered tickets may stamp `OrderingChannel` type `POS` or `PHONE` **without** a `ChannelOrder`. Inbound orders that arrive through an `ExternalConnection` (WEB, KIOSK, API, `DELIVERY_PLATFORM`, guest QR app, phone-ordering platform) **must** become `ChannelOrder` first.

```text
POS
PHONE
QR_TABLE
WEB
KIOSK
DELIVERY_PLATFORM
API
```

### 2. Anti-corruption adapter

The adapter verifies authenticity, stores the raw payload, normalizes, maps external ids, turns the result into canonical commands, and translates canonical statuses back. Internal domain statuses are not provider-specific names.

Tenant and location come from the **authenticated** `ExternalConnection`. A `tenant_id` in the webhook body is ignored. ADR 0001 is not amended. This ADR owns the inbound-auth lock.

### 3. OrderingChannel and ExternalConnection

```text
OrderingChannel
---------------
tenant_id
location_id
type
name
status
acceptance_mode
service_type
```

```text
ACTIVE
PAUSED
DEGRADED
DISABLED
```

`PAUSED`: no new desired accepts; in-flight orders and statuses must still complete. `DISABLED` does not erase history or audit credentials.

```text
ExternalConnection
------------------
tenant_id
location_id
provider
external_merchant_id
status
credential_generation
webhook_secret_generation
configuration_version
```

```text
PENDING
ACTIVE
SUSPENDED
BROKEN
REVOKED
```

One external merchant id must not silently bind two locations. Credentials are encrypted, rotatable, and stored apart from business payloads. Access tokens and webhook secrets must not live in the raw business payload.

### 4. Inbox

```text
ExternalInboxEvent
------------------
connection_id
provider_event_id
event_type
received_at
provider_occurred_at
raw_payload
payload_hash
signature_result
processing_status
```

```text
RECEIVED
VERIFIED
PROCESSED
DUPLICATE
REJECTED
QUARANTINED
```

Invalid signature never enters the business domain. Same event replay is not a new order. Same event id + different payload → `QUARANTINED`. Raw payload is evidence, not a live Ticket. Provider retry returns an idempotent result. Processing is restart-safe.

If a provider has no stable event id, the adapter derives a provider-specific deterministic dedupe fingerprint from the authenticated connection, event class, external order id/version, and canonical body hash. The fallback key is versioned and auditable. A random id per delivery is forbidden.

### 5. ChannelOrder and accept

```text
ChannelOrder
------------
tenant_id
location_id
channel_id
external_order_id
status
service_type
requested_at
promised_at
customer_snapshot
delivery_snapshot
monetary_snapshot
source_payload_hash
version
```

```text
RECEIVED
VALIDATING
AWAITING_ACCEPTANCE
ACCEPTED
REJECTED
CANCEL_REQUESTED
CANCELLED
COMPLETED
MANUAL_REVIEW
```

```text
UNIQUE (external_connection_id, external_order_id)
```

A repeated webhook updates the **same** `ChannelOrder` under explicit rules. It does not create a second Ticket.

Accept is one transaction:

1. Lock `ChannelOrder`.
2. Confirm it is not already accepted.
3. Re-validate location and channel.
4. Map every item and required modifier.
5. Resolve prices, tax, and routing.
6. Create one Ticket.
7. Create `ExternalOrderLink`.
8. Mark `ACCEPTED`.
9. Write the outbound acknowledgement to the outbox.

```text
One ChannelOrder → at most one Ticket
```

Retry accept returns the same Ticket. Any validation failure creates **no** partial Ticket.

`acceptance_mode`: `MANUAL` → `AWAITING_ACCEPTANCE`. `AUTOMATIC` accepts only if every check passes. Missing mapping, invalid price, paused channel, closed location, no routing station, outside delivery zone, or an unreliable payment result → `REJECTED` or `MANUAL_REVIEW` per config. Both modes use the same backend workflow.

### 6. Order fence and provider confirmation

Inbox processing and acceptance for the same `(external_connection_id, external_order_id)` use one deterministic **order fence**. Accept must re-check the latest processed provider revision under that fence. A concurrent or already-observed provider cancel wins over a stale accept. The system must not create a Ticket after the external order was cancelled or its acceptance deadline elapsed.

```text
local_acceptance_status:
- NOT_ACCEPTED
- ACCEPTED_LOCAL

provider_acceptance_status:
- NOT_SENT
- PENDING
- ACKNOWLEDGED
- REJECTED
- UNKNOWN
```

`ACCEPTED_LOCAL` means the local transaction created the one Ticket and outbox row. It does **not** claim that the provider received or accepted the acknowledgement.

```text
provider_confirmation_mode:
- LOCAL_ACCEPT_IS_AUTHORITATIVE
- REQUIRE_PROVIDER_ACK_BEFORE_PRODUCTION
```

In `REQUIRE_PROVIDER_ACK_BEFORE_PRODUCTION`, the Ticket may exist but kitchen send is blocked until provider acknowledgement. In `LOCAL_ACCEPT_IS_AUTHORITATIVE`, production may start after the local commit; a later provider rejection or cancel uses the explicit cancellation and compensation workflow. The UI must show provider acceptance `PENDING` / `UNKNOWN` and must not collapse it into local `ACCEPTED`.

### 7. Mapping, price, and money

```text
ExternalItemMapping
-------------------
connection_id
external_item_id
sale_action_id
modifier_mapping_version
external_menu_revision
mapping_version
valid_from
valid_until
```

Mapping binds `sale_action_id` (product, unit, stock, tax base, price, allowed modifiers). No unique mapping → fail-closed. No silent miscellaneous item. Menu publish and availability stay ADR 0029.

An order is interpreted using the provider menu revision / mapping generation that produced it, not whichever mapping is current when a delayed webhook is processed. Accept snapshots external item and modifier ids, external menu revision, mapping version, resolved `sale_action_id`, and price/rule versions onto the created Ticket lines. If the provider supplies no usable revision and old and new mappings are ambiguous, the order goes to `MANUAL_REVIEW`. It is never reinterpreted silently.

The Ticket stores, separately: external displayed price; canonical resolved price; platform discount; merchant-funded discount; platform-funded subsidy; delivery fee; service fee; tip; tax result; source and rule of each amount. Do not net into one “platform total”.

ADR 0016 still owns the internal list, discount, Comp, and final TicketLine snapshot.

```text
price_mismatch_policy:
- REJECT
- MANUAL_REVIEW
- ACCEPT_WITHIN_TOLERANCE
```

Tolerance is versioned and audited. No implicit accept of any platform price.

All monetary values use canonical decimal strings and one order currency. The normalized component equation must reconcile exactly:

```text
items
+ customer-facing fees
+ tip
- customer-facing discounts
= customer total
```

Merchant-funded discounts, platform-funded subsidies, commission, and settlement fees remain separately identified. They must not be inserted into that equation as if they were the same economic component. No float, exponent form, implicit rounding, negative customer total, or silent remainder. A tolerated mismatch is stored as an explicit audited adjustment or allocation. It is not hidden by replacing the provider total.

### 8. Payment and commission

```text
PAY_ON_DELIVERY      → POS collects per ADR 0011
MERCHANT_COLLECTED   → merchant terminal / own online Payment
PLATFORM_COLLECTED   → record claim; later reconciliation confirms payout
```

A platform claim is not bank settlement. Record external payment reference, amount, currency, provider status, payer/collector, provider event, and later settlement reference. The same order must not be marked paid twice (platform claim **and** local capture).

Provider cancellation and refund assertions are also **external financial claims**, not proof that a local Refund, reversal, or bank payout occurred. They carry their own unique provider reference and lifecycle. Replayed claims bind the same internal financial operation. They must not create a second refund. A later settlement or reconciliation result resolves the claim without rewriting the original Ticket, Payment, or Invoice.

Platform commission, marketing fee, or logistics fee is **not** a product price cut, TicketLine discount, Comp, or cash variance. It belongs to settlement and ADR 0025.

### 9. Delivery, address, promise, and raw PII

`Ticket READY` is not delivered.

```text
DeliveryFulfillment
-------------------
channel_order_id
mode
address_snapshot
promised_at
courier_reference
status
version
```

```text
MERCHANT_DELIVERY
PLATFORM_DELIVERY
PICKUP
```

Courier modes:

```text
NOT_REQUIRED
UNASSIGNED
ASSIGNED
COURIER_ARRIVED
PICKED_UP
DELIVERED
FAILED
CANCELLED
```

`PICKUP` must not invent a courier.

```text
AWAITING_COLLECTION
COLLECTED
FAILED
CANCELLED
```

Ticket, production, payment, and delivery statuses stay separate.

Address is a snapshot on the order. A later `CustomerProfile` edit does not change it. Delivery-zone rules are versioned (area, min amount, fee, ETA, max distance, service period). External coordinates are not blindly trusted. Zone publish to channels may stay ADR 0029.

Address and contact snapshots and raw provider payloads are classified personal data. The raw payload is encrypted separately, access-controlled, and retained only under the ADR 0027 retention and legal-hold policy. Audit may retain event id, canonical hash, signature result, normalized decision, and minimal evidence after the raw body expires. Raw payload is not an indefinite audit substitute and must not be copied into ordinary application logs, dead-letter messages, or error traces.

Distinguish `requested_ready_at`, `accepted_ready_at`, `actual_ready_at`, `picked_up_at`, and `delivered_at`. Accept freezes the promised time and prep-rule version. A later global prep-time change does not rewrite an accepted order. A restaurant time shift is an audited update to the platform. The original value is not silently overwritten.

### 10. Cancel, change, and field ownership

External `cancel` is not Ticket delete.

- Before accept: `ChannelOrder` → `CANCELLED`; no Ticket.
- After accept, before production: controlled Ticket cancel per ADR 0012.
- After kitchen send: ProductionInstruction stays; visible cancel per ADR 0014; stock and finance use compensation; refund is separate; platform reason stays in audit.
- After fiscalization or capture: no mutation of the original document; reversal, refund, or credit.

The provider must not silently replace the whole payload after accept. Each change is a typed event: `ADD_ITEM`, `REMOVE_ITEM`, `CHANGE_QUANTITY`, `CHANGE_PROMISE`, `CHANGE_ADDRESS`, `CANCEL`.

- Before accept: may update `ChannelOrder`.
- After accept: canonical Ticket command.
- After production or fiscal freeze: cancel/replacement or compensation.
- A stale event must not overwrite a newer decision. No last-write-wins.

Field ownership:

- External order id: platform before accept; immutable after.
- Items: platform before accept; Ticket workflow after.
- Internal price: backend; frozen snapshot after accept.
- Customer snapshot: channel; frozen after accept.
- Prep time: negotiated; accepted snapshot after accept.
- Production status: Tablio always.
- Courier status: delivery provider / fulfillment lifecycle.
- Fiscal status: Tablio always.
- Settlement status: reconciliation always.

Ordering of inbound events uses provider event id, provider order version when present, provider occurred time, server received time, local `ChannelOrder.version`, and an explicit transition matrix. Server received time alone is **not** the business clock.

### 11. Outbox, pause, and retry

Local commit and platform acknowledgement are separate.

```text
ExternalOutboxMessage
---------------------
connection_id
channel_order_id
message_type
payload
idempotency_key
attempt_count
next_attempt_at
status
```

```text
PENDING
SENDING
ACKNOWLEDGED
RETRY
DEAD_LETTER
```

If the platform is down after the Ticket exists: the Ticket stays; no second Ticket; the same acknowledgement retries; the operator sees that the platform is not confirmed; permanent failure opens a recovery task. No outbound HTTP inside the accept database transaction.

An outbound timeout is `UNKNOWN`, not proof of failure. If the provider supports idempotency, retries reuse the same provider idempotency key. If it does not, the adapter must query or reconcile provider state before repeating a non-idempotent accept, cancel, or refund call. Blind retry of a non-idempotent provider operation is rejected. `DEAD_LETTER` preserves the original intent and opens a recovery task. Manually resolving it must not mint a new logical operation id.

Pause is a local decision. Provider confirm is a side effect.

```text
desired_channel_status
provider_acknowledged_status
```

Until the provider acknowledges pause, a new webhook must still be processed. It must not be dropped because the UI says `PAUSED`. Emergency stop-accepting is audited and has a per-connection result.

### 12. Customer snapshot

External customer payload is a snapshot. It must not auto-create a `CustomerProfile`, merge by phone, write marketing consent, touch loyalty, or lift suppression. Explicit profile create stays ADR 0021. Fulfillment data is not a marketing license.

### 13. First-party WEB / QR / KIOSK context

A first-party guest channel still needs an authenticated, expiring channel context. It must not trust client-sent tenant, location, table, or session ids.

```text
ChannelContextToken
-------------------
tenant_id
location_id
channel_id
service_type
optional service_table_id
optional seating_session_id
generation
expires_at
allowed_actions
```

For `QR_TABLE`, submit and accept re-check that the token generation still belongs to the current table or session context. A QR copied from an old party must not add items to a later `SeatingSession`. ADR 0013 still owns seating and is not amended.

For `KIOSK`, device trust remains ADR 0019. The channel token cannot widen device capabilities. ADR 0019 is not amended.

First-party guest orders still become `ChannelOrder` before Ticket acceptance.

### 14. Connection reassignment

An active `ExternalConnection` cannot be silently moved to another location. Reassignment enters `REASSIGNING`, stops new accepts, drains and verifies inbox and outbox high-waters, resolves `PENDING` / `UNKNOWN` acceptance and financial operations, rotates or rebinds credentials and config, then activates at the new location with a new generation. Historical `ChannelOrder`s remain bound to the old location. Failure leaves the connection in `REASSIGNING`. No mixed-location processing and no automatic rollback.

### 15. Offline and security

Webhooks land only on the central server. An offline POS device does not ingest a provider webhook as canonical, does not accept a `ChannelOrder` the server already accepted, and does not emit provider acknowledgements. After sync it receives the accepted Ticket or `ChannelOrder` snapshot. Local print or delivery is not platform acknowledgement. Ticket write leases stay ADR 0020.

Inbound: verify signature before normalize; replay protection; timestamp tolerance; body hash; secret generation; rate limit; payload size limit; schema validation; quarantine on same id / different payload.

Outbound: credential rotation; least-privilege scopes; PII redaction in logs; no access token in the business payload; dead-letter access only for authorized operators.

ADR 0028 later owns Tablio’s public API and tenant-facing webhooks. This ADR owns the **provider inbound inbox** and **provider outbound outbox** for ordering channels.

### 16. Permissions

ADR 0017 owns the catalog. This ADR adds:

```text
channel.view
channel.configure
channel.pause
channel.security_manage
channel.raw_payload_view
order.accept_external
order.reject_external
order.cancel_external
order.resolve_external
delivery.assign
delivery.update
delivery.override
```

`channel.view` does not grant credentials or raw payload.

### 17. Audit

Keep at least: raw inbound payload and hash; signature result; normalized order version; item mapping; external versus internal price delta; accept/reject actor and reason; Ticket link; cancel and refund path; delivery lifecycle; outbound messages and attempts; provider acknowledgements; connection, config, and credential generation; pause changes; manual resolution; who viewed the raw payload.

### 18. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- A valid inbound order creates a `ChannelOrder` and no Ticket until accept.
- Walk-in POS create Ticket without `ChannelOrder` still succeeds.
- `tenant_id` in the webhook body cannot select another tenant; tenant comes from `ExternalConnection`.
- Invalid signature stays out of the business domain; no `ChannelOrder`.
- Replay of the same `provider_event_id` is `DUPLICATE` and does not create a second order or Ticket.
- Same event id, different payload → `QUARANTINED`.
- Accept racing with a provider cancel or deadline under the same order fence does not create a Ticket after cancellation or expiry.
- Local accept creates one Ticket but does not claim provider acknowledgement; local and provider acceptance statuses remain separate.
- `REQUIRE_PROVIDER_ACK_BEFORE_PRODUCTION` blocks kitchen send until acknowledgement; `LOCAL_ACCEPT_IS_AUTHORITATIVE` uses compensation on a later provider reject.
- `UNIQUE (external_connection_id, external_order_id)` rejects a second order row.
- Accept retry returns the same Ticket; `ExternalOrderLink` is 1:1.
- Automatic accept with a missing item mapping creates no Ticket (`REJECTED` or `MANUAL_REVIEW`).
- Paused channel, closed location, no route, or outside zone fails automatic accept with no partial Ticket.
- Unmapped required modifier is fail-closed; no miscellaneous product.
- A delayed order referencing an old external menu revision uses its frozen mapping version; ambiguous revision goes to `MANUAL_REVIEW`.
- Platform total is stored un-netted; commission is not a TicketLine discount.
- Monetary components must reconcile exactly with decimal arithmetic; a tolerated mismatch is an explicit adjustment, not a hidden overwrite.
- Price outside versioned tolerance follows `price_mismatch_policy`; implicit accept is rejected.
- `PLATFORM_COLLECTED` claim does not mark bank settlement complete.
- Platform claim plus local capture on the same order is rejected.
- Replayed provider refund or cancel claim binds the same operation and does not create a second Refund.
- `Ticket READY` does not set fulfillment `DELIVERED`.
- `PICKUP` cannot be assigned a courier.
- Profile edit after accept does not change `DeliveryAddressSnapshot`.
- Global prep-time change does not rewrite `accepted_ready_at`.
- Cancel before accept leaves no Ticket.
- Cancel after send does not delete the ProductionInstruction; the ADR 0014 cancel event is visible.
- Cancel after fiscalization does not mutate the issued Invoice.
- Silent full-payload replace after accept is rejected; typed events only.
- Stale change event does not overwrite a newer `ChannelOrder` or Ticket version.
- Outbound HTTP inside the accept transaction is rejected / not modeled.
- Platform down after accept: Ticket remains; no second Ticket; the same outbox idempotency key retries.
- Outbound timeout is `UNKNOWN`; a non-idempotent provider call is reconciled before retry and is not blindly repeated.
- Desired `PAUSED` without provider acknowledgement still processes a new inbound webhook.
- Channel payload does not create a `CustomerProfile`, marketing consent, or loyalty touch.
- Expired QR or table token, or a token for an old `SeatingSession`, cannot submit an order for the current party.
- First-party guest order cannot select tenant, location, or table from client fields; context comes from `ChannelContextToken`.
- Raw payload expiry may remove the encrypted body while retaining minimal hash, signature, and decision evidence; raw PII is absent from ordinary logs.
- Connection reassignment with unresolved inbox, outbox, or payment state remains `REASSIGNING` and cannot process in two locations.
- Offline device cannot accept a `ChannelOrder` already accepted by the server.
- Offline device cannot emit a provider acknowledgement.
- `channel.view` cannot read raw payload or credentials.
- Access token or webhook secret in the business payload is rejected.

## Rejected alternatives

- A provider payload as the Ticket.
- One status for Ticket, payment, production, and delivery.
- A second Ticket on webhook retry.
- An unmapped item mapped to a miscellaneous product.
- A platform payment claim as settlement proof.
- A provider refund or cancel as an automatic local Refund.
- Commission as a product discount.
- Automatic `CustomerProfile` or marketing consent from the order.
- Last-write-wins on external edits.
- Deleting the Ticket on a cancel webhook.
- Assuming provider pause before acknowledgement.
- An outbound API call inside the accept database transaction.
- Offline POS as webhook authority.
- An access token or webhook secret in the business payload.
- `tenant_id` from the webhook body.
- Inventing a courier for `PICKUP`.
- Accept after a fenced cancel or elapsed deadline.
- Collapsing provider acknowledgement into local `ACCEPTED`.
- Reinterpreting a delayed order against the current mapping.
- Hiding a money remainder by overwriting the provider total.
- Blind retry of a non-idempotent outbound call.
- Client-sent tenant, location, or table on first-party channels.
- Silent connection move between locations.
- Indefinite raw PII as the audit record.
- Random per-delivery event ids.
- Amending ADR 0001, 0013, or 0019 in this change.

## Consequences

### Positive

- External platforms stay an integration source. Tablio keeps the canonical Ticket.
- A webhook retry cannot open a second sale.
- Local accept and provider acknowledgement cannot be confused.
- Commission and platform claims cannot rewrite price, Payment, or Invoice.

### Negative

- v1 kitchen send may wait for provider acknowledgement when that mode is on.
- Connection reassignment is blocked until inbox, outbox, and financial claims drain.
- First-party guest channels need a server-issued `ChannelContextToken`.

### Neutral

- Documentation can merge without a Wolt or Glovo adapter.
- Menu publish stays ADR 0029. Public API stays ADR 0028. Settlement posting stays ADR 0025.
- Seating stays ADR 0013. Kiosk device trust stays ADR 0019.

## Invariants

1. `ChannelOrder` ≠ Ticket ≠ `DeliveryFulfillment` ≠ Payment ≠ Settlement.
2. An external payload never creates a Ticket except through accept of one `ChannelOrder`.
3. `UNIQUE (external_connection_id, external_order_id)`. One `ChannelOrder` → at most one Ticket. Accept retry returns the same Ticket.
4. Tenant and location come from the authenticated connection or `ChannelContextToken`, never from a client-sent `tenant_id`.
5. Local accept and provider acknowledgement are separate. Cancel or deadline under the order fence wins over a stale accept.
6. Mapping is revision-bound. Money components reconcile exactly. Commission is not a discount.
7. Platform payment, refund, and cancel assertions are claims. They do not mutate issued invoices or invent a second Refund.
8. Raw payload is bounded PII. It is not a live Ticket and not an indefinite audit substitute.
9. Webhooks are server-only. An offline device does not accept a server-accepted order or emit provider acknowledgement.
10. Tenant isolation. Ids alone do not authorize. No last-write-wins.

## Follow-up ADRs

```text
Supplier Invoices and Accounts Payable
Accounting Posting and Export
Reporting, Analytics and Historical Snapshots
Audit Trail, Data Retention and Privacy
Public API, Webhooks and Integration Idempotency
Menu Publishing, Availability and Dayparts
```

Do not implement a concrete Wolt or Glovo adapter, menu publish, or DSAR automation from this ADR.

## See also

- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)
- [ADR 0014: Kitchen, Bar Production Routing and KDS](0014-kitchen-bar-production-routing-and-kds.md)
- [ADR 0015: Reservations, Waitlist and Guest Seating](0015-reservations-waitlist-and-guest-seating.md)
- [ADR 0016: Price Lists, Discounts, Comps and Approval Rules](0016-price-lists-discounts-comps-and-approval-rules.md)
- [ADR 0017: Staff Identity, Roles and Operator Authorization](0017-staff-identity-roles-and-operator-authorization.md)
- [ADR 0020: Offline POS Operation and Synchronization](0020-offline-pos-operation-and-synchronization.md)
- [ADR 0021: Customer Profiles, Consent and Loyalty](0021-customer-profiles-consent-and-loyalty.md)

## Out of scope

This ADR does not define:

- menu publish, availability, or dayparts (ADR 0029)
- customer profiles or consent (ADR 0021)
- accounting platform settlement (ADR 0025)
- detailed reporting (ADR 0026)
- gift cards (ADR 0030)
- Tablio public API or tenant-facing webhooks (ADR 0028)
- a concrete Wolt or Glovo adapter
- courier routing or a guest tracking map
- a campaign engine
- marketplace contract terms
- concrete retention days (ADR 0027)
- exact webhook skew, payload-size cap, or rate-limit numbers
- POS screen layout

## Amendment — 2026-08-16: Channel facts readable by ADR 0026

The original Decision that `ChannelOrder` is the canonical received-order record, and that an external payload is not the Ticket, remain in the original text.

ADR 0026 may read canonical channel and delivery facts. Raw provider payload stays out of generic analytics datasets.

This amendment does not change `ChannelOrder` first, the order fence, or provider-ack rules.

## Amendment — 2026-08-16: Tenant-facing API and webhooks owned by ADR 0028

The original Decision that `ChannelOrder` is the canonical received-order record, and that an external payload is not the Ticket, remain in the original text.

ADR 0028 now owns the tenant-facing public API and outbound webhooks. This ADR still owns the provider inbound inbox and the provider outbound outbox.

This amendment does not change `ChannelOrder` first, the order fence, or provider-ack rules.
