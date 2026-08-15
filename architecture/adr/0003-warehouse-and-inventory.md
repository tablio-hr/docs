# ADR 0003: Warehouse and Inventory

## Status

Accepted (2026-08-15)

## Date

2026-08-15

## Context

ADR 0002 locked one canonical, tenant-scoped `Product`, a single `base_unit` for quantity, packaging conversion at the document edge, and the rule that POS does not compute stock deductions.

This ADR makes those rules operational. Hospitality stock is not a number on the product card. It lives in a place, changes only through posted documents, and must survive retries, voids, inventura during service, and later rename of products or storages.

Without a warehouse boundary, procurement, production, and POS would each invent their own quantity table. That would break the Input → Product → Output model.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the warehouse domain **before** `apps.inventory`. Physical schema details belong in a later implementation. The semantics below must not change.

The governing rule:

```text
Product is identity.
Location + storage is place.
The ledger is truth for quantity.
The balance is a same-transaction projection of on_hand.
```

Warehouse never learns what a cappuccino is. It records `−8 g Coffee Beans` and `−120 ml Milk` because a backend document told it to.

## Decision

### 1. Located stock, not stock-on-product

Current quantity is never a field on `Product`.

Authority grain of on-hand stock:

```text
tenant_id + storage_id + product_id
```

`location_id` on a movement or balance row is a denormalized snapshot of `StorageArea.location_id` at post time. The backend must validate that they match. A Vodice storage must never be combined with a Šibenik `location_id`.

A tenant with Vodice, Šibenik, and Zadar has three independent on-hand figures for the same vodka. Do not clone `Product` per venue.

Even a one-venue cafe has a `BusinessLocation`. There is no tenant-global pile of stock.

v1 API returns `on_hand` (the sum of posted quantities). It must not be named `available`. Later:

```text
available = on_hand − reservations/allocations
```

Reservations are out of scope for v1.

### 2. Minimal place model

Stock cannot exist without a place. This ADR defines a minimal `BusinessLocation` and `StorageArea`. It does not define floor plans, tables, rooms, staff roster, address, municipality, fiscal devices, or tax-rate tables.

```text
BusinessLocation
----------------
id
tenant_id
name
timezone
is_active

StorageArea
-----------
id
tenant_id
location_id
name                # Bar, Kitchen, Cellar, Walk-in
is_default
is_active
```

- A location has at least one storage. Implementation may seed `Main`.
- Exactly one **active** default storage per location.
- Multi-storage is in the model from day one so later transfers are not a retrofit.
- Location is not resolved from host or URL (ADR 0001). The authenticated tenant plus an explicit location on the document select the place.
- Deactivated location or storage: no new operational movements; historical movements and balances remain.

`BusinessLocation.timezone` is the place clock for business dates. Tenant timezone alone is not enough for a multi-city group.

### 3. Append-only stock ledger

Source of truth is `StockMovement`. `StockBalance.on_hand` is a projection maintained in the same posting transaction.

Every posted movement is immutable. Corrections are new reversing or correcting movements, not an `UPDATE` of quantity.

Conceptual movement:

```text
id
tenant_id
location_id               # snapshot; must match storage.location_id
storage_id
product_id
quantity_in_base_unit     # signed decimal; + in, − out
base_unit_id              # snapshot; must equal product.base_unit_id
ledger_sequence           # tenant-monotonic order; not timestamp-only
occurred_at               # timezone-aware UTC instant
posted_at                 # server-controlled UTC instant
movement_type
source_document_type
source_document_id
posting_generation        # 0 = original post; later = reverse/correct
reversal_of_movement_id   # set on reversing movements
reason_code               # required for WASTE, INTERNAL_USE, ADJUSTMENT
snapshots                 # long_name, sku, base unit code, …
```

Idempotency is recorded on the **posting**, not copied onto every movement line.

Ledger rules:

- Quantity is always in the product `base_unit`. Packaging conversion happens on the source document, never as a second unit inside the ledger.
- Quantity is a decimal, never a float. The backend **rejects** a quantity with more decimals than the UoM allows. There is no silent rounding. `piece` with precision 0 rejects `0.5`. Balance sums use the same canonical precision as the ledger.
- `quantity_in_base_unit != 0`.
- `is_stock_tracked=false` produces no movements (corkage, service). Those products have no on-hand, not zero.
- An inactive product accepts no new operational movements except adjustment, stocktake, or reversal of an existing document.
- After first use, ADR 0002 locks `base_unit_id`. A movement stores the unit at post time and it must equal the product’s locked base unit. Warehouse cannot change the unit.

