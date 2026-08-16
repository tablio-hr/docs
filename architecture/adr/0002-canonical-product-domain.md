# ADR 0002: Canonical Product domain

## Status

Accepted (2026-08-15)

Amended 2026-08-15: Divisible COUNT products.

## Date

2026-08-15

## Context

Tablio is a multi-tenant platform for cafes, restaurants, and other hospitality venues.

ADR 0001 already locks the tenancy boundary: the host selects the surface; authentication selects the tenant. Every business record is tenant-scoped. A URL slug or host must never determine tenant data.

Before warehouse, procurement, POS, recipes, or invoices are designed, the platform needs one canonical Product model. Those modules must share the same `Product` as the source of truth.

The goal is not separate masters for “warehouse article”, “procurement article”, and “POS item”. There is one `Product`. Configuration, units, inputs, outputs, recipes, and sale actions determine how it behaves.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the Product domain **before** the first catalog implementation. Physical schema details belong in the later `apps.catalog` implementation, but the semantics below must not change.

## Decision

### 1. One canonical Product

`Product` is anything whose quantity, purchase, consumption, production, or sale the system must track.

The same model represents:

- a warehouse-only item that never appears on POS
- a raw material
- trade goods received and sold as-is
- a product sold by measured amount
- a product made from other products
- a semi-finished product
- a finished product
- a recipe component
- a bundle component
- a POS-visible product

There are no separate primary tables for raw material, article, POS item, or goods. All of them are `Product`.

Every `Product` belongs to exactly one tenant. `product_id` alone is never enough for authorization. Every read and write is tenant-scoped.

```text
tenant_id = 42
product_id = 1001
long_name = "Absolut Vodka 0.7 L"
```

Warehouse, procurement, and POS must not create their own copies of the Product master.

### 2. Product identity

Minimum identity fields:

```text
id
tenant_id
long_name
short_name
sku                 # optional
barcode             # optional
barcode_source
category_id
base_unit_id
is_active
created_at
updated_at
```

Exact column types are an implementation concern. The meaning of these fields is not.

Price is **not** a Product field. Price is channel-, location-, and time-scoped and belongs to a later POS / price-list ADR.

### 3. Names

`long_name` is the full product name and is required.

```text
Absolut Vodka 0.7 L
Coca-Cola Original 0.25 L
Lavazza Espresso Coffee Beans
Fresh Whole Milk 3.2%
```

`short_name` is stored, not computed on read. It is used where space is limited: POS buttons, mobile lists, compact receipts, some reports.

Rules:

- Implementation defines maximum lengths. `short_name` is shorter than `long_name`.
- Duplicate `short_name` within a tenant is allowed.
- If `short_name` is empty at **create**, the backend generates and stores it from `long_name`.
- A later change to `long_name` must not overwrite a stored `short_name` (manual or previously generated).
- Truncation must be Unicode-safe. Characters such as `č`, `ć`, and `ž` must not be split.
- The exact shortening algorithm is not part of this ADR.

```text
long_name  = "Absolut Vodka Original 0.7 L"
short_name = "Absolut Vodka"
```

### 4. SKU

`sku` is an optional internal code for receiving, imports, and staff. It is not a barcode, fiscal code, supplier code, or external integration id.

Rules:

- Trim whitespace. Empty string is stored as `NULL`.
- Unique per tenant when set.
- Compared case-insensitively (normalized store or a case-insensitive unique index).
- Does not change when `long_name` or `short_name` changes.
- Equal SKUs across tenants are not the same product.

### 5. Barcode

v1 allows one barcode on `Product` and optionally one barcode on each `ProductPackaging`. Generation and label printing come later; the model must not block them.

Rules:

- Barcode is a nullable string, never a number (leading zeros must survive).
- Trim whitespace. Empty string is stored as `NULL`.
- Uniqueness is checked on the normalized value.
- When set, the barcode is unique in **one tenant namespace** shared by product barcodes and packaging barcodes.
- v1 does not require GTIN/EAN. Internal barcodes are allowed.
- `barcode_source` is reserved for values such as `MANUFACTURER` and `INTERNAL`.

A product without a barcode may remain without one.

### 6. Categories are not POS groups

Products are organized in hierarchical, tenant-scoped categories. Depth is not fixed. `parent_id = NULL` is a root category.

```text
ProductCategory
---------------
id
tenant_id
parent_id
name
is_active
sort_order
```

