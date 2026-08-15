# ADR 0006: POS Sales and Sale Actions

## Status

Proposed

## Date

2026-08-15

## Context

ADR 0002 locked one canonical `Product`, Sale Actions as the way one product is sold, and divisible COUNT products. ADR 0003 locked the stock ledger, the posting contract, and 12-decimal `ROUND_HALF_EVEN` conversion of exact portion ratios. ADR 0005 locked SALE vs PRODUCTION recipes and freeze of `recipe_version_id` when a POS line is added.

This ADR owns the commercial POS document and the authorized Sale Action types. Without it, a POS client would send arbitrary deductions, a vodka pour would become a second Product, tax rates would freeze too early or recompute after post, and a half-defined bundle would post stock.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the POS sales domain **before** `apps.pos`. Physical schema details belong in a later implementation. The semantics below must not change.

The governing rule:

```text
Product defines what is sold or consumed.
Sale Action defines the commercial offer and its stock effect.
POS Ticket records the commercial sale.
Only a posted POS Ticket creates SALE movements.
Fiscalization reports the sale but does not independently move stock.
```

The client picks an allowed Sale Action and a quantity. The backend expands the frozen action version into `SALE` movements through the ADR 0003 writer. The client must not submit arbitrary stock deductions.

```text
A Sale Action is the only backend-authorized definition of how a POS
ticket line affects stock.

DIRECT deducts whole base-unit quantities.
PORTION deducts an exact fraction of a divisible COUNT product.
RECIPE deducts the frozen SALE recipe components.
BUNDLE is reserved and cannot activate or post in this ADR.
NON_STOCK creates no inventory movement.
```

## Decision

### 1. Product is not a Sale Action

`Product` is the warehouse or service identity. `Sale Action` is a concrete way to sell it.

One Product — Vodka 0.7 L, `base_unit=piece`, `divisible=true`, declared content `700 ml` — may have several actions: 0.03 L, double 0.06 L, whole bottle. Those are not three warehouse products.

POS button, color, screen group, and layout are not Sale Action. They belong to a later POS layout ADR.

### 2. Sale Action types

**`DIRECT`.** Sells a whole base-unit quantity of one product. Does not use declared content. A whole bottle is `SALE −1 piece`, not `−700 ml`. Coca-Cola is `SALE −1 piece`.

**`PORTION`.** Sells a measured share of a divisible COUNT product.

- Product must have `base_unit=piece` and `divisible=true`.
- Declared content must exist. Portion and content must share a compatible dimension. No `ml ↔ g` without a later density model.
- The effect is an exact ratio, never binary float. Ledger conversion uses ADR 0003: 12 decimals, `ROUND_HALF_EVEN`. A canonical result of `0` is rejected.
- Line quantity first multiplies the exact ratio, then the backend canonicalizes **once**:

```text
qty = 10, portion = 30 ml, declared = 700 ml
stock effect = (10 × 30) / 700 = 300/700 = 3/7
canonical SALE = −0.428571428571 piece
not 10 × round(30/700)
```

```text
1 × 30/700 = 3/70 → −0.042857142857 piece
1 × 60/700 = 3/35 → −0.085714285714 piece
```

**`RECIPE`.** Deducts components of the frozen SALE recipe (ADR 0005). No `PRODUCTION_IN` for the guest cup. Semi-finished products are not exploded. Uses `recipe_version_id` frozen when the line was added.

**`BUNDLE`.** Reserved. Conceptually one commercial line may become several `SALE` movements, and a bundle is not a recipe. Until a later Bundle ADR defines components, nesting, cycles, quantities, freeze of nested PORTION/RECIPE, tenant checks, and price/tax allocation, a `BUNDLE` action **cannot be activated or posted**. Fail closed.

**`NON_STOCK`.** May have price and tax classification. Creates no `SALE` movement and must not invent a stocked Product. Examples: delivery, service, tip, tourist tax.

### 3. Sale Action is versioned

