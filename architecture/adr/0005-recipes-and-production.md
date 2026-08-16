# ADR 0005: Recipes and Production

## Status

Proposed

## Date

2026-08-15

## Context

ADR 0002 locked one canonical `Product`, Recipe ≠ Bundle, and the rule that POS expands a frozen sale-action version. ADR 0003 locked the stock ledger, `PRODUCTION_OUT` / `PRODUCTION_IN`, and the posting contract. ADR 0004 locked physical procurement documents that call that writer.

This ADR owns the normative and the production document. Without it, a recipe save would move stock, a cappuccino would be booked as produced-then-sold, a syrup component would explode into sugar and water (double deduction), and an open POS ticket would change its bill of materials mid-shift.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the recipes and production domain **before** `apps.recipes`. Physical schema details belong in a later implementation. The semantics below must not change.

The governing rule:

```text
Recipe defines the expected transformation.
Production Order records an actual successful production event.
A successful posted order has PRODUCTION_OUT inputs
and a positive primary PRODUCTION_IN.
A fully failed batch is WASTE, not a zero-output production.
Semi-finished products are deducted as themselves;
recipes are never exploded recursively.
POS freezes recipe_version_id when the sale line is added.
```

POS ticket schema stays in the POS ADR. This ADR locks what a frozen recipe version means.

## Decision

### 1. Two recipe purposes, two posting events

**SALE.** A guest item (cappuccino) deducts components at sale time as `SALE`. There is no `PRODUCTION_IN` for the cup handed to the guest.

```text
1 Cappuccino
→ SALE −8 g coffee
→ SALE −120 ml milk
```

**PRODUCTION.** A stored output (house sauce). A posted Production Order writes `PRODUCTION_OUT` for inputs and `PRODUCTION_IN` for the output.

```text
Recipe: 1 kg tomato + 100 ml oil → expected 900 g sauce
Actual: 2 kg tomato + 200 ml oil → 1.75 kg sauce
Ledger:
  PRODUCTION_OUT −2000 g tomato
  PRODUCTION_OUT −200 ml oil
  PRODUCTION_IN  +1750 g sauce
```

Shared component concepts. **Not** the same posting event.

A product may have a SALE recipe, a PRODUCTION recipe, both, or neither. They are separate `Recipe` records. A SALE-purpose recipe must not be used on a Production Order.

### 2. Recipe is a versioned normative

```text
Recipe
------
id
tenant_id
output_product_id
name
purpose                 # SALE | PRODUCTION

RecipeVersion
-------------
recipe_id
version
status                  # DRAFT | ACTIVE | RETIRED
expected_output_quantity
output_base_unit_id
timestamps

RecipeComponent
---------------
recipe_version_id
component_product_id
quantity_in_base_unit
sequence
is_optional
waste_or_yield_note
snapshots               # name, sku, base-unit code
```

- Output and every component belong to the same tenant as the recipe.
- Quantities are decimal, `> 0`, in each product’s `base_unit`. No float. Excess precision is rejected.
- A component produces a movement only if `is_stock_tracked=true`.
- A recipe does not change `Product.base_unit`.
- Recipe is not a Product and has no `on_hand`. House Sauce is a Product. The recipe describes how it is made. A product may exist without a recipe. `is_producible` does not imply a recipe exists. Do not clone Product because the recipe changed.

**One ACTIVE version.** At most one `ACTIVE` version per Recipe. Activating a new version and retiring the previous happen in one transaction. `(recipe_id, version)` is unique. Version number is not an authorization key. `ACTIVE` is immutable. `RETIRED` cannot be edited; clone it into a new `DRAFT`. Many recipes per output product are allowed.

A published version snapshots each component: `product_id`, name, SKU if any, base-unit code, normative quantity, sequence, optional flag. Live Product remains the posting and authorization reference. Historical display must not depend only on today’s name. Deactivating a product does not rewrite old versions.

**`is_optional`.** The component may be omitted without a substitution. Omission is recorded on the Production Order. An unused optional component produces no movement. A required component cannot disappear without an explicit override reason. Optional on a PRODUCTION recipe is not a POS Modifier.

### 3. Expected output quantity is required

The recipe always states how much output the component list is for (`1 piece` pizza, `5000 g` sauce, `10000 ml` soup).

```text
factor = planned_output / recipe_expected_output
scaled_component = recipe_component_qty × factor
```

Scaling is decimal and deterministic. Reject a planned or actual quantity that violates UoM precision (`2.4 piece` if precision is 0). Scaling is the **plan**, not the ledger.

### 4. Never explode a semi-finished product

A SALE or PRODUCTION step that consumes a prepared Product deducts **that** Product. It does not open that Product’s recipe.

```text
Cappuccino uses 30 ml prepared syrup.
Syrup has its own PRODUCTION recipe.
Sale: SALE −30 ml Syrup
Do not expand syrup into sugar and water.
Those were consumed on the Production Order that created syrup on_hand.
```

```text
Sauce uses prepared stock.
Production: PRODUCTION_OUT of stock.
Do not expand the stock’s old recipe.
```