The category tree must not contain cycles. Delete and reparent rules apply when products still reference a category. Tenants do not share one category tree. The platform may later offer templates; each tenant still owns its structure.

`ProductCategory` is business classification, not POS button layout. The same vodka may live under `Beverages > Alcoholic > Spirits > Vodka` and appear in a POS group named `PROMOCIJE`. POS grouping is a later POS ADR.

### 7. Units of measure vs packaging

Two different concepts:

```text
UnitOfMeasure     = dimensional unit (piece, g, kg, ml, L)
ProductPackaging  = product-specific conversion (crate = 24 pieces, bag = 1000 g)
```

The platform seeds a shared UoM catalog with dimensions `COUNT`, `MASS`, and `VOLUME`, a reference unit per dimension (`piece`, `g`, `ml`), and exact conversions (`kg = 1000 g`, `L = 1000 ml`).

- Conversion is allowed only inside the same dimension. `g → ml` requires a later density or yield ADR.
- Quantity precision follows the UoM (piece uses 0 decimal places; mass and volume may be fractional).
- Platform UoMs are not tenant-deletable.
- A tenant must not invent a dimensionless shared unit such as “box” or “crate”. Those names belong on `ProductPackaging`.

### 8. Base unit

Every product has one canonical `base_unit`. Warehouse quantity is always in that unit.

Purchase packaging converts into the base unit. Packaging never replaces it.

```text
Coffee Beans    base_unit = g      1 × 3 kg container → +3000 g
Coca-Cola 0.25 L base_unit = piece  2 crates × 24      → +48 pieces
Vodka           base_unit = ml     1 bottle            → +700 ml
```

### 9. Hard lock: `base_unit_id` after first use

Until first use, `base_unit_id` may still be corrected.

**First use** means the product has any stock movement, goods receipt, recipe or component link, sale, or other business document.

After first use:

- The backend rejects an ordinary update that changes `base_unit_id`.
- The lock is enforced on the backend, not only in a client.
- Deactivating or reactivating the product does not lift the lock.
- Changing `L` → `ml` must not rewrite the meaning of historical quantities. Stored `10 L` must never become `10 ml`.
- Cloning a product must not be used to rewrite history. A clone is a new `Product` with a new identity. Historical documents, movements, and quantities stay on the original product.
- A new base unit for the same business item requires either a **new product** or a **separate controlled migration** that converts stored quantities, writes an audit trail, and snapshots affected documents. It is not a field edit.

### 10. Hard lock: exact packaging conversion

`ProductPackaging` has at least:

```text
product_id
name
barcode                 # optional
quantity_in_base_unit
is_active
```

Rules:

- `quantity_in_base_unit` must be `> 0`.
- It is stored as a decimal, never a float.
- Packaging is tenant-scoped through its product.
- The name does not define the conversion. “Gajba” for Coca-Cola may be `24` pieces; “gajba” for beer may be `20` pieces.
- A packaging barcode must not collide with a product barcode or another packaging barcode in the same tenant.

### 11. Input / Output

The Product domain is conceptually:

```text
INPUT → PRODUCT → OUTPUT
```

`Input` describes how quantity enters the business (purchase, production, return, transfer, initial stock, adjustment). `Output` describes how it leaves or is consumed (sale, recipe use, waste, transfer, adjustment).

The complete warehouse movement model is a later ADR. This ADR only requires that purchase input can convert a packaging quantity into the product base unit, and that one product may have several simultaneous outputs.

A product must not have a single `sale_unit` field that tries to describe every output.

### 12. Sale actions

One physical product may have several sale actions. POS may show one primary control and expose the others (for example by long press).

```text
Vodka
├── 0.03 L                         → -30 ml
├── Whole bottle                   → -700 ml
├── Bottle + 4 juices              → -700 ml vodka, -4 juice pieces
├── Bottle + 4 Red Bulls           → -700 ml vodka, -4 Red Bull pieces
└── Bottle + 2 Red Bulls + 2 juices
```

There is still one warehouse `Product` for vodka.

Conceptual types: `DIRECT`, `MEASURED`, `RECIPE`, `BUNDLE`. Names and schema are confirmed in a later POS / Recipe ADR.

**POS must not contain stock-deduction business logic.** POS loads available products and allowed actions, then posts the chosen `sale_action_id`. The backend validates tenant, location, product, and action, then decides the stock effects.

Sale-action stability (guardrail; versioning schema is a later ADR):

