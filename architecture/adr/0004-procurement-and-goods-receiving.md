# ADR 0004: Procurement and Goods Receiving

## Status

Proposed

Amended 2026-08-15: AP invoice lifecycle owned by ADR 0023.
Editorially aligned 2026-08-16: Obsolete AP lifecycle, uniqueness, matching and correction examples replaced by ADR 0023 ownership pointers.

## Date

2026-08-15

## Context

ADR 0002 locked one canonical `Product`, packaging conversion into `base_unit`, and the rule that purchase price is not `Product.price`. ADR 0003 locked the stock ledger, the posting contract, and the movement types `RECEIPT` and `RETURN_TO_SUPPLIER`.

This ADR owns the commercial documents that may call that writer. Without it, receiving would become a raw warehouse adjustment, supplier codes would collide with `Product.sku`, and invoices would be used to move stock.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the procurement domain **before** `apps.procurement`. Physical schema details belong in a later implementation. The semantics below must not change.

The governing rule:

```text
Supplier is the commercial party.
Purchase Order is the intent to buy.
Goods Receipt is evidence of physically received goods.
Supplier Invoice is the financial claim.
Only posted physical inventory documents move stock.
Goods Receipt posts RECEIPT.
Return to Supplier posts RETURN_TO_SUPPLIER.
Purchase Order, Supplier Invoice, and Supplier Credit Note never move stock.
```

Purchase Order, Goods Receipt, and Supplier Invoice stay three documents. Return-to-supplier is a fourth **physical** document. A supplier credit note is financial and does not move stock.

A Goods Receipt may exist without a Purchase Order (emergency local buy). An invoice must not post stock. Replacement goods from a supplier are a new Goods Receipt, not a netted return.

## Decision

### 1. Supplier

Minimal tenant-scoped commercial party:

```text
id
tenant_id
legal_name
display_name
tax_id
country_code
email
phone
is_active
timestamps
```

- A supplier belongs to one tenant.
- The same `tax_id` (OIB) may exist for different tenants. OIB is not an authorization key and is not a global supplier id.
- A deactivated supplier remains on historical documents.
- Posted documents snapshot legal name, display name, tax id, and address if present.
- Supplier is not manufacturer or brand.
- Full address book, contacts, and payment terms are later expansions.

### 2. SupplierProduct

Link of an internal `Product` to a supplier catalogue entry. Grain is not `supplier + product` alone:

```text
tenant + supplier + product + packaging
```

Conceptual fields: `supplier_product_code`, `supplier_product_name`, `preferred_packaging_id`, `is_preferred`, `is_active`.

- `supplier_product_code` is not `Product.sku`. A supplier barcode must not overwrite the internal barcode.
- When not empty, `supplier_product_code` is unique within that supplier (tenant-scoped). The same code must not point at two products.
- Ambiguous import or match is rejected, never resolved with `.first()`.
- Changing the supplier code does not rewrite posted documents.
- Preferred packaging converts into `Product.base_unit`.

Example: internal Coca-Cola 0.25 L, SKU `CC025`. Supplier A code `88342`, crate of 24. Supplier B code `COLA025`, pack of 12.

### 3. Purchase Order

Intent to buy. `SENT` creates no `StockMovement`.

```text
PurchaseOrder
-------------
id
tenant_id
supplier_id
delivery_location_id
status
order_number
ordered_at
expected_at
currency
supplier snapshots
notes
timestamps

PurchaseOrderLine
-----------------
purchase_order_id
product_id
ordered_packaging_id
ordered_quantity
quantity_in_base_unit
agreed_unit_price
price basis
tax classification snapshot
product and packaging snapshots
```

Lifecycle:

```text
DRAFT → SENT → PARTIALLY_RECEIVED → RECEIVED
                 ↓
              CANCELLED
```

- `DRAFT` is editable. After send, changes are controlled and audited.
- Agreed line price is not `Product.price`. Currency lives on the PO and must not change after `SENT`.
- Delivery location is a `BusinessLocation` from ADR 0003.
- Partial receive is allowed. One PO may have many Goods Receipts.
- Cancel does not delete already posted receipts.
- `order_number` is unique per tenant.

### 4. Goods Receipt

Evidence of physically received goods. Optional `purchase_order_id`.