```text
SaleAction
----------
id
tenant_id
product_id              # optional for NON_STOCK
type
name

SaleActionVersion
-----------------
sale_action_id
version
status                  # DRAFT | ACTIVE | RETIRED
snapshots
```

- At most one `ACTIVE` version per action. Activate and retire happen in one transaction. `(sale_action_id, version)` is unique. Version number is not an authorization key.
- `ACTIVE` and `RETIRED` are immutable. Clone `RETIRED` into a new `DRAFT`.
- A published version snapshots type, product identity (id, name, sku, unit, `divisible`, declared content), PORTION quantity and unit, `recipe_version_id` for `RECIPE`, default price, currency, price basis, and **tax classification** (not the rate).
- Changing live Product declared content does not rewrite published action versions (ADR 0002).
- `BUNDLE` versions cannot become `ACTIVE` in this ADR.

### 4. Freeze on add-line; tax rate at post

**Frozen when the line is added**

- `sale_action_version_id`
- product identity snapshots
- unit price, price basis, currency
- tax **classification**
- `RECIPE`: `recipe_version_id`
- `PORTION` must freeze all of:

```text
portion quantity and unit
declared-content quantity and unit
exact normalized numerator and denominator
sale-action version
```

Do not store only the rounded ledger quantity as the PORTION freeze. A quantity change on that line recomputes `(qty × portion) / declared_content` from the frozen exact ratio and canonicalizes once.

**Resolved at post**

- Location and business context live on the ticket.
- Applicable tax **rate** (VAT and consumption tax) is resolved at post from the ticket location and the legally relevant business time.
- The computed rate is then stored permanently on the posted line.
- Missing or invalid tax configuration blocks post.
- A posted ticket never recomputes tax.

A quantity change on the same line keeps commercial freezes. A new line added after a newer action or recipe was activated gets the new `ACTIVE` versions. Post expands stock from stored freezes. Missing commercial freeze is fail-closed. Never take current `ACTIVE` silently.

Reversal copies the **stored** ledger quantity with the opposite sign. It must not recompute `30/700`.

### 5. Ticket lifecycle

```text
DRAFT → POSTED → REVERSED
DRAFT → CANCELLED
```

- Draft does not move stock. `CANCELLED` never produces a stock movement.
- A posted ticket cannot become `CANCELLED`.
- Only `POSTED` creates `SALE` movements, atomically with the ticket, through the ADR 0003 writer (idempotency with payload, unique posting generation, no partial post).
- Full reversal voids the whole posted ticket. Partial return or refund is a later ADR and must not be an edit of the original ticket.
- Fiscalization reports a posted sale. It must not post a second stock effect. Posted or fiscalized tickets are not reopened for edit.

### 6. Place, time, and currency

- One ticket, one `location_id`, one sales `storage_id`. Storage belongs to that location. All `SALE` lines deduct from that storage. Cross-location or cross-storage movement is a transfer first.
- `occurred_at` is a timezone-aware UTC instant. `posted_at` is server-set. Business date uses the location timezone (ADR 0003).
- One ISO 4217 currency per ticket. All lines use it. Currency is immutable at or before the first line. Mixed currencies are forbidden. Pay-in-another-currency and FX belong to a Payments ADR.

### 7. Price, quantity, merge, and zero-price sales

Price is not `Product.price`. The line freezes unit price, basis, and ticket currency (decimal, never float). Line net / tax / gross totaling belongs to the Tax / Invoice ADR.

- Line quantity must be `> 0`. A return is not a negative quantity on a normal draft ticket.
- `DIRECT` on an indivisible COUNT product requires a whole quantity.
- `PORTION` quantity is the number of portions and must follow sale-unit precision.
- Every stock expansion result must be non-zero.

POS may merge two lines only if **all** freezes match: sale-action version, portion ratio or recipe version, price and basis, tax classification, modifiers, storage, and currency. Lines that only share a display name must not merge.