### 4. Movement types

v1 uses a closed list. New types may be added later; clients must not invent free-text types.

| Type | Sign | Typical source document |
|------|------|-------------------------|
| `INITIAL_STOCK` | + | opening balance / go-live |
| `RECEIPT` | + | goods receipt (procurement ADR posts this) |
| `SALE` | − | POS sale-action expansion |
| `PRODUCTION_IN` | + | finished / semi-finished output |
| `PRODUCTION_OUT` | − | components consumed by production |
| `TRANSFER_OUT` / `TRANSFER_IN` | − / + | one transfer document, two or more lines |
| `WASTE` | − | physical loss: breakage, expiry, quality, spill |
| `INTERNAL_USE` | − | staff, representation, complimentary |
| `ADJUSTMENT` | +/− | manual correction with reason |
| `STOCKTAKE_VARIANCE` | +/− | inventura count vs theoretical at cutoff |
| `RETURN_TO_SUPPLIER` | − | procurement return |
| `CUSTOMER_RETURN` | + | rare in cafe; keep in the list |
| `REVERSAL` | opposite of original | linked void or correct of any posted type |

Warehouse does not own purchase-order or POS ticket schema. It owns the movement types those documents are allowed to post.

`WASTE` and `INTERNAL_USE` are different classifications, not extra document tables.

- `WASTE` — physical loss: breakage, expiry, quality, spill.
- `INTERNAL_USE` — staff, representation, complimentary.

Both require a `reason_code`, as does `ADJUSTMENT`. Seed waste reasons: `BREAKAGE`, `EXPIRED`, `QUALITY`, `SPILL`, `THEFT_SUSPECTED`, `OTHER`. Seed internal-use reasons: `STAFF`, `COMPLIMENTARY`, `REPRESENTATION`. Do not freeze the legal list.

Complimentary and staff consumption are inventory events, not free POS items that skip the ledger. Fiscal treatment belongs to a later POS / fiscal ADR.

Do not treat `SALE_VOID` as an unlinked `+` movement. Voids use the general reversal mechanism.

There is no public “set quantity to X” API.

### 5. Documents post movements; clients do not

```text
source document is posted
→ backend expands lines (packaging → base unit, sale action → components)
→ backend writes movements + idempotency record + balance updates
  in one transaction
```

POS, Android, and web never send “deduct 8 g”. They send a document identity. Procurement sends a receipt, not a raw `+3000`.

Receipt and sale posting are owned by later ADRs. They must call the same movement writer.

Warehouse-owned documents in spirit: initial stock, transfer, waste, internal use, adjustment, stocktake. First implementation may be operator-only.

### 6. Hard lock: idempotency includes payload check

Same key alone is not enough.

- Same `idempotency_key` + same document content → return the original result.
- Same key + different content → conflict (`409`); do not post.
- The idempotency record and the movements are created in the same transaction.
- A failed transaction must not permanently consume the key.
- Concurrent posts of the same key must not produce two ledgers.

Otherwise a client can reuse a key for two different sales.

### 7. Hard lock: source document posting is unique

The same document must not post twice under two different idempotency keys.

```text
UNIQUE(tenant_id, source_document_type, source_document_id, posting_generation)
```

`posting_generation` allows a later reverse or correct posting. The original posting remains exactly one. Generation `0` is the first post. A reversal uses a new generation, not a second generation `0`.

### 8. Hard lock: reversal is linked, general, and bounded

```text
reversal_of_movement_id
```

- Reversal belongs to the same tenant.
- Same product, storage, and base unit as the original.
- Quantity has the opposite sign.
- The original movement is never updated.
- A movement must not be fully reversed more than once.
- Partial reversal is allowed only when explicit, and must not exceed the original quantity (sum of reversals ≤ |original|).
- One general reversal mechanism covers receipt, transfer, waste, internal use, adjustment, sale, and stocktake.

`location_id` on the reversal is the snapshot of the same storage.

### 9. Hard lock: document lifecycle and atomic post

Canonical states, without designing every document table:

```text
DRAFT → POSTED
POSTED → REVERSED
```

- Draft does not affect `on_hand`.
- Only `POSTED` produces movements.
- A posted document is not edited.
- “Deleting” a posted document means posting a reversal.
- The document and all of its movements post atomically.
- There is no partially posted state.

This is the warehouse posting contract later document ADRs must use.