```text
GoodsReceipt
------------
id
tenant_id
supplier_id
location_id
storage_id
purchase_order_id       # optional
receipt_number
supplier_delivery_note
status
occurred_at
posted_at
idempotency_key
supplier snapshots
timestamps

GoodsReceiptLine
----------------
goods_receipt_id
product_id              # required for received stock
packaging_id
received_packaging_quantity
quantity_in_base_unit
purchase_order_line_id  # optional
unordered               # true = does not fulfill a PO line
product / packaging / unit / expected price snapshots
```

Lifecycle is the ADR 0003 contract:

```text
DRAFT → POSTED → REVERSED
```

**v1 storage authority: one destination per Goods Receipt.** The header has `location_id + storage_id`. Every line enters that storage. Split to Bar / Kitchen / Cellar is a later transfer. Do not also put `storage_id` on lines. A future per-line storage model must remove header `storage_id` and validate each line on its own. Never both without a single authority.

A line that represents received stock **must** have a `Product`. Non-product charges belong on the invoice, not on the Goods Receipt.

Posting rules:

- Only `POSTED` Goods Receipt creates `RECEIPT` movements, through the ADR 0003 writer, in the same transaction.
- Each movement points at the Goods Receipt and its line.
- Packaging converts to `base_unit` before the ledger. Excess quantity precision is rejected (ADR 0002 / 0003).
- Posted receipt is not edited or deleted. Error = linked reversal + new correct receipt.
- Goods Receipt without a Purchase Order is allowed.
- One receipt, one tenant. Location and storage belong to that tenant. `location_id` matches `StorageArea.location_id` (ADR 0003).
- `is_stock_tracked=false` may appear on the document and produces no movement.
- Idempotency includes payload check (`409` on same key, different content). Unique `(tenant, document_type, document_id, posting_generation)`. `posted_at` is server-set. No partial post.

Example: 2 crates × 24 pieces → document `2` packagings → ledger `RECEIPT +48 piece`.

`receipt_number` is unique per tenant.

### 5. Partial delivery and line resolution

Receipt, return, and reversal are three quantity facts. A return must not shrink “how much was delivered”.

Per Purchase Order line:

```text
ordered_qty_base
gross_received_qty_base          # posted GR lines linked to this PO line
reversed_receipt_qty_base        # reversals of those GR lines (wrong GR)
returned_qty_base                # posted ReturnToSupplier against those GR lines
net_received_qty_base            # gross − reversed
accepted_or_retained_qty_base    # net_received − returned; derived if needed
remaining_qty_base               # ordered − net_received
resolution                       # OPEN | FULFILLED | CLOSED_SHORT
```

- Reversing a wrong Goods Receipt undoes the receipt. That is not a return.
- A return does not erase the proof that goods were delivered.
- A return must not automatically reopen a Purchase Order. Reordering or replacement is a separate commercial decision.
- Example: ordered 24, received 24, returned 2 damaged → the PO line stays `FULFILLED`; there is a return of 2 and optionally a new order or replacement Goods Receipt.
- `PARTIALLY_RECEIVED` when any posted Goods Receipt exists and not all lines are resolved.
- `RECEIVED` / `FULFILLED` uses `net_received` versus ordered (or `CLOSED_SHORT`), not retained quantity after returns.
- Closing short is an explicit, audited action. It does not create stock and must not hide posted receipts.
- A later Goods Receipt against a `CLOSED_SHORT` line is rejected unless the close is reversed (reopen is a hook).
- Unordered extras on a PO-backed Goods Receipt are allowed only when marked `unordered=true`. They do not fulfill any PO line.

### Goods receipt reversal with linked returns

A posted goods receipt MUST NOT be reversed while any posted,
non-reversed supplier return line references that receipt or one of
its receipt lines.

The user MUST reverse all such supplier returns before reversing the
goods receipt. Reversing a supplier return preserves the original
return document and delivery proof and posts compensating inventory
movements; it MUST NOT delete or rewrite the original return.

The system MUST NOT automatically cascade a goods receipt reversal
into linked supplier return reversals.

The validation and posting of the goods receipt reversal MUST execute
in the same database transaction. The receipt, its lines, and linked
posted return lines MUST be locked before eligibility is evaluated.
If an active linked return is found, posting MUST fail without creating
inventory movements or changing purchase-order fulfillment quantities.

Posting a supplier return MUST lock the referenced goods receipt and
its referenced receipt lines before validating and posting the return.

Goods receipt reversal and supplier return posting MUST acquire locks
in the same deterministic order: goods receipt, receipt lines, and
then supplier return lines.