- A promotional line with price `0` is still a POS line and produces a normal `SALE` if the action has a stock effect.
- Staff consumption that is not a sale uses `INTERNAL_USE`.
- Breakage or write-off uses `WASTE`.
- POS is not a shortcut for internal warehouse events.
- Price `0` does not mean no stock effect.

Modifiers remain a hook. They must not silently rewrite the frozen recipe version.

## Rejected alternatives

- POS sending component movements.
- Three Products for three vodka pours.
- `DIRECT` whole bottle as `−700 ml` when `base_unit=piece`.
- Rounding `30/700` ten times inside one line.
- Storing only the rounded ledger quantity as the PORTION freeze.
- Freezing the tax **rate** when the line is added.
- Recomputing tax on a posted ticket.
- Activating or posting `BUNDLE` before the Bundle ADR.
- Treating price `0` as no stock effect or as `INTERNAL_USE`.
- Using POS as a warehouse waste or staff shortcut.
- Negative quantity on a draft ticket as a return.
- Merging lines that only share a display name.
- Mixed currencies on one ticket.
- `CANCELLED` after post, or editing a posted ticket into a partial refund.
- Fiscalization as a second stock post.
- Editing a published action version or a fiscalized ticket.
- Recursively exploding a recipe.
- Inventing a stocked Product for delivery or tip.

## Consequences

### Positive

- One vodka Product supports pours and whole-bottle sales without cloned stock.
- An open ticket keeps the action, recipe, price, and portion ratio that applied when the line was added, while the legal tax rate is taken at post.
- Draft cancel and posted reversal stay distinct. Partial refund cannot rewrite history.
- Zero-price promotions still deduct stock. Staff drinks and breakage stay warehouse events.
- A half-defined bundle cannot post.

### Negative

- POS must persist freezes on the draft line, not only at fiscalization.
- Operators cannot activate bundles until a later ADR.
- Tax configuration must exist at post or the sale is blocked.

### Neutral

- First POS implementation can ship `DIRECT`, `PORTION`, `RECIPE`, and `NON_STOCK` without layout, payments, or fiscal XML.
- The Tax ADR will own rate tables; this ADR owns when classification vs rate is bound.

## Invariants

1. Only a posted POS Ticket creates `SALE` movements, through the ADR 0003 writer. Fiscalization does not move stock by itself. `CANCELLED` never moves stock. A posted ticket cannot become cancelled. A partial refund is not an edit of the original ticket.
2. The client cannot submit arbitrary deductions. Only an authorized Sale Action type defines the stock effect.
3. `DIRECT` deducts whole base-unit quantities. `PORTION` uses a frozen exact ratio and canonicalizes once per line after `qty × portion`. `RECIPE` deducts frozen components and does not explode. `BUNDLE` cannot activate or post. `NON_STOCK` creates no movement.
4. A Sale Action has at most one `ACTIVE` version. Published versions are immutable.
5. Add-line freezes action version, PORTION numerator/denominator, recipe version, price, and tax classification. Tax rate is resolved at post and then stored. A posted ticket never recomputes tax. Missing commercial freeze or tax config is fail-closed.
6. Reversal copies the stored ledger quantity. It does not recompute the ratio.
7. One ticket has one location, one sales storage, and one ISO 4217 currency.
8. Line quantity is `> 0`. Price `0` still produces `SALE` if the action has a stock effect. Staff use and waste are warehouse events.
9. Lines merge only when all freezes match.
10. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Modifiers
Bundles and Promotions          # required before BUNDLE can activate
Tax Model
Invoices and Fiscalization
Payments
Partial return / refund
POS layout
```

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)
- [ADR 0005: Recipes and Production](0005-recipes-and-production.md)

## Out of scope

This ADR does not define:

- POS screen layout, buttons, or KDS
- modifier UX or schema
- bundle structure, nesting, or promotions lifecycle
- payments, tenders, or FX
- fiscal device protocol, JIR, or fiscal XML
- e-račun
- accounting journals
- partial return or refund documents
- price-list / happy-hour engines (price still freezes on add-line)
- tax rate tables