### 10. Hard lock: balance updates in the same transaction

The ledger is authority. Operational POS still needs an immediately consistent `on_hand`.

- Movement insert and the matching `StockBalance` change succeed or fail together.
- After a successful post, the API must not return the previous `on_hand`.
- Async rebuild is for check and repair of the projection, not for ordinary posting.
- Rebuild must not drop movements that arrived during the rebuild.
- A detected ledger-versus-balance gap is audited and visible, never silently hidden.

Two posts against the same `(storage, product)` must not lose updates. Row lock versus serializable isolation is an implementation choice; the outcome is locked.

### 11. Hard lock: stocktake has a cutoff

If sales continue during inventura, `counted − current theoretical` is the wrong variance.

- A count session has `cutoff_at` and/or a ledger high-water marker.
- Theoretical quantity is computed **at that cutoff**.
- Movements after the cutoff are not in that comparison.
- Posting variance must detect a stale session or an already-posted session.
- The same count session may produce variance movements only once.

```text
count session (cutoff_at / ledger high-water)
→ theoretical qty at cutoff
→ variance = counted − theoretical_at_cutoff
→ one POSTED generation of STOCKTAKE_VARIANCE movements
```

A posted stocktake is immutable. A recount is a new document. Count is in base unit; the UI may convert packaging. Blind count versus show-theoretical is UX, not domain. Full cycle-count and ABC classification are out of scope.

### 12. Hard lock: used storage cannot change location

After the first movement on a storage:

- `StorageArea.location_id` is immutable.
- Editing location must not turn Vodice history into Šibenik stock.
- Rename is allowed.
- Moving goods is a transfer, not a field edit.
- Deactivate or reactivate does not lift the lock (same spirit as Product `base_unit_id` in ADR 0002).

Transfer source and destination must not be the same storage.

### 13. Snapshots on movements

A movement stores at least: product id, `long_name`, `sku` if any, `base_unit` code, and the posted quantity. Later rename or deactivate of the product must not change the movement text or unit meaning.

Do not use live `Product.long_name` as the only display source for a 2026 movement.

### 14. Theoretical versus actual stock

Recipe and sale-action deductions are **theoretical** consumption (normativ). Overpour, theft, and evaporation appear as stocktake variance. Known complimentary or staff use should be posted as `INTERNAL_USE`, not left for inventura to discover if the operator already knows.

Warehouse must not “fix” a sale by silently changing the recipe quantity. `STOCKTAKE_VARIANCE`, `WASTE`, and `INTERNAL_USE` are first-class, not dirty adjustments.

v1 can ship without a full inventura UX. The movement type and the theoretical/actual distinction must exist so POS sales do not pretend to be exact physical truth.

### 15. Negative stock policy

Default: **allow** negative `on_hand` for `is_stock_tracked` goods. A cafe may sell the last Coke before the receipt is posted.

- Reports must show negative as negative, not clamp to zero.
- `is_stock_tracked=false` has no on-hand, not zero.
- A later per-location or per-product warn/block policy is a hook, not the ledger shape.

Hard-blocking all negatives in v1 is rejected; it stops service.

### 16. Transfers

- Same tenant only. Never transfer across tenants.
- Same product identity on both sides. Changing product identity is production, not transfer.
- Both legs use the same base-unit amount in v1. In-transit loss or spill is a later hook.
- Cross-location and cross-storage are both transfers.
- One document posts all legs atomically. Partial failure is not allowed.
- Source storage ≠ destination storage.
- In-transit / “on the van” storage is a later hook if both legs post atomically in v1.

Example: Vodice cellar `TRANSFER_OUT −700 ml` vodka and Šibenik bar `TRANSFER_IN +700 ml` on one document.

### 17. Production is not a transfer

Warehouse must accept `PRODUCTION_OUT` and `PRODUCTION_IN` so semi-finished goods from ADR 0002 work (tomatoes → house sauce).

Yield, batch, and cook-loss belong to a Recipes / Production ADR. Production movements are against existing Products.

### 18. Opening balance

`INITIAL_STOCK` is a posted document, not a silent balance insert. Same snapshots, same immutability, same grain, same lifecycle.

### 19. Time, timezone, and ledger order