After the goods receipt lock is acquired, supplier return posting MUST
fail if the receipt is already reversed or is being reversed.

### 6. Over and under

Physical truth wins for stock. Commercial variance is not a reason to refuse the ledger.

- **Under:** remaining stays open, or the operator `CLOSED_SHORT`.
- **Over:** posted Goods Receipt still moves the **received** base quantity. Do not silently cap the ledger at ordered quantity.
- v1 allows over-receipt. A later tolerance table may warn or require a reason.
- Over-receipt versus the PO is a match exception for the invoice, not a warehouse denial.
- A Goods Receipt without a PO has no ordered quantity.

Never fix over or under by editing a posted Goods Receipt.

### 7. Money and price basis

The quantity ledger stays quantity (ADR 0003). Money is explicit.

- Decimal, never float. Excess money decimals are rejected, not silently rounded.
- Currency is an ISO 4217 code. PO and invoice currency must not change after post.
- Line price must state **price basis**. Never infer piece versus kilogram versus crate.

```text
price_quantity = 1
price_packaging = crate
unit_price = 18.40 EUR
```

or

```text
price_quantity = 24
price_unit = piece
unit_price = 0.766666 EUR
```

- PO agreed price is commercial intent in PO currency, with that basis.
- Goods Receipt may snapshot expected unit price and basis from the PO line for matching. Receipt posting does not write valuation layers.
- Supplier Invoice carries claimed unit price, basis, and original supplier amounts. Price difference is an AP / purchase-price-variance hook, not a `StockMovement`.
- Line net, tax, and gross totaling belong to the Tax / Invoice ADR. This ADR keeps original supplier amounts and price basis.
- Landed-cost allocation onto goods is a later valuation ADR. Do not put `unit_cost` on `Product`.

### 8. Purchase tax snapshot

PO, Goods Receipt, and invoice lines snapshot **purchase** tax classification (input VAT), not a live rate from Product.

Rate tables, recoverability, and date-effective PDV belong to the Tax ADR. Product category or consumption-tax classification is not purchase-tax proof. The invoice is the financial tax document. The Goods Receipt is not a tax invoice.

### 9. Return to supplier

Physical document `ReturnToSupplier`. Posts `RETURN_TO_SUPPLIER` through the ADR 0003 writer.

```text
DRAFT → POSTED → REVERSED
```

- v1: every return line **must** reference a Goods Receipt line. Returnable quantity is GR line quantity − GR reversals − already returned against that line.
- A posted, non-reversed return line blocks reversal of the referenced Goods Receipt. Reverse those returns first. Return reversal posts compensating movements; it does not delete or rewrite the original return or delivery proof.
- Posting a return locks the referenced Goods Receipt and its referenced receipt lines first, in the same order as Goods Receipt reversal, and fails if that receipt is already reversed or is being reversed.
- Physical stock is posted from the **currently chosen** return storage. It need not be the original receipt storage (goods may have been transferred).
- Product, tenant, and base unit must match the original Goods Receipt line.
- `on_hand` at the return storage may go negative under ADR 0003. Do not invent lot or serial tracking here.
- Return does not edit the original Goods Receipt and does not reopen the Purchase Order.
- A supplier credit note is not this document and does not move stock.
- `is_stock_tracked=false` documented return produces no movement.
- Replacement goods are a **new Goods Receipt**. Linking replacement to return is a later hook. Net-only booking is rejected; it loses the physical-flow audit.

The same ADR 0003 posting locks apply: idempotency with payload, unique posting generation, linked reversal, atomic post, server `posted_at`.

### 10. Supplier Invoice and match allocation

```text
Supplier Invoice is a separate financial document.
It never moves stock and never creates or changes a Goods Receipt.

ADR 0023 owns its fields, lifecycle, matching, approval,
APOpenItem, corrections and supplier payment.
```

A posted Supplier Invoice remains `POSTED` and is not `VOIDED`. Correction is a `CREDIT_NOTE`, `DEBIT_NOTE`, or other compensating AP document under ADR 0023. That correction does not move stock.

**Non-product lines.** An invoice may contain freight, handling, service, deposit, rounding, and other non-stock charges. `product_id` is optional. Such a line never produces a movement, must not force a fake Product, and may stay unmatched or be amount-matched to a PO charge. Landed-cost spread onto goods is a later valuation ADR.

**Match allocation.** ADR 0023 owns matching, legal-entity alignment of PO, Goods Receipt, and invoice, provisional reservation, and committed allocation. This ADR does not define allocation lifecycle.

