# ADR 0010: Invoices and Fiscalization

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

It must not become `Accepted` and must not be implemented until ADR 0009 has a confirmed tax formula and rounding rule. Concrete payment-method to fiscal-code mapping for cash, card, transaction account, and mixed tenders must also be locked before implementation.

This ADR does not authorize fiscal, eInvoice, or eReporting adapters.

## Date

2026-08-15

## Context

ADR 0006 owns the POS Ticket and `SALE` movements. ADR 0009 owns tax calculation and forbids fiscalization from mutating tax. The Fiscalization Act (NN 89/2025) applies from 1 January 2026 to both end-consumer invoices and structured eInvoices. B2C fiscalization applies regardless of payment method, including a transaction account.

Without this ADR, a POS print would be treated as the legal invoice, a waiter button would choose “R1” versus eRačun, a network outage would roll an issued invoice back to draft, a B2B cash sale would emit both a fiscal receipt and an eInvoice, and a ticket reversal would void stock without a legal credit document.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the outgoing sales invoice domain **before** adapters. Physical schema details belong in a later implementation. The semantics below must not change once accepted. They do not authorize implementation while ADR 0009 rounding and payment fiscal codes are unset.

The governing rule:

```text
POS Ticket records the operational sale.
Invoice is the immutable legal and commercial document.
Fiscalization submits and records legally required data about that Invoice.
eInvoice Exchange delivers a structured Invoice to an eligible business recipient.
Payment records settlement and does not determine whether the sale exists.
```

```text
Invoice is the immutable legal document.

B2C fiscalization, B2B eInvoice exchange, eInvoice fiscalization, rendering,
and payment are separate processes attached to the same frozen Invoice.

Posting creates the local Invoice, stock effects, tax snapshots, number,
and outbox atomically. External failure never mutates or renumbers the Invoice.
```

This ADR governs **outgoing sales Invoices only**.