- `sale_action_id` must belong to the same tenant as the product and the POS.
- A deactivated action cannot start a new sale.
- An already posted or fiscalized sale remains readable.
- The backend expands the action using the version that applied at sale time, not a later recipe or bundle edit.

### 13. Recipe and bundle stay distinct

`Recipe` describes how a product is made or what it consists of (espresso consumes 8 g coffee beans; cappuccino consumes 8 g coffee beans and 120 ml milk).

`Bundle` is a commercial combination of existing products (vodka bottle + 4 Red Bulls). It is not a kitchen normative.

Both may deduct several products. They are not the same concept and must not be merged because both have a component list.

A semi-finished product (house sauce) is still a `Product`. It can be produced from other products and later used as a component. A product is not permanently labelled only as raw material or only as finished good.

`Modifier` (extra shot, oat milk, no ice) is a third concept. It is not a bundle and not a new Product. Schema belongs to a later POS / Recipe ADR.

Shopify-style `ProductVariant` (size / color) is rejected for v1. Hospitality pours and combos are sale actions.

### 14. Capabilities, not a type enum

Do not classify a product as `raw_material | finished_good | pos_item`. Use capabilities (on Product or a 1:1 config):

- `is_stock_tracked`
- `is_purchasable`
- `is_sellable`
- `is_producible`

The same vodka may be stocked, purchased, poured, sold as a bottle, used in a recipe, and used in a bundle.

Invariants:

- `is_producible=true` does not imply `is_stock_tracked=true`.
- `is_sellable=false` may still be a recipe or bundle component.
- `is_purchasable=false` does not forbid existing stock.
- Deactivating a product must not rewrite historical capability values on documents.

### 15. Tax, allergens, age, and deposit are hooks

These fields exist so later tax, menu, and fiscal ADRs do not reinvent Product. They are not the final models.

**VAT.** Product has a VAT classification, not a raw decimal rate. Rate changes, effective dates, historical invoices, and jurisdictions belong to a Tax ADR.

**Consumption tax.** Product has a classification hook, not a rate. Preferred values: `NONE`, `ALCOHOLIC_BEVERAGE`, `NON_ALCOHOLIC_BEVERAGE`, with room for later legal groups. A boolean `consumption_tax_applicable` is only a temporary stand-in for that classification, not the final tax model. The rate belongs to the business location and period. Category is not proof of how a product is taxed.

**Location.** A tenant may have several business locations (Vodice, Šibenik, Zadar). Product stays a tenant product. Local tax, stock, POS visibility, and price are per location. Do not clone Product per venue.

**Allergens.** The hook must be able to distinguish `contains` from `may_contain` (EU 1169/2011). Recipe-derived allergens and a manual override belong to a later ADR. Do not lock a single undifferentiated allergen set.

**Age.** `age_restricted` marks products that POS must treat as age-gated (alcohol).

**Deposit.** Amount is never on Product. Deposit can depend on packaging, location, and period. The same liquid may exist in packagings with and without deposit. The lasting link is more likely `ProductPackaging` or a dedicated deposit rule. A Product-level `deposit_applicable` flag, if present, is only a hook.

**External / fiscal codes** come later. Do not overload `sku` or `barcode` for Fiskalizacija.

### 16. Lifecycle and historical snapshots

Products that have been used in business documents are not hard-deleted. `is_active` hides them from new operations where they are not allowed. They remain on historical documents, invoices, and stock movements.

Names, tax classification, price, and similar attributes can change. Future documents (purchase receipt, warehouse document, POS sale, invoice) must store the snapshot values they need. A later rename of Coca-Cola must not rewrite a 2026 invoice. Snapshot detail belongs to those document ADRs.

### 17. Tenant isolation

At least these records are tenant-scoped: `Product`, `ProductCategory`, `ProductPackaging`, `ProductSaleAction`, `Recipe`, `RecipeComponent`, `Bundle`, `BundleComponent`.

If an entity can exist as a global platform catalog (UoM), its link to a tenant Product is explicit.

Products of different tenants must never be joined implicitly by equal name, barcode, SKU, category, or supplier code.

### 18. API principle

POS, warehouse, and procurement clients do not interpret Product business rules. The backend is the authority.

```text
POS
→ GET available products/actions
→ operator chooses action
→ POST selected sale action
→ backend validates tenant/location/product/action
→ backend determines resulting stock effects
```

Android POS, web POS, and future devices use the same rules.

### 19. Product is defined before Warehouse, Procurement, and POS