A non-normative link example: an invoice line may later point at a Purchase Order line and a Goods Receipt line. The link does not change received quantity.

Stock-safety rules that remain here:

- Changing a match must not change the Goods Receipt or any stock movement.
- Auto-match with several equally good candidates stays `AMBIGUOUS`. Never `.first()`.
- Invoice-before-receipt and receipt-before-invoice may both exist as documents. Only physical documents move stock.
- Posted invoice and credit note never create `RECEIPT` or `RETURN_TO_SUPPLIER` movements.
- Stock does not wait for the invoice.

### 11. Invoice number and snapshots

Supplier-invoice number normalization, uniqueness and duplicate detection are owned by ADR 0023.

A posted document number does not become available again because of a credit note, debit note, or compensating document.

Posted documents snapshot supplier, product, packaging, unit, tax classification, price basis, and original amounts. Live rename of supplier or product must not rewrite history.

`supplier_id`, `product_id`, `location_id`, or document id alone never authorizes. Every query is tenant-scoped.

## Rejected alternatives

- Receiving stock by marking a Purchase Order `RECEIVED` without a Goods Receipt.
- Invoice- or credit-note-driven stock.
- “Only a posted Goods Receipt moves inventory” as the sole rule (it contradicts return).
- Subtracting returns from received quantity and automatically reopening the PO.
- Netted replacement (return minus new goods as one delta).
- Header and line `storage_id` both authoritative.
- Per-line storage in v1.
- Editing a posted Goods Receipt to fix quantity or price.
- Capping ledger quantity at ordered quantity.
- Using `Product.sku` or product barcode as the supplier code.
- `supplier + product` as the only SupplierProduct grain.
- Resolving an ambiguous supplier code or invoice match with `.first()`.
- Treating OIB as a global supplier id.
- Putting purchase price or cost on `Product`.
- Implicit price basis (piece versus packaging).
- Float money or silent money rounding.
- Creating a fake Product so a freight line can be entered.
- Return via unlinked `ADJUSTMENT` or `WASTE`.
- Requiring return storage to equal the original Goods Receipt storage.
- A credit note that silently deducts stock.
- POS or warehouse clients converting packaging themselves.
- Reversing a Goods Receipt while a posted, non-reversed supplier return still references it.
- Automatically cascading a Goods Receipt reversal into linked supplier return reversals.
- Locking only existing return lines when reversing a Goods Receipt (a concurrent new return can still post).

## Consequences

### Positive

- Physical flow and financial matching stay auditable and independent.
- Receipt, return, and reversal remain three quantity facts. PO fulfillment does not move backwards because damaged goods were later returned.
- v1 receiving has one destination storage; later internal splits use transfer.
- Match allocations prevent double billing without turning the invoice into a stock document.
- Non-stock charges do not pollute the Product master.
- Every price has an explicit unit and basis.

### Negative

- Operators must post a Goods Receipt even for a small local purchase.
- Returns require a link to the original receipt line in v1, which is stricter than “return any Coca-Cola”.
- Invoice matching is more than a foreign key; ambiguous auto-match must stay unresolved.

### Neutral

- First procurement implementation can ship Supplier, PO, Goods Receipt, and Return without payments, e-račun, or landed cost.
- Currency is stored; FX conversion waits for a later ADR.

## Invariants