Cite as legal duties, not as a complete wire specification: [Zakon o fiskalizaciji, NN 89/2025](https://narodne-novine.nn.hr/clanci/sluzbeni/2025_06_89_1233.html) (arts. 8, 15, 21, 22, 24, 39, 42, 43, 48, 49), [Porezna uprava – B2C](https://porezna-uprava.gov.hr/hr/opcenito-8032/8032), [izdavatelji i primatelji eRačuna](https://porezna-uprava.gov.hr/hr/izdavatelji-i-primatelji-eracuna-te-obveza-izdavanja-eracuna-azurirano-7-11-2025/8048), [slijednost računa](https://porezna-uprava.gov.hr/hr/slijednost-racuna-8050/8050).

## Decision

### 1. One Invoice, exclusive legal routes

Do not create unrelated masters for “R1”, fiscal receipt, and eRačun.

```text
Invoice
-------
tenant_id
issuer_legal_entity_id
business_location_id
source_document_type
source_document_id
recipient_type
recipient_snapshot
invoice_number
currency
issued_at
supply_at
due_at
status                      # ISSUED
correction_status           # NONE | PARTIALLY_CREDITED | FULLY_CREDITED
totals
tax_snapshots
routing_decision
routing_legal_basis
```

Routing is server-side, from recipient and transaction. A waiter button does not choose the channel.

```text
Consumer
→ B2C_FISCAL

Domestic eligible business + eInvoice route
→ B2B_EINVOICE

Domestic eligible business + cash/card exception explicitly selected and valid
→ B2C_FISCAL
→ B2B_EINVOICE forbidden for the same transaction

AMS identifier unavailable
→ PAPER_OR_OTHER_FALLBACK
→ mandatory eReporting
```

- `B2C_FISCAL` and `B2B_EINVOICE` are mutually exclusive for the same supply (Act art. 39).
- The backend checks whether the cash/card exception is allowed. A **transaction account does not qualify**.
- A user cannot later issue an eInvoice for a transaction already fiscalized on the other route.
- Routing decision and legal basis remain on the snapshot.

**`B2C_FISCAL`.** End-consumer invoice, or a valid B2B cash/card exception. True B2C is fiscalized regardless of cash, card, or transaction account. Uses JIR/ZKI and QR when applicable. Payment method is a required fiscal datum.

**`B2B_EINVOICE`.** Structured eInvoice for a domestic transaction to a legally eligible recipient. Not a PDF sent by email. Follows the EU norm plus Croatian extensions. eInvoice **exchange** and **issued-eInvoice fiscalization** are two workflows. In 2026, VAT-registered parties issue and receive for covered domestic transactions; some non-VAT parties receive in 2026 and issue from 2027.

**`PAPER_OR_OTHER_FALLBACK`.** Only when the law does not require the first two routes, or a legal fallback exists (recipient identifier unavailable in AMS), plus mandatory eReporting. Not a user shortcut. A user cannot mark “AMS unavailable” by hand.

### 2. POS Ticket is not the legal invoice

- One posted POS Ticket creates **at most one** original Invoice.
- A repeated post request must not allocate a new invoice number (ADR 0003 / 0006 idempotency).
- The Invoice stores its own immutable lines, amounts, and ADR 0009 tax results.
- Print or PDF is not Invoice authority. JIR is not the primary Invoice identity. eInvoice XML is not the only copy of business truth.
- External payloads are generated from the frozen Invoice snapshot. Changing a print template does not change the issued Invoice.

**Split bill.** Split while the ticket is still `DRAFT` into separate tickets. Each posts independently and receives one Invoice. A posted ticket cannot be split later. Stock effects book exactly once.

Ticket lifecycle remains ADR 0006 (`DRAFT → POSTED → REVERSED`). Ticket `REVERSED` does not un-issue the Invoice.

### 3. The original stays `ISSUED`; correction is a new document

```text
Invoice.status = ISSUED

correction_status:
NONE
PARTIALLY_CREDITED
FULLY_CREDITED
```

There is no `CANCELLED` or `CANCELLED_BY_CREDIT_DOCUMENT`. The original legal document stays issued and unchanged. A new storno or credit document corrects its effect (Act art. 24; negative amounts are allowed on return).

Each corrective document has its own number, references the original number and date, has its own fiscal and eInvoice flows, and does not delete or rewrite the original. A full reversal copies commercial and tax amounts with the opposite sign from the original snapshot.

An issued Invoice never returns to draft because Porezna uprava, an access point, or the network is down.

```text
Invoice issued successfully
≠
External fiscal submission already accepted
```

### 4. Four external workflows

Not one generic `channel_status`.

```text
B2C fiscal submission
eInvoice exchange/delivery
issued eInvoice fiscalization
fallback eReporting
```

Each workflow stores: status, attempt count, last attempt, next retry, statutory deadline, request fingerprint, external identifiers, response/error snapshot, adapter/schema version.

Statuses include `NOT_REQUIRED`, `PENDING`, `SUBMITTED`, `ACCEPTED`, `REJECTED`, `RETRY_DUE`, `PERMANENT_FAILURE`, and B2C `OFFLINE_ISSUED` where applicable.

`eInvoice delivered` does not mean `eInvoice fiscalized`. Fiscal acceptance does not mean the recipient received it.

### 5. Atomic local post; immediate B2C attempt; outbox for retry

POS ticket post, in **one local database transaction**:

1. Lock the ticket.
2. Run the **channel-specific validator**. Fail before allocating a number.
3. Allocate a non-repeatable invoice number.
4. Freeze the Invoice, ADR 0009 tax results, and ZKI.
5. Create `SALE` movements through the ADR 0003 writer.
6. Create the required workflow outbox rows.

External HTTP is not inside that transaction. Rollback must not leave a partially issued Invoice.

**B2C after commit** (Act arts. 15 and 21):

```text
Local commit Invoice + number + ZKI + outbox
→ immediate bounded B2C submission attempt
→ JIR received: render final receipt with JIR
→ classified technical/network failure: OFFLINE_ISSUED, lawful no-JIR receipt
→ durable outbox retry before statutory deadline
```

- The first attempt is immediate, not arbitrarily deferred.
- The offline path is used only after a **classified technical** error. A business rejection is not offline.
- Every offline invoice gets `retry_deadline_at` (two working days for B2C).
- Missing the deadline raises a **critical alert**. It does not silently become `PERMANENT_FAILURE`.
- The outbox is the retry authority. Retry uses the same Invoice, number, ZKI, canonical payload, and idempotency identity.

Technical inability to fiscalize an eInvoice: retry within up to five working days (art. 49). Total device failure and the certified invoice book (arts. 21–22) remain a later operational hook.

### 6. Numbering, premises, and device

```text
numeric / business-premises code / device code
```

The numeric part is an unbroken sequence under the legally chosen sequence model.

- The client does not send the next number.
- The counter is locked in the database.
- Sequence scope is explicit: premises or device.
- An allocated number is never reused.
- Failed external fiscalization does not release the number.
- A credit document gets its own number.
- The snapshot stores all number parts, sequence scope, counter period (for example calendar year), and internal-act version.

Premises and device must be `REGISTERED` / `ACTIVE`. Sequence policy must match the versioned internal act. An inactive premises or device blocks issue **before** number allocation. Parallel tills must not produce the same number.

### 7. Issuer and operator

Issuer identity comes from backend legal-entity configuration (OIB, name, address, VAT status), not from POS free text. Premises and device codes come from the ticket’s trusted location and device registration.

Operator snapshot (Act art. 8):

```text
operator_public_code
operator_oib_encrypted_or_protected
operator_display_snapshot
internal_act_version
```

The public code is what prints. OIB is sent only to the fiscal adapter. OIB is not printed as a substitute for the public code, is not written to ordinary application logs, and is available only to the fiscal adapter and authorized audit. A self-checkout or unattended device may use the obligor’s OIB as the law requires.

Missing issuer, active premises, device, operator, or internal-act version blocks local issue. No number is allocated.

### 8. Channel-specific validators

The backend runs the validator for the chosen route **before** allocating a number.

**B2C profile.** Issuer and OIB, issuer VAT status, issue date and time, operator public code, operator OIB for the fiscal message, payment method, amounts by rate and exemption, ZKI, JIR when received, QR payload. Invoice number is assigned last in the same transaction after the other checks pass.

**B2B eInvoice profile** (Act art. 48). Issuer and recipient, issue and due dates, supply date when required, quantity and type of goods or services, six-digit KPD, unit price excluding VAT, discounts, bases by rate or exemption, VAT rates and amounts, original-invoice reference on a correction, payment account or payment identifier.

### 9. KPD

Every eInvoice goods or services line requires a six-digit KPD (Act art. 42). `kpd_code` lives on the eInvoice line snapshot. It comes from a controlled classification, not POS free text. Missing or invalid KPD blocks `B2B_EINVOICE`. A later catalog change does not rewrite an issued Invoice. A B2C invoice does not block for missing KPD unless that channel legally requires it.

### 10. ZKI, certificate, and QR

- ZKI is computed from a versioned canonical algorithm. Store ZKI, algorithm version, and certificate fingerprint.
- The private key is never stored on the Invoice, in an outbox payload, or in logs.
- Certificate rotation does not change an already issued ZKI.
- Retry uses the identical canonical fiscal payload.
- No fiscally relevant field may change after ZKI.
- QR stores the canonical payload or deterministic inputs and generator version, not only a PNG.

### 11. AMS fallback proof

The fallback snapshot stores: recipient identifier queried, query time, result or unavailability category, response/reference id if any, adapter/schema version, legal fallback reason code, created eReporting outbox and its deadline. A user cannot manually mark AMS unavailable.

### 12. Atomic full reversal and credit Invoice

One local transaction:

1. Lock the original ticket and Invoice.
2. Assert that a full reversal does not already exist.
3. Create reversal stock movements.
4. Copy tax results with the opposite sign.
5. Allocate a new number for the corrective Invoice.
6. Create the corrective Invoice (references original number and date).
7. Create the matching fiscal / eInvoice outbox rows.
8. Set the original `correction_status = FULLY_CREDITED`.

Stock reversal without the legal document, or a credit document without stock reversal, is forbidden. Partial credit remains a later ADR. `PARTIALLY_CREDITED` is reserved.

### 13. Non-tax eInvoice correction is not a credit

Act art. 43 allows resending a corrected eInvoice under the **same number** when the change does not affect tax, with a copy indicator.

```text
Invoice remains immutable
→ create TransmissionRevision
→ same invoice number
→ copy indicator
→ only legally permitted non-tax fields
→ new exchange and fiscalization attempts
```

A change to price, quantity, tax, recipient, or supply must not go through as a copy. Those require a corrective Invoice.

## Rejected alternatives

- Unrelated masters for R1, fiscal receipt, and eRačun.
- Always routing an eligible business to `B2B_EINVOICE` (ignores Act art. 39).
- Both `B2C_FISCAL` and `B2B_EINVOICE` for the same supply.
- Applying the cash/card exception to a transaction-account payment.
- Treating the POS Ticket, a PDF, or JIR as the Invoice master.
- A waiter button choosing the channel, or a user marking AMS unavailable.
- An emailed PDF as an eRačun.
- Porezna uprava or CIP HTTP inside the stock/invoice database transaction.
- Deferred-only B2C submit with no immediate attempt.
- Treating a business rejection as an offline outage.
- Silently setting `PERMANENT_FAILURE` after missing the statutory deadline.
- `ISSUED → CANCELLED_BY_CREDIT_DOCUMENT`.
- Stock reversal without a credit Invoice, or the reverse.
- One generic `channel_status`.
- Free-text KPD, or mutating the Invoice master for an art. 43 copy.
- A client-supplied next number, or reusing a number after fiscal failure.
- Printing operator OIB instead of the public code, or logging OIB.
- Splitting an already posted ticket.
- Incoming supplier eInvoices in this ADR.
- Implementing adapters from this Proposed ADR before ADR 0009 confirmation and payment fiscal-code mapping.
- Amending ADR 0002–0009 in this change.

## Consequences

### Positive

- One legal Invoice can travel B2C fiscal, eInvoice exchange, or fallback eReporting without cloned masters.
- A network outage cannot un-issue or renumber a committed Invoice.
- A B2B cash sale cannot emit two legal invoices for one supply.
- Ticket reversal cannot move stock without a numbered credit document.
- Split-bill UX cannot post two invoices against one stock event.

### Negative

- Implementation waits on ADR 0009 acceptance and payment fiscal-code mapping.
- Operators cannot post if premises, device, operator, KPD (B2B), or AMS proof is incomplete.
- Offline B2C creates operational urgency: a missed two-working-day deadline is a critical alert.

### Neutral

- Documentation can merge without CIS/CIP adapters.
- The Payments ADR will own settlement. This ADR only snapshots payment method and uses it for routing.
- Incoming supplier eInvoices wait for their own ADR.

## Invariants

1. POS Ticket ≠ Invoice ≠ the four external workflows ≠ payment. This ADR governs outgoing sales Invoices only. Fiscalization does not calculate tax or move stock.
2. `B2C_FISCAL` and `B2B_EINVOICE` are mutually exclusive for one supply. The B2B cash/card exception is explicit, backend-validated, and snapshotted. A transaction account cannot use it.
3. One posted POS Ticket creates at most one original Invoice. Post is idempotent. Split only while `DRAFT`. Print, JIR, and XML are not the master.
4. Original `status` remains `ISSUED`. `correction_status` is `NONE`, `PARTIALLY_CREDITED`, or `FULLY_CREDITED`. A credit is a new numbered document that references the original.
5. The channel validator runs before number allocation. Local post is atomic. B2C’s first submit is an immediate bounded attempt after commit. Offline is only a classified technical failure, with `retry_deadline_at`. Missing the deadline is a critical alert, not a silent permanent failure.
6. Four workflows each carry attempt metadata. Delivered ≠ fiscalized ≠ received.
7. Number form is `numeric / premises / device`. The counter is database-locked. Premises and device must be active. Internal-act version and period are snapshotted. Numbers are never reused. Parallel tills cannot collide.
8. Operator public code prints; protected OIB is for the fiscal adapter and authorized audit only. ZKI uses a versioned canonical algorithm and certificate fingerprint. The private key never appears on Invoice, outbox, or logs. QR stores canonical inputs, not only a PNG.
9. Six-digit KPD is required on B2B eInvoice lines from a controlled classification. AMS fallback requires query proof and an eReporting outbox.
10. Full reversal is one local transaction: opposite stock, opposite tax snapshots, new credit Invoice, outbox, and original `FULLY_CREDITED`.
11. A non-tax eInvoice resend is a `TransmissionRevision` (same number, copy indicator). Price, quantity, tax, recipient, or supply changes require a corrective Invoice.
12. Tenant isolation. Ids alone do not authorize. This ADR stays `Proposed` and does not authorize adapters until ADR 0009 and payment fiscal codes are locked.

## Follow-up ADRs

```text
Payments                            # settlement + fiscal-code mapping
Incoming eInvoice and AP            # supplier receive, inbound fiscalization
Partial return / refund
POS layout
```

The next domain ADR should define **Payments**, including the fiscal-code mapping this ADR needs before implementation.

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
- [Zakon o fiskalizaciji, NN 89/2025](https://narodne-novine.nn.hr/clanci/sluzbeni/2025_06_89_1233.html)
- [Porezna uprava – B2C](https://porezna-uprava.gov.hr/hr/opcenito-8032/8032)
- [Porezna uprava – izdavatelji i primatelji eRačuna](https://porezna-uprava.gov.hr/hr/izdavatelji-i-primatelji-eracuna-te-obveza-izdavanja-eracuna-azurirano-7-11-2025/8048)
- [Porezna uprava – slijednost računa](https://porezna-uprava.gov.hr/hr/slijednost-racuna-8050/8050)

## Out of scope

This ADR does not define:

- implementation or activation of fiscal, eInvoice, or eReporting adapters
- the concrete payment-method to fiscal-code mapping
- ADR 0009 tax-engine implementation
- incoming supplier eInvoices, inbound fiscalization, rejection, or AP
- matching to ADR 0004 Supplier Invoice
- payments, tenders, or FX beyond the payment-method snapshot used for routing
- POS screen layout, buttons, or KDS
- certified invoice-book paper procedure
- CIS / CIP / AMS wire formats
- partial return or refund documents
- cross-border eInvoice
- recurring or advance invoices

## Amendment — 2026-08-15: Offline cash is not fiscal success

The original Decision that invoice numbers and fiscalization are server-owned, and that external failure never mutates or renumbers an Invoice, remain in the original text.

ADR 0020 may record offline cash. That cash movement is **not** proof that an invoice was issued or fiscalized. A Ticket must not become fully `POSTED` from a local cash movement alone.

Offline invoice finalization is allowed only if this ADR later defines a pre-reserved number/block **and** an allowed offline fiscal flow. Without that, the payment stays `PENDING_DOCUMENT` / `PENDING_FISCALIZATION`. The guest must not be shown a successful fiscalization that did not happen.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.

## Amendment — 2026-08-15: Invoice recipient snapshot owned against ADR 0021

The original Decision that `recipient_snapshot` is frozen on the issued Invoice, and that external failure never mutates or renumbers an Invoice, remain in the original text.

ADR 0021 owns `CustomerProfile`. The live profile is **not** the invoice recipient. A later profile edit or pseudonymization must not rewrite an issued Invoice.

`LoyaltyEarnSource` binds to `invoice_id` or the fiscal document, not to a mutable Ticket. Tax and fiscal treatment of a loyalty benefit stay this ADR and ADR 0009.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.

## Amendment — 2026-08-15: Incoming supplier document owned by ADR 0023

The original Decision that this ADR governs outgoing sales Invoices only remain in the original text.

ADR 0023 owns the canonical `SupplierInvoice` and AP subledger. Incoming eInvoice transport and recipient fiscalization stay ADR 0024. `legal_entity_id` on the AP invoice is ADR 0023.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.

## Amendment — 2026-08-15: Inbound receive and recipient fiscalization owned by ADR 0024

The original Decision that this ADR governs outgoing sales Invoices only remain in the original text.

ADR 0024 now owns inbound `EInvoiceExchangeRecord`, MANUAL and API receive, recipient fiscalization, deadline, and AP handoff. Outgoing Invoice, B2C fiscalization, and issued-eInvoice exchange stay this ADR. Recipient fiscalization of a received supplier eInvoice is not an outgoing sales fiscalization.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.

## Amendment — 2026-08-15: Accounting export of issued Invoice owned by ADR 0025

The original Decision that this ADR governs outgoing sales Invoices only, and that external failure never mutates or renumbers an Invoice, remain in the original text.

ADR 0025 may export an `ISSUED` Invoice as an accounting source. Issued or fiscalized is not accounting exported. racunai.hr must not re-issue the Tablio legal invoice. A technical provider ACK is not `BOOKED_CONFIRMED`.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.

## Amendment — 2026-08-16: Stored-value issue and redemption facts from ADR 0030

The original Decision that this ADR governs outgoing sales Invoices only, and that external failure never mutates or renumbers an Invoice, remain in the original text.

ADR 0030 hands issue and redemption facts, classification snapshot, consideration, and issuer legal entity. This ADR still decides the fiscal document.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.

## Amendment — 2026-08-16: Prepayment document facts from ADR 0031

The original Decision that this ADR governs outgoing sales Invoices only, and that external failure never mutates or renumbers an Invoice, remain in the original text.

ADR 0031 hands receive, apply, refund, and reclassify facts plus `PrepaymentDocumentLink`. This ADR still decides whether to issue an advance or final document. Recurring invoices stay out of scope.

This amendment does not change B2C/B2B routing, sequence policy, or outbox atomicity.
