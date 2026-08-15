# ADR 0023: Supplier Invoices and Accounts Payable

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize an AP product, a bank adapter, an OCR vendor, or application code.

## Date

2026-08-15

## Context

ADR 0004 owns Purchase Order, Goods Receipt, and ReturnToSupplier, and forbids an invoice from moving stock. ADR 0009 owns the tax model. ADR 0010 owns the outgoing sales Invoice. ADR 0011 owns customer Payment. ADR 0017 owns the permission catalog.

Without this ADR, a received supplier PDF would become a payable, OCR would overwrite an approved invoice, two uploads would post the same bill, an invoice would inflate a Goods Receipt, a dispute would unpost the obligation, and a bank timeout would look like a failed payment.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the AP domain **before** implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

Cite as legal duties, not as a complete compliance program: [Porezna uprava – issuing, receiving and fiscalizing eInvoices](https://porezna-uprava.gov.hr/hr/izdavanje-i-primanje-eracuna-i-fiskalizacija-eracuna/8047) (exchange is not recipient fiscalization), [EUR-Lex – VAT invoice requirements](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX%3A62021CJ0247), [EUR-Lex – Directive 2011/7/EU](https://eur-lex.europa.eu/eli/dir/2011/7/oj/eng) (late-payment duties; this ADR does not lock a universal day count).

The governing rule:

```text
SupplierInvoice      = what the supplier claims
GoodsReceipt         = what was actually received          (ADR 0004)
InvoiceMatch         = link of invoice to PO / receipt
InvoiceApproval      = internal decision to accept
APOpenItem           = the actual open payable
SupplierPayment      = settlement of that payable
```

```text
Received document ≠ AP obligation
APPROVED ≠ POSTED
POSTED ≠ DISPUTED / HELD
Tablio POSTED ≠ racunai.hr journal
OCR extract ≠ canonical value
Invoice ≠ Goods Receipt
PRO_FORMA ≠ APOpenItem
One posted document = one APOpenItem
```

Paper, PDF, manual entry, and eInvoice enter the **same** `SupplierInvoice` workflow after ingest.

```text
Informacijski posrednik = primitak i fiskalizacija eRačuna   (ADR 0024)
Tablio AP               = provjera, approval i obveza        (this ADR)
racunai.hr              = knjiženje i izvještavanje          (ADR 0025)
```

## Decision

### 1. Ownership versus 0004, 0024, and 0025

ADR 0004 still owns Purchase Order, Goods Receipt, ReturnToSupplier, and the rule that invoices never move stock. This ADR owns the AP lifecycle, bank accounts, match lock, approval, `APOpenItem`, and supplier payment.

ADR 0024 later owns intermediary receive, XML and envelope, recipient fiscalization, technical acknowledgements, and `EInvoiceExchangeRecord`. This ADR only says that after ingest every source uses the same AP workflow. Tablio must not assume fiscalization just because an XML was uploaded.

Reserved for ADR 0024 (not written here): `e_invoice_intermediary_mode` `MANUAL` | `API`; one active mode per **legal entity** and intermediary; controlled MANUAL↔API drain.

ADR 0025 later owns GL mapping, period lock, `AccountingExportBatch`, `AccountingOutboxMessage`, and the racunai.hr adapter. `POSTED` here creates the AP subledger item. It does **not** mean the invoice is booked in racunai.hr.

Reserved for ADR 0025 (not written here): `accounting export = NOT_EXPORTED | PENDING | ACKNOWLEDGED | REJECTED | UNKNOWN`; frozen export batch; outbox; no API inside AP posting; timeout `UNKNOWN`; `REJECTED` does not void `APOpenItem`; racunai.hr must not silently edit Tablio’s invoice.

### 2. Supplier, legal entity, and bank accounts

```text
Supplier
--------
tenant_id
supplier_id
legal_name
tax_identifier
registration_identifier
country
status
version
```

```text
ACTIVE
BLOCKED
INACTIVE
MERGED
```

A rename, address change, or bank change must not rewrite old invoices. Each invoice stores a frozen `supplier_snapshot` (name, address, tax number, country, identifier used on the document).

Every `SupplierInvoice` has `tenant_id` and `legal_entity_id`. The invoice belongs to exactly one legal entity. PO, GR, and invoice in a match must share that legal entity. Functional currency and tax identity come from it. After `RECEIVED` the invoice cannot move to another legal entity. Location or cost-center allocations do not change who owes the payable.

```text
SupplierBankAccount
-------------------
supplier_id
iban / account_number
bic
country
status
verification_status
valid_from
valid_until
version
```

```text
UNVERIFIED
VERIFIED
SUSPENDED
RETIRED
```

An invoice or email must not silently change the IBAN. Activating an IBAN needs an audited master change, proof, maker-checker, and a warning if the invoice asks for a new or unverified account. The person who changes an IBAN must not also verify it **and** release payment to that IBAN.

### 3. Supplier merge

Merge is scoped to tenant and legal entity. It needs an explicit survivor and source. Maker-checker is required if either side has open items or bank accounts.

- Posted invoices keep the original `supplier_snapshot`.
- `APOpenItem` may point at the survivor for **future** payment. The historical supplier id stays in audit.
- Duplicate checks look at the whole merged supplier cluster.
- Bank accounts are **not** auto-activated on the survivor.
- Two obligations are not summed or deleted.
- Split is not a simple `Undo`.

### 4. SupplierInvoice

```text
SupplierInvoice
---------------
tenant_id
legal_entity_id
supplier_id
document_type
supplier_invoice_number
issue_date
supply_date
received_date
due_date
currency
gross_total
tax_total
net_total
supplier_snapshot
source
document_status
dispute_status
payment_hold_status
version
```

```text
INVOICE
CREDIT_NOTE
DEBIT_NOTE
```

`PRO_FORMA` may be a non-posting supporting document and a payment-proposal basis. It must not create an `APOpenItem`.

All entered document amounts are **non-negative**. `document_type` sets AP polarity: `INVOICE` and `DEBIT_NOTE` increase the payable; `CREDIT_NOTE` decreases it. A negative `gross_total` in the payload is rejected, or normalized only through an explicit validation rule. A credit note of `100.00` always reduces the payable by 100.

Dispute is not a `document_status`:

```text
document_status:
DRAFT | RECEIVED | VALIDATING | MATCHING
| AWAITING_APPROVAL | APPROVED | POSTED
| REJECTED | CANCELLED_DRAFT

dispute_status:
NONE | OPEN | PARTIALLY_RESOLVED | RESOLVED

payment_hold_status:
NONE | HELD | RELEASED
```

A posted invoice may be `POSTED` + `OPEN` + `HELD` at once. Dispute does **not** undo posting.

`POSTED` is immutable. Correction is a credit note, debit note, or compensating document. It is not an edit and not ADR 0004 `VOIDED` of a posted AP invoice.

Dates stay separate: issue, supply or service, received, due, posting, tax recognition, and the later ADR 0024 fiscalization date. No single `invoice_date` for all.

### 5. Artifact, extraction, and duplicates

```text
SupplierInvoiceArtifact
-----------------------
source_type
content_hash
mime_type
received_at
storage_reference
extraction_status
```

The source document is immutable, hashed, and access-controlled. OCR or AI is not authority. Keep `extracted_value` ≠ `validated_value`. An operator or a structured eInvoice confirms the canonical value. A later extraction model must not rewrite an approved invoice.

```text
UNIQUE (
  tenant_id,
  legal_entity_id,
  supplier_id,
  normalized_invoice_number,
  document_type
)
```

Year or series enters the key only by an explicit rule. Duplicate detection also walks the merged supplier cluster. Fuzzy duplicate (amount, date, hash, eInvoice id, IBAN) warns and **blocks automatic posting** until resolved. It must not auto-reject a valid invoice.

Header and line totals must reconcile with decimal arithmetic and one document currency:

```text
sum(line net) + explicit header charges
= header net

sum(line tax)
= header tax

header net + header tax + rounding adjustment
= header gross
```

Rounding adjustment is explicit and tolerance-capped. Freight, packaging, and other charges are not a hidden remainder. Mismatch above tolerance blocks posting. A tolerated mismatch stays audited and does not rewrite the source document.

### 6. Lines, match, and tolerances

```text
SupplierInvoiceLine
-------------------
invoice_id
description_snapshot
supplier_item_reference
quantity
unit
unit_price
net_amount
tax_category
tax_rate
tax_amount
gross_amount
cost_allocation
```

A line may link a Purchase Order line, Goods Receipt line, internal product, warehouse, location, cost center, expense mapping, or asset candidate. The supplier description must not rewrite canonical `Product`.

Three-way for goods: PO ↔ GR ↔ invoice. Two-way for services without a Goods Receipt. Non-PO (utilities, rent) needs reason, cost allocation, owner, stricter approval, and an optional contract. Do **not** invent a Purchase Order after the invoice to pass control. Cross-legal-entity match is rejected.

```text
InvoiceMatchAllocation
----------------------
invoice_line_id
purchase_order_line_id
goods_receipt_line_id
matched_quantity
matched_net_amount
match_version
status
```

```text
PROVISIONAL
COMMITTED
RELEASED
```

- `MATCHING` creates a **provisional** reservation under the lock.
- `POSTED` atomically turns it `COMMITTED`.
- `REJECTED`, `CANCELLED_DRAFT`, or controlled expiry **releases** it.
- Two provisional or committed allocations together must not exceed remaining received quantity.
- A disputed posted invoice does **not** release a committed match.
- A credit note does **not** automatically return Goods Receipt quantity. It only corrects the financial effect.

One invoice may cover many receipts. Many invoices may cover one receipt. Mixed goods and service, and freight extras, are allowed. The invoice must not increase received quantity or create a Goods Receipt.

Versioned tolerances (`quantity`, `unit_price`, `total`, `tax`, `freight`) yield `MATCHED` | `WITHIN_TOLERANCE` | `OUTSIDE_TOLERANCE` | `UNMATCHED`. Tolerance does not rewrite the invoice or the Goods Receipt. Outside tolerance needs reason, owner, approval, and an optional dispute.

### 7. Approval versus POSTED

`InvoiceApprovalRequest` binds the exact invoice version, lines, match, deviations, tax, cost allocation, bank account, amount, and currency.

Approval is consumed **atomically** on the transition to `APPROVED`. That consume writes `ApprovedInvoiceSnapshot` (hash of invoice, match, tax, bank account, and cost allocation). One approval decision cannot be consumed twice.

`POSTED` re-checks that the current hash equals the approved snapshot. Any change after approval returns the invoice to `AWAITING_APPROVAL` and the old approval becomes `EXPIRED`.

`approve_and_post` is allowed. It uses the same snapshot and one transaction.

Maker-checker at least for: bank-account change; large non-PO; above-tolerance; manual tax override; duplicate override; payment release.

`APPROVED` means internal control accepted the invoice. `POSTED` means the AP obligation exists.

Posting is one transaction: lock the invoice; re-check version, approval snapshot, and hashes; confirm it is not already posted; write AP subledger events; create **exactly one** `APOpenItem`; freeze the financial snapshot; mark `POSTED`; commit match allocations. Retry returns the same result. No second obligation.

### 8. APOpenItem, schedule, dispute, and credit

One posted supplier document creates **exactly one** `APOpenItem` in the document currency.

```text
UNIQUE (supplier_invoice_id)
```

```text
APOpenItem
----------
supplier_invoice_id
supplier_id
original_amount
open_amount
currency
due_date
status
```

```text
OPEN
PARTIALLY_PAID
PAID
ON_HOLD
DISPUTED
CLOSED_BY_CREDIT
```

Instalments are children (`APPaymentSchedule`), not separate obligations. They only split due dates. A multi-location invoice is still one payable. A credit note creates its **own** credit open item, which is then allocated onto the invoice item.

Hold and dispute on the item are **projections** of `payment_hold_status` and `dispute_status`. They do not replace `document_status = POSTED`. `open_amount` is derived from posting, allocations, credits, and reversals. No manual balance overwrite.

`APPaymentSchedule` instalments must sum to the open obligation. After posting, a schedule change is audited, keeps history, and must not create or delete the obligation.

`InvoiceDispute` does not delete the invoice and does not unpost it. Hold may exist after posting. The disputed amount is not proposed for payment. The undisputed part may pay if policy allows. A supplier credit is a separate document.

A credit note has its own number and artifact. It references the original when known. It does not edit the original. Polarity decreases the payable by the entered non-negative amount. It has its own tax validation. The same credit is applied once. Excess over the open amount becomes supplier credit, not an unexplained negative invoice.

### 9. Currency and tax

The invoice stores document currency, functional currency from `legal_entity_id`, rate source, rate date, rate value, and a converted snapshot. Decimal arithmetic only. A later FX rate does not rewrite a posted invoice. Payment FX difference is an AP or payment event for ADR 0025.

ADR 0009 owns the tax model. This ADR records input VAT, deductible and non-deductible parts, reverse charge, exemption, jurisdiction, evidence, and rule version. A formally required tax element that is missing blocks automatic posting.

### 10. Payment proposal, reservation, UNKNOWN, allocation, and advances

A due invoice does not by itself create a payment.

```text
SupplierPaymentProposal
-----------------------
tenant_id
supplier_id
proposed_allocations
payment_account
beneficiary_account_version
amount
currency
execution_date
status
```

```text
DRAFT
VALIDATING
AWAITING_APPROVAL
APPROVED
SUBMITTED
UNKNOWN
EXECUTED
REJECTED
CANCELLED
RECONCILED
```

Before approval, re-check open balance, hold or dispute, IBAN version and verification, duplicate pay, currency, already-reserved allocations, and the approval threshold.

```text
APPaymentReservation
--------------------
payment_proposal_id
ap_open_item_id
reserved_amount
status
```

Two proposals must not reserve the same open amount. `APPROVED` or `SUBMITTED` reserves. `REJECTED`, `CANCELLED`, or controlled expiry releases. `UNKNOWN` does **not** auto-release. The bank may have paid.

Bank timeout: `SUBMITTED → UNKNOWN`. Do not send a new payment blindly. Reconcile by a stable bank or provider reference. The same instruction keeps the same idempotency key. The open item is not paid until there is sufficient execution proof. `EXECUTED` is not `RECONCILED`.

```text
APPaymentAllocation
-------------------
supplier_payment_id
ap_open_item_id
amount
currency
allocation_type
```

One payment may cover many invoices. Many payments may cover one invoice. Partial pay, supplier credit, and advance are allowed. Bank fee is **separate** from invoice settlement. Allocations must not exceed the proven executed amount.

```text
SupplierAdvance
---------------
supplier_id
amount
currency
source_payment
remaining_amount
status
```

An advance is not an expense and not a final paid invoice. Later it is allocated to the final invoice. A proforma may fund a proposal. It still creates no `APOpenItem`.

### 11. Location, due dates, and closed period

One invoice may allocate lines across locations, warehouses, cost centres, projects, and expense categories. It remains **one** document and **one** legal-entity payable. Line allocations must sum to the canonical line amount.

Original payment terms and due-date snapshot stay. A later supplier-master change does not rewrite them. Late-payment interest or fees are a separate document or AP event, not a silent principal change.

A late invoice for a closed GL period keeps the original issue and supply dates. The posting date respects the **open** period. Late-entry needs a reason and approval. Dates are not rewritten to pass the lock. The concrete period lock stays ADR 0025.

### 12. Permissions

ADR 0017 owns the catalog. This ADR adds:

```text
supplier.view
supplier.manage
supplier.bank_account_manage
supplier.bank_account_verify
ap.invoice_create
ap.invoice_validate
ap.invoice_match
ap.invoice_approve
ap.invoice_post
ap.invoice_dispute
ap.duplicate_override
ap.payment_prepare
ap.payment_approve
ap.payment_submit
ap.payment_reconcile
```

### 13. Audit

Keep at least: artifact hash; extracted versus validated values; supplier snapshot; legal entity; bank version and verification; duplicate result and override, including the merged cluster; match allocations and their status; header and line total check; tolerances; approval snapshot hash; AP posting; the single open item; dispute and hold columns; credit allocation; proposal, submit, and provider reference; `UNKNOWN` recovery; payment allocation and reconciliation; cost and location allocation; merge mapping; actor, membership episode, and then-current permissions.

### 14. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Ingest (manual, PDF, or eInvoice reference) creates a `SupplierInvoice` and no `APOpenItem`.
- `PRO_FORMA` cannot become `POSTED` or an `APOpenItem`.
- OCR extract is not used as canonical without validation. Changing the extractor does not rewrite an approved invoice.
- Duplicate `(tenant, legal_entity, supplier, number, type)` is rejected. A fuzzy duplicate blocks auto-post, not auto-reject.
- An invoice cannot increase Goods Receipt quantity or create a Goods Receipt.
- Two invoices cannot allocate the same remaining Goods Receipt quantity (provisional plus committed).
- A fake Purchase Order created after the invoice to pass match is rejected.
- An outside-tolerance match cannot auto-post.
- An approval bound to an old invoice version is `EXPIRED` and cannot be consumed.
- `APPROVED` without posting creates no `APOpenItem`.
- Posting retry returns the same `APOpenItem`. No second obligation.
- Manual overwrite of `open_amount` is rejected.
- A credit note does not edit the original posted invoice. The same credit applied twice is rejected.
- Excess credit becomes supplier credit, not an unexplained negative invoice.
- A later FX rate does not rewrite a posted invoice.
- Missing required tax evidence blocks automatic posting.
- Due date alone does not create a payment.
- Two proposals cannot reserve the same open amount.
- An `UNKNOWN` bank result does not release the reservation and does not mark the item paid.
- A blind second submit of a payment instruction is rejected. The same idempotency key is reused.
- Allocations cannot exceed the proven executed amount.
- Bank fee is not netted into invoice settlement.
- An IBAN taken from an emailed invoice does not change `SupplierBankAccount`.
- The same actor cannot change an IBAN, verify it, and release payment to it.
- A supplier rename does not change a posted invoice snapshot.
- A late invoice keeps original issue and supply dates. The posting date is in an open period. A silent date rewrite is rejected.
- Line cost allocations must sum to the line amount.
- Tablio `POSTED` does not set accounting export to `ACKNOWLEDGED`.
- No outbound accounting API inside the AP posting transaction.
- A `POSTED` invoice can be `DISPUTED` and `HELD` without losing posted status.
- An invoice cannot match a Purchase Order or Goods Receipt of another legal entity.
- `CREDIT_NOTE 100.00` reduces the payable by exactly 100. A negative payload does not create a double minus.
- Line or header mismatch above tolerance blocks posting.
- A rejected invoice releases its provisional Goods Receipt allocation.
- A posted invoice keeps its committed match while disputed.
- One `SupplierInvoice` can have only one `APOpenItem`.
- An edit after approval voids the approved snapshot and requires a new approval.
- A duplicate invoice through two merged Supplier rows is still rejected.
- Supplier merge does not activate an unverified IBAN.

## Rejected alternatives

- A received invoice immediately as an AP obligation.
- OCR as the canonical value.
- An invoice rewriting Supplier master or IBAN.
- A second posting via another upload.
- An invoice increasing Goods Receipt quantity.
- A fake Purchase Order after the invoice.
- Auto-post above tolerance.
- An approval that survives an invoice edit.
- A mutable AP balance.
- Editing a posted invoice.
- A credit note overwriting the original.
- Bank timeout as a failed payment.
- Blind payment retry.
- `UNKNOWN` releasing the reservation.
- Bank fee netted into invoice settlement.
- Proforma as a final `APOpenItem`.
- Rewriting original dates to pass a period lock.
- Tablio `POSTED` meaning racunai.hr success.
- An API call inside the AP posting transaction.
- `DISPUTED` replacing `POSTED` in one status column.
- Moving a received invoice to another legal entity.
- Matching a Purchase Order or Goods Receipt of a different legal entity.
- Signed amounts that double-minus a credit.
- A hidden header or line remainder.
- `MATCHING` permanently blocking Goods Receipt quantity without `POSTED`.
- More than one `APOpenItem` per document.
- Schedule rows as separate payables.
- Auto-activating bank accounts or summing obligations on supplier merge.
- Writing ADR 0024 or ADR 0025 in this change.

## Consequences

### Positive

- A received document cannot become a payable without match, approval, and idempotent `POSTED`.
- Dispute and hold cannot hide that the obligation already exists.
- Two invoices cannot bill the same remaining received quantity.
- A bank timeout cannot silently free the reserved amount or invent a second payment.

### Negative

- `MATCHING` holds Goods Receipt quantity only provisionally; long-lived drafts must expire or be rejected.
- Supplier merge cannot silently combine payables or activate IBANs.
- Accounting export and eInvoice fiscalization wait for later ADRs.

### Neutral

- Documentation can merge without a bank adapter, OCR vendor, or racunai.hr client.
- ADR 0004 still owns physical receiving. ADR 0009 still owns the tax model.
- ADR 0024 and ADR 0025 stay reserved roadmap entries.

## Invariants

1. `SupplierInvoice` ≠ Goods Receipt ≠ `APOpenItem` ≠ `SupplierPayment` ≠ accounting export.
2. A received document is not an obligation. `APPROVED` is not `POSTED`. `POSTED` is not `DISPUTED` or `HELD`.
3. One posted document creates exactly one `APOpenItem`. Retry returns the same item.
4. Amounts are non-negative. Polarity comes from `document_type`. Header and line totals reconcile.
5. Match allocations are `PROVISIONAL` until `POSTED`. Reject releases. Dispute does not. A credit note does not return Goods Receipt quantity.
6. Tenant and legal entity bind the invoice. PO, GR, and invoice in a match share that legal entity. After `RECEIVED` the legal entity does not move.
7. OCR is not canonical. An invoice does not rewrite Supplier master or IBAN. Duplicate checks include the merged supplier cluster.
8. Payment `UNKNOWN` does not release the reservation and does not mark the item paid.
9. Tablio `POSTED` does not mean racunai.hr acknowledgement. No accounting API inside the posting transaction.
10. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Incoming eInvoices and Recipient Fiscalization
Accounting Posting and Export
Reporting, Analytics and Historical Snapshots
Audit Trail, Data Retention and Privacy
```

Do not implement an intermediary adapter, racunai.hr export, or a bank API from this ADR.

## See also

- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)
- [ADR 0009: Tax Model](0009-tax-model.md)
- [ADR 0010: Invoices and Fiscalization](0010-invoices-and-fiscalization.md)
- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0017: Staff Identity, Roles and Operator Authorization](0017-staff-identity-roles-and-operator-authorization.md)

## Out of scope

This ADR does not define:

- eInvoice receive or recipient fiscalization (ADR 0024)
- GL, journal, or racunai.hr export (ADR 0025)
- AP aging dashboards (ADR 0026)
- retention or privacy execution (ADR 0027)
- a bank API adapter
- an OCR or AI vendor
- a supplier portal
- purchase budget planning
- payroll or staff expenses
- a detailed fixed-asset register
- landed-cost valuation
- the legal-entity master beyond the invoice, PO, and GR binding
- exact tolerance amounts, late-payment day counts, or “large non-PO” thresholds
- POS screen layout