- `occurred_at` is a timezone-aware UTC instant.
- `posted_at` is a server-controlled UTC instant. The client does not set it.
- Location timezone derives the business date for reports.
- Backdated `occurred_at` is allowed only under later authority or period-close rules (hook).
- Store UTC so DST does not produce an ambiguous local timestamp. Derive the local business date from the location timezone.
- Equal `occurred_at` is not a sufficient order. Reports and rebuild use a tenant-monotonic `ledger_sequence` (or equivalent). Do not sort the ledger by timestamp alone. This will matter for later valuation.

### 20. Capabilities and lifecycle (from ADR 0002)

- `is_stock_tracked=false`: no balance row required; no movements.
- `is_purchasable=false`: may still have on-hand.
- `is_sellable=false`: may still be deducted as a component.
- `is_producible=true` does not imply stock is tracked.
- Deactivating a product, location, or storage does not rewrite history.

### 21. Authorization

`product_id`, `location_id`, `storage_id`, or `movement_id` alone never authorizes. Every query is tenant-scoped. Location, storage, and product must belong to the same tenant as the API key.

Celery rebuilds use `TenantTask` and an explicit `tenant_id` (ADR 0001).

### 22. Quantity-only v1; valuation and lots are hooks

The v1 ledger stores quantity, not value. FIFO, LIFO, weighted average, and inventory value are a later valuation layer. Do not put `unit_cost` on Product as warehouse truth (ADR 0002 already rejected `Product.price`).

Lots, expiry, and serials are out of scope. `lot_id` is not required. The movement shape must not make a later lot dimension impossible.

Reorder min/max and par levels may later sit on `(location, storage, product)`. They are not required to post stock.

Virtual locations (supplier, customer, production, inventory-loss), quarantine storage, in-transit storage, period close, and reserved quantity are hooks. v1 does not require a “Supplier” storage to receive goods.

### 23. Worked examples

**Coca-Cola, Vodice bar.** Receipt of 1 crate: the document converts 24 pieces → `RECEIPT +24 piece`. POS sale → `SALE −1 piece`. `on_hand` is 23. Rename of the product does not change the two movement snapshots.

**Vodka.** Receipt of 1 bottle → `+700 ml`. Tap 0.03 L → `SALE −30 ml`. Bundle bottle + 4 Red Bulls → several `SALE` lines, still vodka `−700 ml`. Warehouse does not store “bundle”.

**Coffee.** Receipt of a 3 kg container → `+3000 g`. Cappuccino sale → `SALE −8 g` coffee and `SALE −120 ml` milk. Stocktake at cutoff finds 50 g less than theoretical-at-cutoff → one `STOCKTAKE_VARIANCE −50 g`. Sales after cutoff are not in that variance.

**House sauce.** `PRODUCTION_OUT` tomatoes, oil, garlic; `PRODUCTION_IN` house sauce. A later recipe sale deducts `−50 g` house sauce.

**Transfer.** One document: Vodice cellar `TRANSFER_OUT −700 ml`; Šibenik bar `TRANSFER_IN +700 ml`.

**Retry.** The same sale posted twice with the same idempotency key and the same payload returns the original posting. The same key with a different payload returns `409`.

**Void.** A posted sale is reversed with linked `REVERSAL` movements. The original `SALE` lines stay unchanged. `on_hand` returns to the pre-sale figure in the same transaction.

## Rejected alternatives

- **Quantity on `Product` or a tenant-wide stock row** — rejected; stock is located.
- **Editing a posted movement or posted document** — rejected; history is append-only.
- **Same `idempotency_key` as success regardless of payload** — rejected; it can merge two sales.
- **Posting the same source document twice via a new idempotency key** — rejected.
- **Unlinked `SALE_VOID` or `+` correction** — rejected; reversal is general and linked.
- **Partially posted documents** — rejected.
- **Returning stale `on_hand` after a successful post** — rejected.
- **Async rebuild as the ordinary write path** — rejected; rebuild is repair.
- **Stocktake variance against live theoretical while the bar is selling** — rejected; use cutoff.
- **Moving a used storage to another location** — rejected; that rewrites place history.
- **Mixing a storage with a foreign `location_id`** — rejected.
- **Calling v1 `on_hand` “available”** — rejected; reservations come later.
- **Silent rounding of quantities** — rejected; excess precision is an error.
- **Sorting the ledger by timestamp alone** — rejected; use `ledger_sequence`.
- **Float quantities or ledger quantities in purchase packaging** — rejected (ADR 0002).
- **POS sending component deductions** — rejected (ADR 0002).
- **Cross-tenant transfer or same-storage transfer** — rejected.
- **Transfer that changes product identity** — rejected; that is production.
- **Cloning Product to represent “vodka in Šibenik”** — rejected (ADR 0002).
- **Treating theoretical recipe stock as physically exact** — rejected.
- **Treating complimentary or staff use as waste by default** — rejected.
- **FIFO / weighted average as a go-live requirement** — rejected.
- **Hard-blocking all negative stock in v1** — rejected; it stops hospitality service.
- **Resolving location from host** — rejected (ADR 0001).
- **Virtual location graph required to receive or sell** — rejected for v1.

