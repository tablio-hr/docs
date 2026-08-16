# ADR 0011: Payments and Settlement

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

Viva.com card charge must not be implemented until the official Android app-to-app contract is confirmed: invoke extras, required result fields, whether merchant reference is returned, last-transaction retrieval, how result authenticity is proven, process-death and POS reopen behaviour, and whether refund/void uses the same local integration. If authenticity cannot be verified enough, this ADR stays `Proposed` and card charge is not activated.

A production POS path also remains blocked until ADR 0009 (tax formula and rounding) and ADR 0010 (invoice/fiscal adapters and payment fiscal-code mapping) have their implementation blockers closed.

This ADR does not authorize a backend Viva card API or a payment engine in application code.

## Date

2026-08-15

## Context

ADR 0006 owns the POS Ticket and left FX and tenders to a Payments ADR. ADR 0009 owns tax calculation. ADR 0010 owns the outgoing Invoice and requires a fiscal payment-method snapshot derived from finalized tender facts, including the B2B cash/card exception versus a transaction account.

Without this ADR, a processor fee would shrink the invoice, a refund would rewrite a paid invoice to unpaid, an unpaid bank invoice would have no fact to derive fiscal code `T`, Viva could capture a card after the ticket changed, and a lost Android result would start a second charge.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the payment domain **before** adapters. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

Tablio has a commercial agreement with Viva.com. Viva is the first supported `CARD` provider via **local Android app-to-app** on the same POS terminal. The Payment domain stays provider-neutral. The backend does not initiate or process the card transaction through the Viva API.

The governing rule:

```text
Invoice records what the customer owes.
Payment records money received or returned.
Payment Allocation links money to one or more Invoices.
Payment Instruction records how an unpaid Invoice is expected to be paid.
Settlement records when an external provider transfers collected funds.
Fiscal payment method is derived from finalized tender / payment facts.
```

```text
Invoice, Payment, Payment Allocation, and Settlement are separate facts.

A CAPTURED Payment is an immutable financial fact.
Refunds create a new OUTBOUND Payment and settle a credit document;
they do not unpay the original Invoice.

Provider fees and bank settlements never reduce the amount paid by the customer.
Mixed tender consists of separate Payments. The fiscal payment code is
derived by a versioned backend rule and frozen on the issued Invoice.
```