Without this lock, components are deducted twice.

### 5. Cycle check is PRODUCTION-only

- Direct and indirect cycles are rejected among **ACTIVE PRODUCTION** recipes (output ↔ component edges) at activate.
- SALE recipes are not expanded recursively. A SALE recipe that consumes a semi-finished Product only deducts that Product.
- A SALE-graph cycle may be rejected as catalog integrity. It must not imply recursive POS expansion.
- Retired versions stay readable. Reuse of a smaller product in a larger recipe is allowed if there is no PRODUCTION back-edge.

### 6. Production Order is the physical document

```text
ProductionOrder
---------------
id
tenant_id
location_id
source_storage_id
destination_storage_id
recipe_version_id
output_product_id
status
planned_output_quantity
actual_output_quantity
occurred_at
posted_at
idempotency_key
snapshots
```

```text
DRAFT → POSTED → REVERSED
```

- Draft does not move stock. Posted is not edited or deleted. Error = linked reversal + new order.
- A SALE-purpose recipe must not appear on a Production Order.
- Reuse the ADR 0003 posting contract: idempotency with payload (`409` on mismatch), unique `(tenant, document_type, document_id, posting_generation)`, server `posted_at`, no partial post.

**Successful production.** A posted order must have a **positive** primary `PRODUCTION_IN`. All `PRODUCTION_OUT`, primary `PRODUCTION_IN`, by-product `PRODUCTION_IN`, and inline `WASTE` post atomically through the ADR 0003 writer. There is no `PRODUCTION_IN 0` (ADR 0003 forbids zero movements). There is no production `OUT` without production `IN`.

**Fully failed batch.** Not a normal production. Consumed or destroyed inputs post as `WASTE` with reason `FAILED_BATCH` (or equivalent). The waste may link to a failed draft or cancelled Production Order for audit.

**Location and storage.** One `location_id`. Both source and destination storage **must belong to that location**. Cross-location production in one order is forbidden; move ingredients with an ADR 0003 transfer first. v1: all inputs leave one source storage; all outputs and by-products enter one destination storage. Same or different storages **within** the location are allowed. Different storages do not make this a transfer.

**Reversal.** Linked general reversal (ADR 0003) reverses `PRODUCTION_OUT`, primary `PRODUCTION_IN`, by-product `PRODUCTION_IN`, and inline `WASTE` in one transaction.

### 7. Planned vs actual; ledger uses actuals

```text
planned_output_quantity
planned_input_qty_base      # recipe × factor; snapshot
actual_input_qty_base       # required at post
actual_output_quantity      # required at post for successful production
```

Posted movements use **actual** quantities only. Do not silently rewrite actuals to match the recipe. `is_stock_tracked=false` lines may appear and produce no movement.

Every posted actual line has provenance (semantics now; table later):

```text
PLANNED_COMPONENT
SUBSTITUTION
EXTRA_INPUT
BY_PRODUCT
WASTE
```

- A planned input points at the original recipe component when it exists.
- A substitute points at the planned component it replaced.
- An unplanned extra input has a reason.
- A by-product is explicitly marked.
- Snapshots store product, unit, planned quantity, and actual quantity.

### 8. Yield and waste

```text
expected_output = planned_output
actual_output
yield_ratio = actual_output / expected_output   # report metric, not a movement
```

- On a **successful** order, the yield gap is not auto-posted as `WASTE`. The ledger already records OUT actuals and IN actuals.
- Explicit spill or discard is an inline `WASTE` line: a real `WASTE` movement through the ADR 0003 writer, with a reason code, not a note that skips the ledger. It posts or fails with the order. Reversal reverses those waste movements.
- A fully failed batch is `WASTE` / `FAILED_BATCH`, not production with zero IN.
- Usable leftover is a by-product (`PRODUCTION_IN`), not waste.

### 9. Substitutions have no automatic equivalence

Substitutions live on the Production Order, not on the published recipe version.

- The operator **types** the actual quantity of the substitute Product.
- `1000 g` fresh tomato is not automatically `800 g` passata because both are MASS.
- No automatic cross-dimension conversion. Different dimensions are allowed only as an explicit override, still without computed equivalence.
- Reason is required.
- The substitute posts only in its own `base_unit`.
- An allowed-substitute catalog is a later hook.
- A guest-facing swap (oat milk) is a Modifier (POS ADR), not a production substitution.

### 10. One primary output; optional by-products

- v1: exactly one primary output. Successful post: primary `PRODUCTION_IN` `> 0`.
- Optional extra `PRODUCTION_IN` lines for by-products. Each is a Product with a **positive actual** quantity.
- A by-product does not automatically reduce primary yield and is not an automatic financial credit. Valuation split is a later ADR.
- Discarded scrap is `WASTE`, not a fake Product.
- Multi-primary finished goods of equal rank are a later hook.

### 11. POS freezes the recipe when the line is added

This ADR owns versioning. The POS ADR owns the sale document.