Design order:

```text
1. Product Domain
2. Warehouse / Inventory
3. Procurement / Receiving
4. Recipes / Production
5. POS / Sales
6. Taxes / Invoicing / Fiscalization
```

Later ADR order may be adjusted, but none of those modules may introduce a parallel Product master.

### 20. Worked examples

**Coca-Cola.** Category `Beverages > Non-Alcoholic > Carbonated Drinks`. `base_unit = piece`. Manufacturer barcode on the product. Packaging: crate = 24 pieces. Direct sale: 1 piece. Receive 24, sell 1, remain 23.

**Vodka.** Category `Beverages > Alcoholic > Spirits > Vodka`. `base_unit = ml`. Packaging: bottle = 700 ml. Tap 0.03 L deducts 30 ml. Whole bottle deducts 700 ml. Bundles deduct vodka plus other existing products. One warehouse Product.

**Coffee.** `Coffee Beans`, `base_unit = g`. Bag 1000 g, container 3000 g, box 6000 g. Receive one 3 kg container → +3000 g. Espresso → −8 g. Cappuccino → −8 g coffee and −120 ml milk. Warehouse does not know what a cappuccino is; it records the resulting movements.

## Rejected alternatives

- **Separate masters** for raw material, article, POS item, and goods — rejected because every later module would diverge.
- **Tenant from host, slug, or barcode** — rejected; ADR 0001 already forbids client-chosen tenancy.
- **Single `sale_unit` on Product** — rejected; one product has many outputs.
- **Category as tax or POS layout** — rejected; those are different concepts.
- **`consumption_tax_rate` or a raw VAT decimal on Product** — rejected; rates belong to location, period, and a Tax ADR.
- **`consumption_tax_applicable` as the final tax model** — rejected; it is only a temporary classification hook.
- **Shopify-style `ProductVariant` in v1** — rejected; pours and combos are sale actions.
- **Tenant-defined dimensionless UoMs** such as a global “box” — rejected; conversion lives on `ProductPackaging`.
- **Packaging name as the conversion key** — rejected; only `quantity_in_base_unit` converts.
- **Barcode as a numeric type** — rejected; leading zeros must survive.
- **Reusing `sku` as fiscal, supplier, or external id** — rejected; those are later, separate codes.
- **Freely editing `base_unit_id` after first use** — rejected; it corrupts historical quantities.
- **Lifting the base-unit lock by deactivate/reactivate or clone** — rejected; clone is a new product, not a history rewrite.
- **Regenerating `short_name` on every `long_name` change** — rejected; the stored value is authoritative.
- **Hard-delete of used products** — rejected; documents must remain readable.
- **Live Product master as the source for historical documents** — rejected; documents snapshot what they need.
- **Expanding a past sale against the current recipe or bundle** — rejected; audit uses the version at sale time.
- **POS calculating stock deductions** — rejected; the backend resolves sale actions.

## Consequences

### Positive

- One Product can move through purchase, receiving, warehouse, recipe or bundle, POS, stock deduction, and invoice without duplicated masters.
- Simple piece goods, measured pours, production, semi-finished goods, and promotional bundles share the same identity rules.
- Immutable base unit, decimal packaging factors, and normalized barcode/SKU uniqueness protect stock meaning.
- Backend authority lets several POS clients share one rule set.
- Guardrails leave room for tax, allergen, deposit, and sale-action ADRs without painting the Product model into a boolean corner.

### Negative

- The model is larger than a single `articles` table.
- Unit and packaging mistakes are costly; conversions must stay validated and deterministic.
- Several follow-up ADRs are required before warehouse, POS, or fiscalization can be finished.

### Neutral

- Platform UoMs are shared reference data; tenant products and packagings remain tenant-owned.
- First catalog implementation can ship Product, Category, UoM seed, and Packaging without sale actions, recipes, or POS.

## Invariants

