# ADR 0021: Customer Profiles, Consent and Loyalty

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a CRM product, a campaign engine, a guest mobile app, or POS application code.

## Date

2026-08-15

## Context

ADR 0001 isolates tenant data. ADR 0010 freezes `recipient_snapshot` on an issued Invoice. ADR 0012 may record a guest or business recipient on a Ticket. ADR 0015 may store a reservation or waitlist contact snapshot. ADR 0016 owns discount, Comp, and stacking order. ADR 0017 owns the permission catalog. ADR 0020 makes the server the only canonical offline authority.

Without this ADR, a reservation would silently create a marketing profile, one `marketing_consent` boolean would stand in for every purpose, phone or email would join guests across restaurants, loyalty points would look like cash, two devices would spend the same balance, and deleting a guest would erase invoices.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the guest-relationship domain **before** CRM or loyalty implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

Cite as legal duties, not as a complete compliance program: [EUR-Lex – GDPR](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX%3A02016R0679-20160504) (purpose limitation, data minimisation, storage limitation, distinct legal bases, and rights of access, rectification, erasure, restriction, portability, and objection to direct marketing, including Article 21), [AZOP – legal bases](https://azop.hr/pravni-temelji-za-obradu-osobnih-podataka/) (real choice, known purpose, and withdrawal without penalty).

The governing rule:

```text
CustomerProfile = who the guest is inside one tenant
ContactPoint    = how the guest may be reached
ConsentEvent    = proof for one exact purpose whose policy uses CONSENT
LoyaltyAccount  = points and program benefits
```

```text
Reservation guest snapshot      ≠ CustomerProfile
Invoice / Ticket recipient snapshot ≠ CustomerProfile
CustomerProfile                 ≠ global Tablio identity
LoyaltyAccount                  ≠ cash wallet / gift card / Payment
Consent                         ≠ legal basis for every processing
ProcessingPurposePolicy         = which legal basis applies to a purpose
```

A guest may reserve, sit, order, and pay without a loyalty account or marketing consent.

## Decision

### 1. Tenant-scoped profile — no global guest

One `CustomerProfile` belongs to exactly one tenant. Tablio does not create a global guest profile that is automatically shared across restaurants.

```text
CustomerProfile
---------------
tenant_id
customer_id
display_name
status
created_at
source
version
```

```text
ACTIVE
RESTRICTED
PSEUDONYMIZED
MERGED
```

v1 `CustomerProfile` has **no** generic unbounded CRM note field. Operational notes stay on the concrete process (reservation, waitlist, Ticket, Invoice) with that process’s own retention and scope. Structured special-category fields need a later model and legal review.

- The same person may have a separate profile at each tenant.
- Tenant A must not learn that the same person exists at tenant B.
- Phone, email, or card must not join profiles across tenants.
- A tenant cannot search or discover another tenant’s profiles.
- Loyalty points and status do not cross tenants.
- Marketing consent given to one tenant is not valid at another.
- Suppression is tenant-scoped.
- The platform must not create a hidden global customer ID for business use.
- Linking several brands later needs an explicit legal basis and a new ADR or amendment.

ADR 0001 isolation stays. This ADR owns the guest-identity lock. ADR 0001 is not amended.

### 2. Profile is optional and explicit

Reservation, waitlist, seating, Ticket, and payment must work without a `CustomerProfile`.

ADR 0015 may keep a contact snapshot without a CRM row:

```text
ReservationGuestSnapshot
- name
- contact used for reservation
- party information
```

Waitlist name and contact are the same class of snapshot, not a profile.

Creating a `CustomerProfile` must be an explicit staff or guest action, or a clearly defined, audited business process. It is never an implicit side-effect of reserve, sit, order, or pay. It never creates marketing consent.

A later profile edit must not rewrite name or contact frozen on an old reservation, waitlist entry, Ticket, Invoice, or fiscal document.

### 3. ContactPoint

Phone and email are not plain fields on the profile.

```text
ContactPoint
------------
customer_id
type                 # EMAIL | PHONE
normalized_value
display_value
verification_status
is_primary
status
```

```text
UNVERIFIED
VERIFIED
BOUNCED
INVALID
RETIRED
```

WhatsApp and SMS are **channels** on consent, not extra contact types. They use `PHONE`.

One person may have many contacts. The same normalized value is **not** unique inside a tenant. Family share, hotel booking, and parent/child are allowed. Duplicates are a warning. Profiles are not auto-merged.

### 4. Legal basis is a versioned policy

Consent is not the legal basis for every processing. Each purpose has a versioned policy:

```text
ProcessingPurposePolicy
-----------------------
tenant_id
purpose
legal_basis          # CONSENT | CONTRACT | LEGAL_OBLIGATION | LEGITIMATE_INTEREST
controller_identity
policy_version
retention_class
valid_from
valid_until
```

Locked examples:

- fulfilling a reservation → `CONTRACT`
- issuing and retaining an invoice → `LEGAL_OBLIGATION`
- email marketing → `CONSENT`, or another explicitly approved basis after legal review
- loyalty membership → contract under the program terms
- loyalty marketing → a separate marketing consent

A `ConsentEvent` may be recorded **only** for a purpose whose **active** policy uses `CONSENT`. A `LEGAL_OBLIGATION` or `CONTRACT` purpose must not mint a fake consent event.

`retention_class` is a handle for ADR 0027. This ADR does not lock concrete retention days.

### 5. Consent is an immutable, server-ordered event

Forbid `marketing_consent = true`.

```text
ConsentEvent
------------
event_id
customer_id
contact_point_id
purpose
channel
action               # GRANTED | WITHDRAWN
policy_version
notice_version
legal_controller
effective_at
server_recorded_at
source_occurred_at
idempotency_key
evidence_hash
locale
rendered_notice_hash
source
proof
actor
```

`source_occurred_at` may come from a client, import, or offline device. It is **not** the ordering clock.

Effective consent is a **projection** of events, ordered deterministically by server-side sequence (`server_recorded_at` / accepted sync order). It is not a mutable boolean.

- A stale imported `GRANTED` must not override a newer `WITHDRAWN`.
- Grant and withdraw are idempotent.
- A backdated grant cannot turn marketing back on.
- Withdrawal is effective as soon as the server accepts it.
- An offline-recorded grant is **not** active for marketing until the server accepts it.
- An offline withdrawal, once synced, takes precedence over a later send.

Minimum purposes:

```text
LOYALTY_MEMBERSHIP
EMAIL_MARKETING
SMS_MARKETING
WHATSAPP_MARKETING
PERSONALIZED_OFFERS
PROFILING
```

Consent is purpose-specific and as easy to withdraw as to give.

Loyalty membership is not marketing consent. Two separate acceptances. Withdrawing marketing:

- stops future marketing on that channel
- does not close the loyalty account
- does not delete reservations or invoices
- does not void already earned points
- does not erase data kept on another legal basis

Consent is not the legal basis for invoice, payment, reservation, or statutory document retention.

### 6. Suppression is tenant HMAC

After withdraw or a marketing objection, write:

```text
MarketingSuppression
--------------------
tenant_id
channel
purpose_scope
destination_hmac
key_generation
normalization_version
suppressed_at        # server timestamp
reason
```

```text
destination_hmac = HMAC(tenant_suppression_key, normalized_destination)
```

A plain SHA, or any other unsalted hash of email or phone, is rejected. The value space is too small.

Key rotation must still verify old rows or reindex them under control. Suppression must not become a hidden marketing profile.

Suppression **wins** over consent on **every** profile in the same tenant that uses the same normalized contact. A new consent must **not** automatically lift suppression. Reactivation needs new proof and an audited action.

Before every marketing send, the backend re-checks at **send time**:

```text
contact valid
AND purpose/channel consent active
AND no suppression
AND tenant/location policy allows
AND message is inside consent scope
AND purpose policy still allows the send
```

A reservation confirmation is not automatically marketing. A transactional message must not hide a promotion without a matching basis.

### 7. LoyaltyProgram and LoyaltyAccount

A tenant may have one or more programs.

```text
LoyaltyProgram
--------------
tenant_id
name
scope                # TENANT | LOCATION_SET
currency/type        # POINTS
status
terms_version
rules_version
valid_from
valid_until
```

```text
DRAFT
ACTIVE
SUSPENDED
RETIRED
```

Published rules are immutable. A change creates a new version. Rules may define earn method, eligible products and sale actions, whether discount and Comp enter the earn base, availability delay, expiry, redemption threshold, max spend per Ticket, and eligible locations.

```text
LoyaltyAccount
--------------
loyalty_program_id
customer_profile_id
membership_number
status
joined_at
terms_version_accepted
```

```text
ACTIVE
SUSPENDED
CLOSED
```

At most one active account per profile and program. `membership_number` is stable and must not embed predictable personal data. A membership barcode may **identify** an account. It does **not** by itself authorize spend.

### 8. Points are a server ledger

```text
LoyaltyLedgerEntry
------------------
loyalty_account_id
type
points
source_reference
occurred_at
available_at
expires_at
idempotency_key
rule_version
actor
```

```text
EARN_PENDING
EARN_AVAILABLE
REDEEM
REFUND_REVERSAL
EXPIRY
MANUAL_ADJUSTMENT
TRANSFER
```

Balance is derived. A correction is a compensating row, never an edit of history.

`Ticket POSTED` is not a sufficient earn key. Split pay, several invoices, and later reversal make the Ticket too wide. Earn binds a stable source:

```text
LoyaltyEarnSource
-----------------
invoice_id / fiscal_document_id
eligible_line_snapshot
settled_amount
loyalty_rule_version
```

```text
UNIQUE (
  loyalty_program_id,
  source_type,
  source_id,
  earn_kind
)
```

Points are computed from the **frozen** eligible lines and the actually recognized amount. Refund or reversal references the **original** earn snapshot. It must not recompute proportionality from today’s catalog or today’s loyalty rules.

```text
Invoice / fiscal source recognized
→ EARN_PENDING
→ refund/settlement condition elapsed
→ EARN_AVAILABLE
```

Void or refund: pending is cancelled; available gets a reversal; the original earn stays; a partial refund returns points proportionally under the frozen earn snapshot.

`TRANSFER` is a compensating out/in pair that shares one transfer ID. v1 does **not** enable guest-initiated account-to-account or cross-program transfer. Without an enabled program transfer rule, merge stays blocked.

### 9. Redemption requires guest authorization

Knowing a membership number, phone, or name is **not** enough to spend points.

```text
LoyaltyRedemptionAuthorization
------------------------------
loyalty_account_id
ticket_id
ticket_version
max_points
verification_method
verified_at
expires_at
nonce
consumed_at
```

Methods: loyalty PIN; one-time code; guest QR or token; confirmation in a future guest app; manager override with audit.

The authorization is short-lived, bound to the exact Ticket and amount, single-use, and consumed **atomically** with the redemption. Redemption must also lock the account, recompute available balance, require sufficient balance, be idempotent, apply the ADR 0016 benefit in the **same** transaction, and forbid a negative balance. Two devices must not spend the same points.

v1: points are a right to a program benefit, not cash, a gift card, or a tender. Euro value, top-up, cash-out, and gift card stay ADR 0030. Tax and fiscal result of the benefit stay ADR 0009 and ADR 0010.

ADR 0016 application order becomes:

```text
base
→ automatic (0008)
→ manual line discounts
→ ticket-level discount allocation
→ loyalty redemption allocation
→ Comp
```

Loyalty is a distinct allocated Ticket benefit, not a Comp and not a Payment. Comp stays last. Line total cannot go negative. A `POSTED` Ticket cannot receive a new redemption.

### 10. Offline — command, not a local ledger

The server remains the only canonical authority (ADR 0020).

- Last confirmed balance may be shown as informational.
- Cached or locally estimated points must be **visually distinct** from the server-confirmed balance.
- The device writes a signed `AccrueLoyaltyEarn` command. Local UI may show only `PENDING_SYNC`.
- The device must **not** create a canonical `LoyaltyLedgerEntry`.
- After server accept, the server creates canonical `EARN_PENDING`.
- A rejected offline sale creates no points.
- The same offline command and the same financial source cannot earn twice.
- Redemption from a plain cached balance is forbidden.
- Offline redemption would need an exclusive server-issued budget or lease **and** a valid `LoyaltyRedemptionAuthorization`. v1 issues neither.

### 11. Merge and split

Auto-merge by phone or email is rejected.

Controlled merge requires an explicit survivor, a source, a reason, an operator and permission, contact and consent conflict review, audit, and idempotency. Each `ConsentEvent` keeps its own proof, contact, purpose, and source. Consent is not OR-ed to `true`.

If **both** profiles have an account for the **same** program, merge is **blocked**. A separate maker-checker **loyalty resolution** must run first. Options: keep one account; controlled transfer if the program rule allows; or close an account under the terms. Every decision keeps **both** original ledgers.

v1 without an enabled transfer rule → merge stays blocked.

A `SUSPENDED` or `CLOSED` account that still has remaining points must not be silently discarded.

Split after merge is not a simple `Undo`. Merge must show consequences, be permission-protected, keep source mapping, and open a controlled correction workflow.

### 12. Subject rights, delete, special data, minors

This ADR must make profile, contacts, consent, suppression, purpose policy, and loyalty ledger findable, exportable, and controllably pseudonymizable.

ADR 0027 later owns retention periods, legal hold, privacy workflow, pseudonymization execution, and DSAR fulfillment.

`Delete customer` must not physically erase invoices, fiscal documents, or the loyalty ledger without a legal-basis check.

```text
ACTIVE → RESTRICTED → PSEUDONYMIZED
ACTIVE → MERGED
```

Pseudonymization removes or replaces direct identifiers, stops marketing, keeps required financial and audit references, keeps minimal suppression proof, and does not rewrite old fiscal documents.

Do not use allergies or inferred order data for marketing or loyalty profiling. Do not rely on scanning free text for special-category markers. v1 has no unbounded CRM note.

Minor loyalty enrollment is a **configured** policy: age threshold by applicable law, parental proof where required, no automatic marketing, minimized data, and no purchase profiling. This ADR does not lock a universal age.

### 13. Permissions

ADR 0017 owns the catalog. This ADR adds:

```text
customer.view
customer.contact_view
customer.create
customer.update
customer.merge
customer.restrict
consent.record
consent.proof_view
loyalty.view
loyalty.enroll
loyalty.redeem
loyalty.adjust
loyalty.suspend
privacy.request_manage
```

- `customer.view` does **not** grant contacts, consent proof, or loyalty ledger.
- `customer.contact_view` is narrower than viewing a reservation.
- `consent.proof_view` is required to read grant or withdraw evidence.
- `loyalty.view` is required to read balance and ledger.
- `loyalty.redeem` is required to consume a `LoyaltyRedemptionAuthorization`.
- Manual `loyalty.adjust` requires reason, points, operator, approval above threshold, and a compensating ledger row.

`privacy.request_manage` records or forwards a request. Execution stays ADR 0027.

### 14. Audit

Keep at least: profile create or change and source; contact source and verification; purpose-policy version and legal basis; consent notice and policy version; grant or withdraw proof and server timestamps; suppression HMAC generation; loyalty rule version; every ledger event and earn source; redemption-authorization consume; merge, loyalty resolution, and correction; export, restrict, and pseudonymization; who viewed or exported sensitive data.

Access audit must not be used for marketing profiling.

### 15. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Reserve, sit, order, or pay without a `CustomerProfile` succeeds; no profile row and no marketing consent are created.
- Profile create without an explicit staff or guest action or a named process is rejected.
- The same phone at two tenants yields two isolated profiles; tenant A search does not return tenant B.
- Platform lookup by phone, email, or card across tenants is rejected; no hidden global customer ID is stored for business use.
- Marketing consent at tenant A does not authorize send at tenant B.
- A duplicate phone inside one tenant warns and does not auto-merge.
- `marketing_consent = true` as a single field is rejected / not modeled.
- A purpose whose active policy is `LEGAL_OBLIGATION` or `CONTRACT` cannot create a `ConsentEvent`.
- Loyalty enroll without marketing purposes leaves marketing consent inactive.
- Withdraw `EMAIL_MARKETING` stops later email marketing, keeps the loyalty account, and does not void earned points or invoices.
- Send-time check fails if consent was withdrawn after the campaign was composed.
- A reservation confirmation may send without marketing consent; the same message with a hidden promo is rejected.
- A stale imported `GRANTED` does not override a newer `WITHDRAWN`.
- A backdated grant does not re-enable marketing after a later server-accepted withdraw.
- An offline grant is not active for marketing before server accept.
- After withdraw, suppression blocks re-import and re-send even if the profile is later `PSEUDONYMIZED`.
- A plain SHA, or other unsalted hash of a contact, is not an allowed suppression key; tenant HMAC is required.
- Suppression of one contact blocks send via another profile in the same tenant that uses that contact.
- A new consent does not automatically lift suppression.
- Two active `LoyaltyAccount`s for the same profile and program are rejected.
- `Ticket POSTED` without a bound invoice or fiscal `LoyaltyEarnSource` does not create `EARN_AVAILABLE`.
- The same invoice or fiscal source cannot earn twice for the same program and `earn_kind`.
- Retry of the same earn `idempotency_key` or source returns the same ledger result, not a second earn.
- A partial refund reverses points from the **frozen original earn snapshot**, not from today’s catalog or rules; the original earn row remains.
- Concurrent redemptions on one account: one succeeds, the other sees insufficient balance; the balance never goes negative.
- Redemption and Ticket benefit apply in one transaction or neither.
- Membership number, phone, or name without a live `LoyaltyRedemptionAuthorization` cannot spend points.
- Offline redemption against a cached balance is rejected in v1.
- An offline `AccrueLoyaltyEarn` before server accept is not canonical loyalty balance; UI is `PENDING_SYNC` only.
- A rejected offline sale creates no ledger row.
- Auto-merge by email or phone is rejected.
- Merge that ORs consent to `true` is rejected; each `ConsentEvent` remains.
- Two profiles with accounts for the same program cannot merge without a prior maker-checker loyalty resolution.
- v1 without an enabled transfer rule keeps that merge blocked; balances are not silently summed or discarded.
- A `SUSPENDED` or `CLOSED` account with remaining points is not silently dropped on merge.
- `Undo merge` as a silent reverse is rejected.
- Profile edit does not change name or contact on an old reservation, Ticket, or issued Invoice.
- `Delete customer` does not physically remove an issued Invoice or loyalty ledger; status becomes `RESTRICTED` or `PSEUDONYMIZED`.
- The API has no generic unbounded customer-note field on `CustomerProfile`.
- Manual `loyalty.adjust` without reason or above-threshold approval is rejected.
- `customer.view` does not grant contact, consent proof, or loyalty ledger.
- Access-audit rows are not usable as a marketing segment.

## Rejected alternatives

- A global cross-tenant guest profile or a hidden global customer ID.
- A mandatory profile on reserve, sit, order, or pay.
- Automatic marketing consent on profile create.
- One `marketing_consent` boolean.
- Treating every processing as consent.
- A `ConsentEvent` for a `LEGAL_OBLIGATION` or `CONTRACT` purpose.
- Client `collected_at` as the consent clock.
- A stale import grant overriding a newer withdraw.
- An offline grant active before server accept.
- A plain SHA suppression of email or phone.
- New consent automatically lifting suppression.
- A local canonical loyalty ledger or a local `EARN_PENDING` row.
- Earn keyed only by `Ticket POSTED`.
- Double earn on the same invoice source.
- A refund recomputed from today’s rules.
- Spend by membership number, phone, or name alone.
- Offline redemption from a plain cache.
- Points as a hidden cash wallet.
- Physical delete of required financial documents.
- An unbounded CRM note on `CustomerProfile`.
- Scanning notes for special-category markers as a control.
- Silently summing or discarding loyalty on merge.
- Merge of two same-program accounts without loyalty resolution.
- Simple `Undo` after merge.
- Treating a transactional booking message as a marketing license.
- Amending ADR 0001 in this change.

## Consequences

### Positive

- A walk-in can pay without becoming a marketing contact.
- Tenant isolation of the guest relationship stays aligned with ADR 0001.
- Marketing cannot hide behind loyalty membership or invoice retention.
- Two devices cannot spend the same points from a cached balance.
- Pseudonymization cannot rewrite an issued Invoice.

### Negative

- v1 cannot redeem loyalty while offline.
- Merge of two members of the same program is blocked until a separate resolution.
- Staff who can view a reservation still cannot see contacts, consent proof, or the loyalty ledger without extra rights.

### Neutral

- Documentation can merge without a CRM UI, campaign engine, or message provider.
- Concrete retention days and DSAR execution stay ADR 0027.
- Gift cards and stored value stay ADR 0030.

## Invariants

1. `CustomerProfile` ≠ `ContactPoint` ≠ `ConsentEvent` ≠ `LoyaltyAccount` ≠ Invoice or Reservation snapshot.
2. One profile belongs to exactly one tenant. There is no global guest identity for business use.
3. Reserve, sit, order, and pay do not require a profile and do not mint marketing consent.
4. A `ConsentEvent` exists only when the active `ProcessingPurposePolicy` uses `CONSENT`. Effective consent is a server-ordered projection.
5. Suppression is tenant HMAC and wins over consent for that normalized destination. A new consent does not lift it.
6. The server is the only canonical loyalty ledger. Earn binds a final invoice or fiscal source. The same source cannot earn twice.
7. Redemption consumes a short-lived `LoyaltyRedemptionAuthorization` atomically with the ADR 0016 benefit. Comp stays last.
8. Merge does not OR consent and does not silently sum or drop same-program accounts.
9. Delete and pseudonymization do not physically erase required financial documents or rewrite issued invoices.
10. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Ordering Channels, Delivery and External Platforms
Accounting Posting and Export
Reporting, Analytics and Historical Snapshots
Audit Trail, Data Retention and Privacy
Gift Cards, Vouchers and Stored Value
```

Do not implement a campaign engine, channel adapters, gift cards, or DSAR automation from this ADR.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0010: Invoices and Fiscalization](0010-invoices-and-fiscalization.md)
- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)
- [ADR 0015: Reservations, Waitlist and Guest Seating](0015-reservations-waitlist-and-guest-seating.md)
- [ADR 0016: Price Lists, Discounts, Comps and Approval Rules](0016-price-lists-discounts-comps-and-approval-rules.md)
- [ADR 0017: Staff Identity, Roles and Operator Authorization](0017-staff-identity-roles-and-operator-authorization.md)
- [ADR 0020: Offline POS Operation and Synchronization](0020-offline-pos-operation-and-synchronization.md)

## Out of scope

This ADR does not define:

- a campaign engine or automated marketing journeys
- channel adapters (ADR 0022)
- gift cards or stored value (ADR 0030)
- retention periods, legal hold, or DSAR automation (ADR 0027)
- accounting loyalty liability (ADR 0025)
- BI segmentation (ADR 0026)
- a guest mobile app
- a concrete email, SMS, or WhatsApp provider
- a special-category operational schema
- brand-group identity
- exact earn-available delay, membership-number format, HMAC rotation interval, or verification-method UX
- POS screen layout

## Amendment — 2026-08-15: Channel customer snapshot owned by ADR 0022

The original Decision that a reservation, Ticket, or Invoice snapshot is not a live `CustomerProfile`, and that profile create is explicit, remain in the original text.

ADR 0022 owns the channel customer and delivery snapshots. An inbound order must not auto-create a profile, merge by phone, write marketing consent, touch loyalty, or lift suppression. Raw provider payload is bounded personal data, not a CRM note and not an indefinite audit substitute.

This amendment does not change tenant-scoped profiles, `ConsentEvent` ordering, or the loyalty ledger.

## Amendment — 2026-08-16: Generic analytics datasets owned by ADR 0026

The original Decision that `CustomerProfile`, consent, and the loyalty ledger stay tenant-scoped, and that snapshots are not the live profile, remain in the original text.

ADR 0026 owns generic reporting datasets. Those datasets are aggregates. Customer PII and note content stay out. Loyalty metrics stay a later analytics source and are not authorized as v1.

This amendment does not change tenant-scoped profiles, `ConsentEvent` ordering, or the loyalty ledger.

## Amendment — 2026-08-16: Retention and DSAR execution owned by ADR 0027

The original Decision that `CustomerProfile`, consent, and the loyalty ledger stay tenant-scoped, and that snapshots are not the live profile, remain in the original text.

ADR 0027 now owns retention, legal hold, and DSAR execution. `ConsentEvent` stays this ADR. `retention_class` binds to `RetentionBinding` and `RetentionPolicy`. Merge joins the tenant-scoped alias cluster. `privacy.request_manage` still only records or forwards a request.

This amendment does not change tenant-scoped profiles, `ConsentEvent` ordering, or the loyalty ledger.

## Amendment — 2026-08-16: Public API customer writes stay on ADR 0021 commands

The original Decision that `CustomerProfile`, consent, and the loyalty ledger stay tenant-scoped, and that snapshots are not the live profile, remain in the original text.

ADR 0028 owns the public API. `customers.write` still goes through this ADR’s commands. The API must not mint marketing consent or bypass `PrivacySubjectFence`.

This amendment does not change tenant-scoped profiles, `ConsentEvent` ordering, or the loyalty ledger.