- A `RECIPE` sale action points at a SALE-purpose recipe.
- `recipe_version_id` is frozen when the sale action is **added** to a draft ticket (the then-`ACTIVE` version).
- Changing quantity on that same line keeps the same version.
- A **new** line added after a newer recipe was activated gets the new `ACTIVE` version.
- Posted sale expands `SALE` movements from the version stored on the line. Never explode semi-finished components.
- If a draft line has no version, post **fail-closed**. Do not silently take current `ACTIVE`.
- Reopening or editing a fiscalized sale is forbidden.
- A later activate must not change posted or fiscalized sales, or other open lines that already froze an older version.
- New lines require `ACTIVE`. `DRAFT` cannot be sold. `RETIRED` is readable for old lines and sales.
- Sale expansion posts `SALE`, never `PRODUCTION_*`.

### 12. Allergens remain a hook

ADR 0002 already requires `contains` versus `may_contain`. Recipe-derived allergens and manual override are not designed here.

## Rejected alternatives

- Posting stock when a recipe is saved or activated.
- Treating make-to-order cappuccino as `PRODUCTION_IN` plus immediate `SALE` of the cup.
- Recursively exploding a semi-finished Product on sale or production.
- Posting a failed batch as production with `PRODUCTION_IN 0` or `OUT` without `IN`.
- Freezing the sale recipe only at post (silent current `ACTIVE`).
- Reopening a fiscalized sale to pick a new recipe version.
- Cross-location production in one order.
- Automatic 1:1 substitute equivalence.
- Auto-waste for a successful order’s yield variance.
- Informative waste that does not hit the ledger.
- Merging Recipe and Bundle.
- Editing an `ACTIVE` or `RETIRED` version in place.
- Cloning Product because the recipe changed.
- Netted “only the difference” production booking.
- POS computing component deductions.
- Expanding an old sale against the current recipe.
- Treating PRODUCTION `is_optional` as a POS modifier.

## Consequences

### Positive

- Normative, batch production, and POS consumption stay three different facts.
- Semi-finished goods keep a single `on_hand` and are not deducted twice.
- An open ticket keeps the recipe that applied when the line was added.
- Failed batches and successful yield variance are not confused.
- Production cannot hide a cross-location transfer.
- Every departure from the normative has provenance.

### Negative

- Operators must post a Production Order (or failed-batch waste) instead of “just making sauce”.
- v1 substitutions require typed quantities and a reason; there is no smart equivalence table.
- POS must persist `recipe_version_id` on the draft line, not only at fiscalization.

### Neutral

- First implementation can ship Recipe versions and Production Orders without costing, lots, or a substitute catalog.
- The POS ADR will consume the freeze rule; this ADR does not design the ticket.

## Invariants

1. A recipe never moves stock. A successful posted Production Order posts `PRODUCTION_OUT` plus a positive primary `PRODUCTION_IN` (and optional by-product IN / inline `WASTE`) atomically through the ADR 0003 writer.
2. A fully failed batch is reasoned `WASTE` (`FAILED_BATCH`). No `PRODUCTION_IN 0`. No production `OUT` without production `IN`.
3. Semi-finished products are never exploded. Sale and production deduct that Product only.
4. SALE and PRODUCTION recipes share components but not posting events.
5. At most one `ACTIVE` version per Recipe. Activate and retire are one transaction. `(recipe_id, version)` is unique. `ACTIVE` and `RETIRED` are immutable; clone `RETIRED` to a new `DRAFT`.
6. Cycle check is on ACTIVE PRODUCTION edges only. SALE is not recursively expanded.
7. POS freezes `recipe_version_id` when the sale line is added. Post uses the stored version; missing version fail-closed. Fiscalized sales are not reopened.
8. Source and destination storage both belong to the order’s one `location_id`. No cross-location production. v1: one source for all inputs, one destination for all outputs.
9. Actual lines have provenance: planned, substitute, extra, by-product, or waste. Snapshots include name, unit, planned quantity, and actual quantity.
10. Substitution has no automatic product equivalence. The operator types actual quantity in the substitute’s `base_unit`. Reason is required.
11. The ledger uses actuals. A successful order’s yield gap is not auto-`WASTE`. Inline waste is a real `WASTE` movement with a reason.
12. Reversal reverses `OUT`, primary `IN`, by-products, and waste in one transaction.
13. A by-product has a positive actual quantity. It does not auto-reduce primary yield or create a financial credit.
14. Recipe is not a Product and has no `on_hand`. Tenant isolation; ids alone do not authorize.

## Follow-up ADRs

```text
POS Sales and Sale Actions
Modifiers
Bundles and Promotions
Tax Model
Invoices and Fiscalization
Barcode Generation and Label Printing
```

The next domain ADR should define **POS Sales and Sale Actions**, posting `SALE` through the ADR 0003 writer and freezing `recipe_version_id` when a recipe line is added.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)

## Out of scope

This ADR does not define:

- POS ticket, layout, pricing, or fiscalization
- modifiers
- bundle schema
- lots, expiry, or serial tracking
- costing, theoretical food cost, or by-product valuation
- production planning or MRP
- an allowed-substitute catalog
- allergen derivation tables
- purchase or sales tax calculation