1. There is one canonical `Product`.
2. Product is always tenant-scoped. `product_id` alone does not authorize.
3. Warehouse, procurement, and POS do not create their own Product masters.
4. `long_name` is required. `short_name` is stored. It is generated only when empty at create. Later `long_name` edits do not overwrite it. Duplicate `short_name` is allowed. Truncation is Unicode-safe.
5. `sku` is optional, normalized, tenant-unique, case-insensitive, and stable when the name changes. It is not a fiscal, supplier, or external code.
6. Barcode is optional, a string, normalized before uniqueness, and unique in one tenant namespace shared by products and packagings. v1 does not require GTIN/EAN.
7. Product has one base unit for quantity.
8. After first use, `base_unit_id` is immutable. The backend enforces the lock. Deactivate, reactivate, and clone do not lift it. A new base unit requires a new product or a controlled migration.
9. Packagings convert into the base unit with `quantity_in_base_unit > 0` stored as a decimal. The packaging name is not the conversion.
10. Different sale outputs do not require duplicating Product. One Product may have many sale actions.
11. Recipe, bundle, and modifier are different concepts.
12. POS does not compute stock deductions. The backend resolves the sale action, using the version that applied at sale time.
13. Categories are hierarchical and tenant-scoped. POS layout is not a product category.
14. Tax classification on Product is separate from the local rate. Local taxes may depend on business location and period.
15. Capabilities are flags, not a type enum, and follow the invariants in Decision 14.
16. Historical documents must not change because the Product master later changed.

## Follow-up ADRs

```text
Warehouse and Stock Ledger
Procurement and Goods Receiving
Recipes and Production
POS Sales and Sale Actions
Modifiers
Bundles and Promotions
Tax Model
Invoices and Fiscalization
Barcode Generation and Label Printing
```

Business Location is a prerequisite for local tax, stock, and price overrides; it is not defined here.

The next ADR should define **Warehouse / Inventory**, using `Product`, `base_unit`, and the Input / Output rules in this document.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)

## Out of scope

This ADR does not define:

- warehouse ledger or stock movement schema
- purchase orders or goods receipt documents
- supplier model or supplier item codes
- recipe, production-batch, or yield schema
- bundle schema
- POS screen layout, pricing, discounts, or promotions lifecycle
- modifier UX
- invoice schema or fiscalization
- concrete VAT or municipal consumption-tax tables
- internal barcode generation or label printing
- lot, expiry, or serial tracking
- inventory valuation or FIFO / LIFO / weighted average
- stocktaking
- catch-weight scales
- product images or search aliases
- the exact `short_name` shortening algorithm
- the controlled base-unit migration procedure

## Amendment — 2026-08-15: Divisible COUNT products

This amendment changes the rule that every product whose base unit is `piece` must have integer quantities only.

The Decision 7 sentence “piece uses 0 decimal places” remains in the original text. It still applies to ordinary COUNT products. It is **superseded only** for a product explicitly marked divisible.

A COUNT product is indivisible by default. Exceptionally, a product that represents a physical package whose contents are sold or recorded in portions may be marked divisible.

A divisible COUNT product still uses `piece` as its warehouse base unit, but its on-hand quantity may include a fractional part of one piece.

`divisible=false` must not accept a fractional `piece`. `divisible=true` does not mean a whole bottle is sold as `700 ml`. A whole closed bottle remains exactly `1 piece`.

Example:

- product: Vodka 0.7 L
- base unit: `piece`
- divisible: `true`
- declared content: `700 ml`

A whole closed bottle is recorded as `1 piece`.
A `30 ml` serving is the share `30/700 piece`.
A whole-bottle sale is exactly `1 piece`.

A portion is defined as an exact ratio between the portion quantity and the declared package content.

```text
30 ml / 700 ml = 3/70 piece
```

The ratio must not be stored or calculated using binary floating-point. Conversion to ledger quantity follows one canonical fixed-precision and rounding rule defined by the inventory domain.

Declared content quantity and unit are required when a divisible COUNT product is intended for measured portion sale. An estimated `0.5 piece` may exist without declared content.

When declared content is present:

- `declared_content_quantity` must be `> 0`
- `declared_content_unit` belongs to one dimension (for example `VOLUME`)
- portion quantity and declared content must share a compatible dimension
- `ml ↔ g` is forbidden without an explicit later density model
- changing declared content must not rewrite existing Sale Actions or historical postings
- a later historical sale line must freeze the ratio that was used

`divisible` and declared content are Product-level hooks. Exact column names are an implementation concern. This amendment does not introduce a second vodka Product, lot tracking, or density conversion.

## Amendment — 2026-08-16: Product ACTIVE is not on a menu

The original Decision that Product is the warehouse or service identity, and that local visibility is not a second Product, remain in the original text.

ADR 0029 owns menu authoring and publishing. `Product ACTIVE` does not mean the product is on a published menu. Local visibility is this later ADR.

This amendment does not change Product identity, capabilities, or historical snapshots.