1. Only posted physical documents move stock: Goods Receipt → `RECEIPT`, ReturnToSupplier → `RETURN_TO_SUPPLIER`. Purchase Order, Supplier Invoice, and Supplier Credit Note never move stock.
2. All stock effects go through the ADR 0003 writer. Same transaction, idempotency with payload, unique posting generation, linked reversal, no partial post.
3. Return does not erase prior receipt and does not automatically reopen a Purchase Order. Reversal of a wrong Goods Receipt is the only way to undo a receipt fact, and only after every posted non-reversed return that references that receipt has itself been reversed. Replacement is a new Goods Receipt.
4. v1: one `location_id + storage_id` on the Goods Receipt header. All lines enter that storage.
5. Invoice matching must not change a Goods Receipt or any stock movement. Ambiguous auto-match stays `AMBIGUOUS`. Allocation lifecycle and legal-entity alignment are owned by ADR 0023.
6. Invoice lines may omit `product_id`. They never move stock and must not invent a Product. A stock Goods Receipt line must have a Product.
7. Money is decimal, never float. Currency is ISO 4217 and immutable after post. Price basis is explicit. Excess decimals are rejected.
8. Supplier-invoice number uniqueness and duplicate detection are owned by ADR 0023. A posted invoice number is not reusable after a credit note, debit note, or compensating document. SupplierProduct grain is `tenant + supplier + product + packaging`. Supplier code is unique per supplier when set. Ambiguous code match is rejected.
9. Packaging converts to `base_unit` on the physical document. The ledger stores base quantity only.
10. Posted Goods Receipt and ReturnToSupplier are corrected through linked physical reversal and, when needed, a new physical document. A posted SupplierInvoice remains `POSTED` and immutable. Its correction is a separate `CREDIT_NOTE`, `DEBIT_NOTE`, or other compensating AP document under ADR 0023.
11. Goods Receipt may exist without a Purchase Order. Over-receipt posts received quantity; it does not cap to ordered quantity.
12. PO `RECEIVED` / line `FULFILLED` uses net received (gross − Goods Receipt reversals), not retained quantity after returns. Every line is fulfilled or explicitly closed short.
13. v1 return line references a Goods Receipt line and posts from the chosen current storage, which may differ from the receipt storage.
14. Supplier and all procurement documents are tenant-scoped. The same OIB across tenants is not the same supplier.
15. `is_stock_tracked=false` lines document but do not move stock.
16. Posted documents snapshot commercial identity. A later rename does not rewrite history.
17. A Goods Receipt must not be reversed while any posted, non-reversed supplier return line references it or one of its lines. Goods receipt reversal and supplier return posting lock in the same order: goods receipt, receipt lines, then supplier return lines. Validation and posting share one transaction. There is no automatic cascade.

## Future implementation acceptance

Later `apps.procurement` MUST cover at least:

- GR + active posted return → GR reversal rejected
- reversed return → GR reversal allowed
- draft/cancelled return → does not block GR reversal
- return linked to only one line → blocks reversal of the whole GR
- concurrent return post + GR reversal → only one operation may succeed
- rejected reversal creates no movements and does not change PO fulfillment
- return reversal preserves the original document, references, and delivery proof
- no automatic cascade reversal

## Follow-up ADRs

```text
Recipes and Production
POS Sales and Sale Actions
Modifiers
Bundles and Promotions
Tax Model
Invoices and Fiscalization
Barcode Generation and Label Printing
```

Payments, accounting postings, landed cost, PDV calculation, and e-račun stay out of this ADR.

The next domain ADR should define **Recipes and Production**, posting `PRODUCTION_OUT` and `PRODUCTION_IN` through the ADR 0003 writer.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0023: Supplier Invoices and Accounts Payable](0023-supplier-invoices-and-accounts-payable.md)

## Out of scope

This ADR does not define:

- sales invoices or fiscalization
- recipe, production-batch, or yield schema
- POS tickets or pricing
- lots, expiry, or serial tracking
- inventory valuation or FIFO / LIFO / weighted average
- landed-cost allocation onto goods
- payments, bank, or accounting journals
- purchase VAT rate tables or recoverability math
- e-račun / UBL
- supplier portal or EDI
- blanket or standing orders
- approval workflows
- manufacturer or brand as a separate entity
- per-line Goods Receipt storage
- over-receipt tolerance tables
- linking a replacement Goods Receipt to a return

## Amendment — 2026-08-15: AP invoice lifecycle owned by ADR 0023

The original Decision that Purchase Order, Goods Receipt, and Supplier Invoice stay three documents, and that an invoice never moves stock, remain in the original text.

ADR 0023 owns the AP lifecycle, bank accounts, `InvoiceMatchAllocation` (`PROVISIONAL` / `COMMITTED` / `RELEASED`), legal-entity alignment of PO, GR, and invoice, approval snapshot, and `POSTED` creating exactly one `APOpenItem`. Uniqueness includes `document_type` and the merged supplier cluster.

A posted AP invoice is not `VOIDED`. Correction is a credit note, debit note, or compensating document. The invoice must not increase received quantity or create a Goods Receipt.

This amendment does not change Goods Receipt posting, ReturnToSupplier, or ADR 0003 stock movements.

Decision 10, Decision 11, and Invariants 5, 8, and 10 were editorially aligned on 2026-08-16. They no longer list `VOIDED/REVERSED`, the obsolete invoice field sketch, or `(tenant, supplier, invoice_number)` as a canonical uniqueness key.