## Consequences

### Positive

- One ledger serves purchase, production, POS, waste, transfer, and inventura without a second Product master.
- Retries, voids, and inventura during service have explicit, auditable rules.
- `on_hand` is immediately consistent for POS while the ledger remains rebuildable.
- Location and storage history cannot be rewritten by a field edit.
- Later valuation can attach cost to an ordered ledger without changing quantity semantics.

### Negative

- Posting is stricter than a simple quantity update: documents, idempotency payload, and atomic balance writes are mandatory.
- Allowing negative `on_hand` requires operational discipline and later policy hooks.
- Several follow-up ADRs are still required before procurement and POS can finish.

### Neutral

- `location_id` on movements is redundant with storage and exists for validation, history, and query performance.
- First inventory implementation can ship location, storage, movements, and `on_hand` without lots, cost, or a full inventura UI.

## Invariants

1. Stock is never a field on `Product`. Authority grain is tenant + storage + product. `location_id` is a validated snapshot.
2. The ledger is append-only. Quantity is in `base_unit`, decimal, non-zero. Excess precision is rejected.
3. Packaging conversion happens on the source document, not in the ledger.
4. `is_stock_tracked=false` produces no movements.
5. Same idempotency key and same payload return the original posting. Same key and different payload is `409`. The record and movements share one transaction. A failed transaction does not consume the key.
6. A source document posts once per `posting_generation`. Generation `0` is unique.
7. Reversal is linked, same tenant/product/storage/unit, opposite sign. The original is unchanged. Full reverse happens at most once. Partial reverse is explicit and bounded.
8. Documents follow `DRAFT → POSTED → REVERSED`. Only `POSTED` moves stock. Posting is atomic. There is no partial post.
9. `StockBalance` / `on_hand` updates in the same transaction as movements. Rebuild is repair, not the write path. Gaps are audited.
10. Stocktake variance uses cutoff or high-water. One post per session. Stale or already-posted sessions are rejected.
11. After first use, `StorageArea.location_id` is immutable. Deactivate does not lift the lock. Rename is allowed. Goods move by transfer.
12. Exactly one active default storage per location. Transfer source ≠ destination. A transfer document is atomic.
13. Location, storage, and product must share the tenant. IDs alone do not authorize.
14. Movements snapshot product identity fields. A later Product rename does not rewrite history.
15. There is no “set quantity to X” without a document that produces movements.
16. ADR 0002 `base_unit` lock applies. Warehouse cannot change it.
17. v1 API returns `on_hand`, not `available`.
18. `WASTE` is physical loss. `INTERNAL_USE` is staff / complimentary / representation.
19. Negative `on_hand` is allowed by default and is not clamped to zero.
20. Ledger order uses `ledger_sequence`, not timestamp alone.

## Follow-up ADRs

```text
Procurement and Goods Receiving
Recipes and Production
POS Sales and Sale Actions
Modifiers
Bundles and Promotions
Tax Model
Invoices and Fiscalization
Barcode Generation and Label Printing
```

Valuation, lots/expiry, and period close may follow warehouse once quantity posting is stable.

The next ADR should define **Procurement and Goods Receiving**, posting `RECEIPT` (and later `RETURN_TO_SUPPLIER`) through this movement writer.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)

## Out of scope

This ADR does not define:

- purchase orders or goods-receipt document schema
- supplier model or supplier item codes
- recipe, production-batch, or yield schema
- POS tickets, pricing, discounts, or fiscalization
- modifier or bundle schema
- invoice schema
- concrete VAT or municipal tax tables
- lots, expiry, or serial tracking
- inventory valuation or FIFO / LIFO / weighted average
- catch-weight scales
- barcode generation or label printing
- floor plans, tables, or rooms
- location address, municipality, or fiscal device binding
- staff permissions beyond tenant API keys
- reserved / allocated quantity
- in-transit, quarantine, or virtual location graphs
- period-close procedure
- full inventura UX
- the controlled base-unit migration procedure (ADR 0002)