Cite as a duty, not a full technical spec: [Porezna uprava – Napojnica](https://www.porezna-uprava.hr/obrazac_joppd/Stranice/Napojnica.aspx).

## Decision

### 1. Invoice is not Payment

An Invoice exists whether immediately paid, partially paid, mixed, paid later, or never collected. A Payment must not change invoice lines, prices, tax results, stock movements, invoice number, JIR/ZKI, or eInvoice content.

Payment settles an Invoice or a credit Invoice. It never rewrites the commercial or tax facts of that document.

### 2. Concepts

**`PaymentIntent`.** Intent before a final result. Not proof that money was received. The backend owns amount, currency, location, an unguessable one-time `merchant_reference`, and `request_payload_hash`.

**`Payment`.** An actual financial event: tenant, legal entity, location, `INBOUND` or `OUTBOUND`, method, currency, amount, `occurred_at`, status, provider reference, idempotency key. Once `CAPTURED`, it is immutable.

**`PaymentAllocation`.** Links a Payment to an Invoice or a Tip. Append-only. A mistake is corrected by an opposite allocation, not an edit or delete.

**`PaymentInstruction`.** How an unpaid Invoice is expected to be paid (`BANK_TRANSFER`: method, currency, expected amount, `due_at`, payment reference, creditor IBAN, status). Not proof of payment.

**`Settlement`.** When a provider transfers collected funds. Does not change historical invoice paid state. A fee is not a customer discount.

**`Tip`.** Not a Product, Sale Action, tax line, discount, or Payment method. Creates no `SALE`.

**`POSDevice`.** Registered Android POS device (tenant, location, provider, pairing/configuration reference, status). The authenticated device session identifies the device. The backend does not trust an arbitrary client-sent `device_id`.

### 3. v1 methods

`CASH`, `CARD`, `BANK_TRANSFER`, `OTHER` (`OTHER` requires a reason code). Later hooks: voucher, gift card, loyalty, delivery platform, room charge.

The method is a backend enum, not free text. Card brand and provider are not the fiscal method. Visa and Mastercard are both `CARD`. An IBAN credit is `BANK_TRANSFER`. `VIVA_COM` is a provider, not a method.

After a confirmed Payment the method is not edited. A wrong method is corrected by a linked outbound Payment and a new Payment.

### 4. Payment lifecycle — the original stays `CAPTURED`

```text
CREATED → PENDING → AUTHORIZED → CAPTURED
CREATED → FAILED | CANCELLED | EXPIRED
lost Viva result / POS crash → UNKNOWN

Original Payment remains CAPTURED
→ new OUTBOUND Payment
→ reversal_of_payment_id

reversal_status: NONE | PARTIALLY_REVERSED | FULLY_REVERSED
```

Cash has no fake provider lifecycle: `CREATED → CAPTURED`. `AUTHORIZED` is reserved, not received. Only `CAPTURED` is money in. `UNKNOWN` is not `FAILED` and not `CAPTURED`.

A provider void and a provider refund are different provider operations. Both create a **new** financial event. Parallel refunds lock the original Payment. The sum of outbound reversals must not exceed the original available amount.

### 5. Invoice paid state is historical

Do not subtract refunds from the original Invoice.

```text
original_invoice_paid_amount = sum inbound allocations to original Invoice
credit_document_paid_amount  = sum outbound allocations to corrective/credit Invoice
```

A properly paid Invoice stays historically `PAID`. The ADR 0010 credit Invoice creates the obligation to return money. An outbound Payment settles the **credit** Invoice. `net_customer_cash_position` may be derived separately.

A refund without a credit document is allowed only for an explicit non-tax obligation. It must never silently correct a sale.

Derived original states `UNPAID`, `PARTIALLY_PAID`, `PAID`, and `OVERPAID` use inbound allocations only. They are not hand-edited. Overpayment must be explicitly allowed; it must not appear from a race.

### 6. `BANK_TRANSFER` uses Payment Instruction

A `BANK_TRANSFER` Invoice may be issued with a finalized Instruction and no `CAPTURED` Payment. Fiscal code `TRANSAKCIJSKI_RAČUN` (`T`) is derived from the Instruction. A later bank credit creates a Payment and an Allocation. Changing IBAN or reference after issue does not rewrite the Invoice snapshot.

POS B2C cash and card cannot use an Instruction as a substitute for received money.

### 7. Mixed tender and partial failure

There is no method `MIXED`. Each tender is its own Payment. POS v1 uses the invoice currency (ADR 0006).

v1 fiscal mapping:

```text
single CASH           → NOVČANICE
single CARD           → KARTICA
single BANK_TRANSFER  → TRANSAKCIJSKI_RAČUN
single OTHER          → OSTALO
more than one method  → OSTALO
```

The Invoice snapshot stores every tender, the computed fiscal code, the mapping-rule version, and finalized-at. The client cannot send the fiscal code. Later settlement or a provider fee does not change it.

```text
Cash 40 CAPTURED + Card 60 FAILED
```

- The ticket stays `PAYMENT_IN_PROGRESS` / `PARTIALLY_TENDERED`.
- The 40 EUR cash Payment remains unallocated until the Invoice is issued.
- The operator may try another method for the remaining 60 EUR, or return 40 EUR through a linked outbound Cash Payment.
- The cash Payment must not be deleted.
- The Invoice is issued only when the tender set is final. The fiscal code is derived from that final set.

### 8. Payment is not stock or tax

A Payment does not create `SALE` or `RETURN`, does not calculate VAT or consumption tax, and does not change production, bundles, or modifiers. It is not a shortcut to credit an Invoice. A goods return and a money refund are distinct; the link must be explicit.

### 9. Freeze the ticket before starting Viva

Before the POS starts the Viva application the backend must:

1. Lock and finalize ticket content.
2. Freeze amount, currency, routing, and tenders.
3. Validate taxes, KPD, premises, device, and operator.
4. Create a unique PaymentIntent.
5. Set the ticket to `PAYMENT_IN_PROGRESS`.

While payment is in progress, lines, quantities, modifiers, discounts, and recipient cannot change. Another device cannot post or start a new charge. Unlock is allowed only after a clear fail or cancel.

After `CAPTURED`, the backend must issue the Invoice from the **same frozen payload hash**. If local issue still fails after capture, that is a **critical recovery** case. The ticket must not silently return to ordinary draft.

POS B2C `CARD` must be backend-`CAPTURED` before invoice number allocation. `CANCELLED`, `DECLINED`, `FAILED`, and `UNKNOWN` card tenders are not placed on the Invoice as `CARD`.

### 10. Viva.com Android app-to-app

The Viva.com payment application and the Tablio POS application run on the same Android POS terminal.

```text
Frozen ticket + PaymentIntent
→ Tablio POS starts the Viva.com Android application
  with frozen amount, currency, and merchant_reference
→ Viva performs the card charge
→ Viva returns the result to Tablio POS
→ POS sends original allowed Viva fields to the backend
  (not only success=true)
→ backend verifies under a database lock and marks CAPTURED
→ Invoice is issued and fiscalized from the frozen payload hash
```

The frozen request includes `payment_intent_id`, unguessable one-time `merchant_reference`, amount, currency, tenant/location/device, `request_payload_hash`, and `created_at`. The Viva result must return or allow binding to the same merchant reference as far as the official integration allows.

- The backend is the authority for Intent, amount, and currency. POS must not invent a charge amount.
- Viva handles the card and all sensitive card data. Tablio never sees or stores full PAN, CVV, PIN, track data, or sensitive EMV data.
- Only an official Viva success result may become a `CAPTURED` candidate.
- The backend accepts a result only for an open Intent of the same authenticated device.
- An identical result is idempotent. The same provider transaction ID with a different amount, currency, or intent is a security incident.
- One PaymentIntent must not produce two accepted captures. One Viva transaction ID must not settle two invoices.
- The backend stores the raw allowed provider result or its canonical hash for audit.
- Provider fee and Viva settlement do not change the amount the guest paid.
- A later Android provider uses the same Intent / report-result contract.

Allowed stored proof fields include provider transaction id, masked PAN, card brand, authorization code, capture amount, currency, provider timestamps, and result code.

### 11. `UNKNOWN` recovery

If Viva charged the card but Tablio POS closed before reporting to the backend, the Intent becomes `UNKNOWN`, not `FAILED`.

- POS must not automatically start a new charge. A new charge for the same sale is blocked.
- First try to retrieve the result from the Viva application if the official integration supports it.
- The operator is told to check the previous transaction.
- Manual confirm without a Viva reference is forbidden.
- `UNKNOWN` enters a dedicated reconciliation queue.
- Later proof from an official Viva report or a controlled provider is allowed and must include transaction ID, amount, currency, terminal, and time.
- Linking is an authorized backend operation with audit. Only then does the Payment become `CAPTURED` and the Invoice complete.
- An unproven case requires controlled refund or escalation, not `success=true`.
- A late or duplicate POS report must not override a newer confirmed backend state.

### 12. Allocation guards

Tenant and currency of Payment and Invoice must match. The sum of allocations on one Payment must not exceed its `CAPTURED` amount. Allocations are idempotent and taken under a lock on Payment and Invoice. They are not edited or deleted. One Viva transaction ID must not settle more than the captured amount across allocations.

### 13. Tip

```text
Invoice 50 EUR + Tip 5 EUR → Card Payment 55 EUR
Payment allocation → Invoice 50 EUR
Tip allocation     → Tip 5 EUR
```

A Tip stores amount, payment method, operator/recipient, and the linked fiscal invoice. Tip fiscal reporting has its own outbox and status. Mixed tip methods map to `OSTALO`. A failed tip report does not change the Invoice; it gets a statutory retry deadline and a critical alert. Payment surplus is never automatically declared a tip.

### 14. Cash drawer is an append-only ledger

```text
OPENING_FLOAT
CASH_SALE
CASH_REFUND
CASH_IN
CASH_OUT
TIP_IN
COUNT_ADJUSTMENT
CLOSING_DEPOSIT
```

Movements are not edited or deleted. Balance is the sum. `CASH_SALE` references a Payment. A cash refund creates an outbound Payment and a drawer movement atomically. Manual `CASH_IN` / `CASH_OUT` require a reason and an operator. Shift, location, and device are snapshotted. A count variance does not change sales invoices.

### 15. Settlement reconciliation

The provider transaction ID is the primary match key. Multiple candidates must not be resolved with `.first()`. Missing or duplicate identifiers stay `UNMATCHED` or `AMBIGUOUS`. A settlement may be partial and may contain several payouts. Fees, refunds, and chargebacks are separate lines. A chargeback is not a customer refund and does not automatically set the Invoice to `UNPAID`.

Gross captures minus refunds minus fees plus or minus adjustments must explain the payout. An unexplained difference stays open for reconciliation; it is not booked as a fee without evidence. A confirmed Settlement and its lines are immutable.

## Rejected alternatives

- `CAPTURED → REVERSED` on the same Payment row.
- Subtracting refunds so a paid Invoice becomes `UNPAID`.
- Issuing `BANK_TRANSFER` with fiscal `T` and no Payment Instruction.
- Using an Instruction instead of captured cash or card on POS.
- Starting Viva on an unlocked or editable ticket.
- Silently returning a ticket to draft after card capture if issue fails.
- POS inventing the charge amount or sending only `success=true`.
- Timeout becoming automatic `FAILED`, or an automatic second charge.
- Manual “it went through” without a Viva reference.
- Trusting a client-supplied `device_id` over the authenticated session.
- Storing PAN, CVV, PIN, or sensitive EMV data.
- Two invoices settled by one Viva transaction ID.
- Method `MIXED`, or a predominant-amount fiscal code in v1.
- A client-sent fiscal payment code.
- Deleting a cash Payment after a card failure.
- Automatically declaring surplus as a tip.
- `.first()` on ambiguous settlement matches.
- Treating a chargeback as a customer refund that unpays the Invoice.
- Backend-direct Viva card API as the v1 path.
- Implementing Viva charge before the Status checklist is confirmed.
- Amending ADR 0002–0010 in this change.

## Consequences

### Positive

- A processor fee cannot shrink what the guest paid.
- A later refund cannot erase that the original Invoice was paid.
- An unpaid bank invoice still has a fact from which fiscal `T` is derived.
- A card cannot be captured against a ticket that is still being edited.
- A lost Android result cannot silently double-charge.
- Mixed cash-then-failed-card remains auditable.
- Tips and sales stay separate fiscal facts.

### Negative

- Viva charge cannot ship until the Android contract and authenticity checks are confirmed.
- Operators cannot finish a card sale while the Intent is `UNKNOWN`.
- Partial mixed tenders require an explicit second method or an outbound cash return.

### Neutral

- Documentation can merge without a Viva SDK integration.
- A later card provider can reuse Intent, report-result, and POSDevice contracts.
- Incoming AP payments wait for a later ADR.

## Invariants

1. Invoice ≠ Intent ≠ Payment ≠ Allocation ≠ Instruction ≠ Tip ≠ Settlement ≠ drawer movement. The domain is provider-neutral. Payment never rewrites Invoice, tax, stock, or JIR.
2. A `CAPTURED` Payment is immutable. Refund or void creates a new `OUTBOUND` Payment with `reversal_of_payment_id`. The original Invoice stays historically `PAID`. Outbound allocations settle the credit Invoice unless the refund is an explicit non-tax obligation.
3. `BANK_TRANSFER` may issue on a finalized Payment Instruction. Fiscal `T` comes from the Instruction. Cash and card on POS B2C require captured money, not an Instruction.
4. Mixed tender is multiple Payments. Partial failure keeps captured cash. The Invoice waits for a final tender set. v1 mixed fiscal code is `OSTALO`.
5. The ticket is `PAYMENT_IN_PROGRESS` with a frozen payload hash before Viva starts. After capture, the Invoice is issued from that hash. Failed issue after capture is critical recovery, not a silent draft.
6. Viva is Android app-to-app. The backend verifies bound merchant reference and payload under a lock. `UNKNOWN` goes to a reconciliation queue. Only authorized proof with a Viva reference may become `CAPTURED`. No automatic second charge.
7. Allocations and cash-drawer movements are append-only. Settlement is fail-closed on an ambiguous match. Chargeback is not a customer refund.
8. Tip is a separate fiscal fact with its own outbox. Surplus is never auto-declared a tip.
9. Tablio does not store PAN, CVV, PIN, or sensitive EMV data. One Viva transaction ID cannot settle more than one captured amount.
10. Tenant isolation. Ids alone do not authorize. This ADR stays `Proposed` and does not authorize Viva charge until the Status checklist is confirmed. Production POS also waits on ADR 0009 and ADR 0010 blockers.

## Follow-up ADRs

```text
Incoming eInvoice and AP
Partial return / refund
FX / pay-in-another-currency
POS layout
```

Confirm the Viva Android contract, then implement the adapter behind this ADR. The next domain ADR should define **incoming supplier eInvoices and AP**.

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
- [Porezna uprava – Napojnica](https://www.porezna-uprava.hr/obrazac_joppd/Stranice/Napojnica.aspx)

## Out of scope

This ADR does not define:

- implementation of Viva Android invoke extras or a backend Viva card API
- FX or pay-in-another-currency
- gift-card or loyalty ledgers
- payroll tip payout
- incoming supplier payments or AP
- counted-drawer UX chrome
- POS screen layout

## Amendment — 2026-08-15: Cash accountability owned by ADR 0018

The original Decision 14 append-only ledger, “`CASH_SALE` references a Payment”, atomic cash refund, and “a count variance does not change sales invoices” remain in the original text.

ADR 0018 owns `CashAccountableUnit`, day-scoped drawer and wallet sessions, the persistent `SAFE` ledger, two-sided `CashTransfer`, expected cash, count, variance, and business-day close.

```text
TIP_IN remains and is expected cash, not sales.
CLOSING_DEPOSIT is CLOSING_REMOVAL.
COUNT_ADJUSTMENT is not the close variance.
Intra-location cash is TRANSFER_OUT + TRANSFER_IN
in one transaction, same cash_transfer_id.
CASH_SALE lands on the mode-required session
(drawer or the actual operator’s wallet).
SAFE is not a daily session and is not reset at day close.
```

This amendment does not change Payment, allocation, Intent, Instruction, Tip fiscal facts, or Settlement.

## Amendment — 2026-08-15: POSDevice registration owned by ADR 0019

The original Decision 2 “the backend does not trust an arbitrary client-sent `device_id`” and the authenticated-device rule on a Viva result remain in the original text.

ADR 0019 owns POSDevice registration, credential, assignment, `REASSIGNING`, and `EffectiveDeviceConfig`. A registered POS device is not Android-only; Viva remains the first card path on a registered device.

A `SUSPENDED`, `COMPROMISED`, or `RETIRED` device may retry a committed payment result or recover `UNKNOWN` status without a new write. It must not start a new sale, payment, or refund.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.

## Amendment — 2026-08-15: Offline cash and card owned by ADR 0020

The original Decision that `UNKNOWN` is not `CAPTURED`, and that a later provider result binds the same Payment, remain in the original text.

ADR 0020 owns the offline command log. Offline cash is not `POSTED` and not fiscal success. The POS app must not declare a card `CAPTURED`. A later provider result updates the **same** Payment.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.

## Amendment — 2026-08-15: Loyalty points are not a Payment

The original Decision that Payment records settlement, and that later hooks may include loyalty, remain in the original text.

ADR 0021 owns the loyalty ledger. Points are not a Payment, wallet, or tender. Redemption is an ADR 0016 Ticket benefit, not a capture. Gift cards and stored value stay ADR 0030.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.

## Amendment — 2026-08-15: Platform payment claims owned by ADR 0022

The original Decision that Settlement records when an external provider transfers collected funds, and that later hooks may include a delivery platform, remain in the original text.

ADR 0022 owns `ChannelOrder` collection mode. `PLATFORM_COLLECTED` is a claim, not bank settlement. A provider refund or cancel is also a claim. A replay binds the same internal operation and must not create a second Refund. The same order must not be marked paid by both a platform claim and a local capture.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.

## Amendment — 2026-08-15: Supplier payment owned by ADR 0023

The original Decision that this ADR owns customer Payment, Intent, and Settlement, and that incoming AP payments waited for a later ADR, remain in the original text.

ADR 0023 owns `SupplierPaymentProposal`, `APPaymentReservation`, bank `UNKNOWN`, and AP allocation. Customer Payment stays this ADR.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.

## Amendment — 2026-08-15: Accounting export of captured Payment owned by ADR 0025

The original Decision that Invoice, Payment, allocation, and Settlement are separate facts remain in the original text.

ADR 0025 may export a `CAPTURED` customer Payment as an accounting source. Fees and Settlement still do not rewrite the invoice. Payment `UNKNOWN` is not exportable. `BOOKED_CONFIRMED` does not mark the customer paid.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.

## Amendment — 2026-08-16: GIFT_CARD Payment sourced from captured stored-value authorization

The original Decision that Invoice, Payment, allocation, and Settlement are separate facts remain in the original text.

ADR 0030 owns the stored-value ledger. A captured `StoredValueAuthorization` becomes a `GIFT_CARD` Payment. A discount voucher is not a Payment method.

This amendment does not change Payment lifecycle, allocation, Intent, or Settlement.
