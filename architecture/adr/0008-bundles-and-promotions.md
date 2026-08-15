# ADR 0008: Bundles and Promotions

## Status

Proposed

Amended 2026-08-15: ticket-level discount, manual discount, override, and Comp owned by ADR 0016.

## Date

2026-08-15

## Context

ADR 0002 locked Recipe ≠ Bundle and one canonical `Product`. ADR 0003 locked the stock ledger and the posting contract. ADR 0005 locked SALE recipes and no recursive explode of semi-finished goods. ADR 0006 reserved `BUNDLE` fail-closed until components, nesting, freeze, tenant checks, and price/tax allocation were defined, and marked `product_id` optional only for `NON_STOCK`. ADR 0007 locked modifiers and forbade stock modifiers on the BUNDLE ticket line.

Without this ADR, a breakfast menu would be stored as a recipe, a happy-hour discount would skip the free drink in the ledger, editing a live combo would rewrite open tickets, nested bundles would cycle, and a 15 EUR lunch mixing food and drink VAT would have no deterministic allocation.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the bundle and promotion domain **before** `apps.pos`. Physical schema details belong in a later implementation. The semantics below must not change. It does not activate bundle behaviour in application code.

The governing rule:

```text
A Bundle defines which commercial components are sold together.
A Promotion defines why and how an eligible price is changed.
Neither Bundle nor Promotion directly submits stock movements.
Only a posted POS Ticket creates SALE movements from frozen component effects.
```

A bundle may exist with no promotion. A promotion may apply to an ordinary Sale Action with no bundle.

```text
Bundle structure determines the frozen commercial components delivered.
Promotion rules determine eligible price changes.

Stock is always derived from the actual frozen bundle components and
quantities delivered, never from the discounted amount.

Bundle nesting is forbidden in v1. Missing component, selection, pricing,
or tax allocation data blocks posting.
```

## Decision

### 1. Bundle is not a Recipe

`Recipe` is a kitchen normative. `Bundle` is a commercial grouping of sale offers.

```text
Cappuccino recipe     Breakfast bundle
→ 8 g coffee          → 1 Cappuccino Sale Action
→ 120 ml milk         → 1 Croissant Sale Action
                      → 1 Orange juice Sale Action
```

- A bundle produces no `PRODUCTION_OUT` or `PRODUCTION_IN`.
- A bundle has no `on_hand`.
- A bundle is not a `Product`.
- A bundle does not replace a Recipe.
- Stock comes from frozen Sale Action components.
- Semi-finished goods are not exploded (ADR 0005).
- The commercial bundle price is separate from the physical stock effect.

### 2. `BUNDLE` is an allowed Sale Action type

ADR 0006 reserved `BUNDLE`. This ADR defines the conditions under which a `BUNDLE` Sale Action may be activated and posted.

A `BUNDLE` Sale Action references **exactly one** Bundle version. It has no `product_id`. That `product_id` rule is recorded as a dated amendment to ADR 0006.

```text
DIRECT, PORTION and RECIPE require product_id.
BUNDLE does not have product_id; it references bundle_version_id.
NON_STOCK may have no product_id.
```

```text
Bundle
------
id
tenant_id
name

BundleVersion
-------------
bundle_id
version
status                  # DRAFT | ACTIVE | RETIRED
pricing_mode
currency
components

BundleComponent
---------------
bundle_version_id
sale_action_version_id
quantity
sequence
selection_group
```

- One Bundle belongs to one tenant. All components and their Sale Action versions belong to that tenant. Ids alone do not authorize.
- At most one `ACTIVE` Bundle version. Activate and retire happen in one transaction. `(bundle_id, version)` is unique. Version number is not an authorization key.
- `ACTIVE` and `RETIRED` are immutable. Clone `RETIRED` into a new `DRAFT`.
- Changing components, quantities, selection groups, or bundle list price requires a new version.
- Component `quantity > 0`.
- When activating a new Bundle version, every referenced Sale Action version must be `ACTIVE`. After activate, those references stay valid if a component later becomes `RETIRED`. `RETIRED` remains readable for that Bundle version and for old draft or posted lines. Do not swap in the current `ACTIVE`. Deleting a referenced published version is forbidden.

The BUNDLE **ticket line** still has no stock modifiers (ADR 0007). Selection is among Sale Actions, not modifiers on those actions.

### 3. No nested bundles in v1

A component cannot reference a `BUNDLE` Sale Action.

Allowed components: `DIRECT`, `PORTION`, `RECIPE`, `NON_STOCK`.

A `RECIPE` component still expands its own frozen `SALE` movements (ADR 0006) without exploding semi-finished goods (ADR 0005).

### 4. Fixed components and selection groups

**Fixed.** Always included. Example: vodka bottle + 4 energy drinks.

**Selection group.** The guest or operator chooses from the frozen option set.

```text
Lunch menu
Main dish: choose exactly 1
Side dish: choose exactly 1
Drink: choose 0 or 1
```

```text
BundleSelectionGroup
--------------------
name
min_selections
max_selections
options
```

```text
0 ≤ min_selections ≤ max_selections
```

- Group name, min, max, and offered Sale Action versions are immutable inside the Bundle version.
- A draft bundle line may be temporarily `incomplete`. An incomplete line cannot post. The ticket cannot post until every line satisfies its frozen groups.
- The post API re-validates all selections. It must not trust a UI flag.
- An option outside the frozen Bundle version is forbidden.
- v1: the same option at most once.
- Bundle line quantity scales **all** fixed and selected component quantities.

```text
2 × Lunch menu, chosen Chicken + Fries + Water
→ 2 × frozen Chicken action
→ 2 × frozen Fries action
→ 2 × frozen Water action
```

Line completeness is not a ticket lifecycle state. Ticket states remain those in ADR 0006.

### 5. Freeze on add-line

When a `BUNDLE` Sale Action is added to a draft ticket, the line freezes:

- `sale_action_version_id`
- `bundle_version_id`
- all fixed components and their Sale Action versions, including PORTION numerator/denominator and `recipe_version_id` where ADR 0006 requires them
- selection groups and offered versions
- component quantities
- base bundle price, basis, and currency
- per-component tax classification and allocation basis
- selected options when the operator selects them

A later activate of a Bundle or component Sale Action does not change an existing draft line. Missing required snapshot is fail-closed. Never take current `ACTIVE`.

ADR 0006 merge still applies: bundle lines merge only if bundle version, selections, prices, promotion freeze, and all other 0006 freezes match.

### 6. Promotion is not stock authority

```text
Buy 2, get 1 free
stock:  SALE −3
price:  charged value of 2
```

A Promotion changes price eligibility and allocation, never the physical quantity actually delivered. Price `0` still produces `SALE` if the action has a stock effect (ADR 0006).

### 7. Promotion types and parameter validation

v1 types: `PERCENT_DISCOUNT`, `FIXED_DISCOUNT`, `FIXED_BUNDLE_PRICE`, `BUY_X_GET_Y`.

All money amounts are decimal, never binary float. Currency and precision follow ISO 4217. Excess decimals in configuration are **rejected**, not silently rounded.

- `PERCENT_DISCOUNT`: `0 < percent ≤ 100`. Example: happy hour −20%.
- `FIXED_DISCOUNT`: amount must be positive and in the ticket currency. The final commercial amount must not go below zero.
- `FIXED_BUNDLE_PRICE`: price must be non-negative. Example: components 18 EUR, menu 15 EUR.
- `BUY_X_GET_Y`: see Decision 10.

Not in v1: arbitrary formula code, stacking, cross-tenant promotions, retroactive edit of a posted ticket, changing stock quantity without delivering the goods, coupon codes, manager override, or ticket-level discount.

### 8. Promotion is versioned

```text
Promotion
---------
id
tenant_id
name
type

PromotionVersion
----------------
promotion_id
version
status                  # DRAFT | ACTIVE | RETIRED
snapshots
```

- At most one `ACTIVE` version per promotion. Activate and retire happen in one transaction. `(promotion_id, version)` is unique.
- `ACTIVE` and `RETIRED` are immutable. Clone `RETIRED` into a new `DRAFT`.
- A published version snapshots type, validated parameters, currency, eligibility, and priority.
- Changing any of those requires a new version.

### 9. Eligibility is server-side

Eligibility lives on the published promotion version: tenant, location set, time window in the **location timezone**, weekdays, quantity thresholds, and targets (`sale_action_id` and/or `bundle_id`, not a product name).

v1 supports **automatic line-level promotions only**. The client may request or display a Promotion. The backend checks tenant, location, target, time, and quantity. An arbitrary `promotion_version_id` is not authorization.

If several promotions are eligible, the backend applies the lowest `priority`, then the stable id. Deterministic. No stacking.

**Bind time**

- Time and location eligibility freeze at the moment of a valid **server-side** apply, when the line is added and the backend selects the winner. Apply time is a server-side instant, not a timestamp the POS client sends.
- Freeze `promotion_version_id` plus computed commercial amounts and allocation inputs.
- A quantity change re-checks quantity thresholds against that **same** frozen Promotion version. If the new quantity no longer meets `BUY_X_GET_Y` or another threshold, the promotion effect is deterministically recomputed or removed. Do not pick a different current `ACTIVE` promotion on a quantity change.
- The post API re-validates structural and quantity eligibility from the freeze. It does **not** re-check whether a happy hour is still time-active.
- A promotion effect without a freeze is fail-closed.

### 10. `BUY_X_GET_Y`

```text
buy_x = charged units in one group
get_y = free units in one group
group_size = buy_x + get_y

free_qty = floor(delivered_qty / group_size) × get_y
charged_qty = delivered_qty − free_qty
```

```text
Buy 2, get 1 free
delivered 2 → charged 2
delivered 3 → charged 2
delivered 6 → charged 4
delivered 7 → charged 5
```

- `buy_x` and `get_y` are positive integers.
- Applies only to actions whose commercial quantity is a whole number.
- The promotion does **not** automatically add the free unit. POS must record the quantity actually delivered.
- Free units still participate in the stock effect and in tax allocation.

### 11. Bundle pricing modes

`BundleVersion.pricing_mode`:

- `SUM_OF_COMPONENTS` — list price is the sum of frozen delivered-component standalone prices.
- `BUNDLE_PRICE` — explicit list price on the bundle version.

A promotion `FIXED_BUNDLE_PRICE` overrides the commercial amount. Without a promotion, stock still comes from the components.

### 12. Tax allocation — pro-rata and largest remainder

Mixed tax classifications are allowed on bundle components. Each delivered component keeps the classification from its frozen Sale Action version. Tax **rate** still resolves at ticket post (ADR 0006) and is then stored. A posted ticket never recomputes tax.

Allocation uses only actually delivered components. Unselected selection-group options are not in the basis.

```text
basis_i =
  frozen standalone component price
  × frozen component quantity
  × bundle line quantity
```

- Every taxable delivered component must have a valid non-negative standalone price.
- Total basis must be **positive** when allocating a non-zero bundle amount or discount.
- Mixed currencies block activate and post.
- No final allocation may be negative.
- Compute exact proportional shares. Round each share to the currency minor unit. Distribute leftover minor units by **largest remainder**. Ties: `sequence`, then stable id.
- The final sum must equal the bundle price or the total discount exactly.
- The algorithm is deterministic and decimal. It must not use binary float.
- Used bases, exact ratios, remainders, and final allocations are stored as the **posted snapshot**.

`NON_STOCK` components keep their own classification and participate only if they have a non-zero frozen standalone price. Explicit allocation weights and fiscal sublines wait for the Tax / Fiscal ADR.

## Rejected alternatives

- Merging Recipe and Bundle.
- Treating a bundle as a Product with `on_hand`.
- Nested bundles in v1.
- A promotion that posts `SALE −2` for “buy 2 get 1”, or that automatically adds the free unit.
- Dumping the allocation remainder on the last component by sequence.
- Allocating against unselected selection-group options.
- Client-sent component movement lists.
- Client-imposed `promotion_version_id` as authorization.
- Re-checking the happy-hour clock at post.
- Using a client-supplied apply timestamp.
- Editing groups or components on an `ACTIVE` Bundle version.
- Activating a Bundle version whose component Sale Action versions are not `ACTIVE`.
- Swapping a later-retired component for the current `ACTIVE`.
- Auto-updating a draft line when a new Bundle or Promotion version activates.
- Blocking add-line because a required selection is empty.
- Counting bundle line quantity toward selection min/max.
- Silent rounding of excess currency decimals.
- Arbitrary promotion scripts, stacking, coupons, manager override, or ticket-level discount.
- Cross-tenant promotions.
- Retroactive change of a posted ticket.
- Reopening ADR 0002–0005 or 0007.

## Consequences

### Positive

- A combo is a commercial structure, not a kitchen normative and not a warehouse product.
- A discount cannot hide delivered stock.
- Published bundle groups cannot be edited in place.
- A waiter can add the menu and choose the main afterward; post still fails if the line is incomplete.
- Happy hour binds at server apply and is not rewritten at bill close.
- Mixed VAT on a lunch menu has a deterministic, auditable allocation.

### Negative

- v1 cannot nest combos or stack promotions.
- Changing a live menu’s sides or limits requires a new Bundle version.
- `BUY_X_GET_Y` does not invent the free unit; the operator must record what was delivered.
- Coupon codes and manager overrides wait for a later ADR.

### Neutral

- First implementation can ship flat bundles and automatic line promotions without fiscal XML or loyalty.
- The Tax ADR will own rate tables and fiscal sublines. This ADR owns when allocation binds and how remainder is distributed.

## Invariants

1. Bundle ≠ Recipe ≠ Product ≠ Promotion. Neither Bundle nor Promotion submits stock movements. Only a posted POS Ticket creates `SALE` movements from frozen component effects, through the ADR 0003 writer.
2. A `BUNDLE` Sale Action has no `product_id` and references exactly one Bundle version. Components may be `DIRECT`, `PORTION`, `RECIPE`, or `NON_STOCK`. Nested bundles are forbidden.
3. Activating a Bundle version requires every referenced Sale Action version to be `ACTIVE`. Later `RETIRED` stays readable for that version and for old lines. Never swap to current `ACTIVE`. Published versions are not deleted.
4. Group name, min/max, and offered options are immutable inside a published Bundle version. A draft line may be `incomplete`. An incomplete line or ticket cannot post. The post API re-validates selections.
5. Add-line freezes bundle version, component Sale Action versions, quantities, selections, prices, and allocation basis. Missing freeze is fail-closed. Never take current `ACTIVE`.
6. Bundle line quantity scales delivered fixed and selected component quantities. A promotion never reduces delivered stock quantity and never auto-adds a free unit.
7. Promotions are automatic and line-level in v1. The backend selects by eligibility, then priority, then stable id. The client cannot impose `promotion_version_id`.
8. Time and location eligibility freeze at server-side apply. A quantity change re-checks quantity thresholds on the same frozen Promotion version and recomputes or removes the effect. Post re-validates structure and quantity, not the happy-hour clock.
9. `BUY_X_GET_Y` uses `free_qty = floor(delivered_qty / (buy_x + get_y)) × get_y`. `buy_x` and `get_y` are positive integers. Commercial quantity must be whole. Free units still stock and allocate.
10. Promotion parameters: `PERCENT_DISCOUNT` in `(0, 100]`; `FIXED_DISCOUNT` positive and not below a zero final amount; `FIXED_BUNDLE_PRICE` non-negative. Decimal money only. Excess ISO 4217 decimals are rejected.
11. Allocation basis is delivered components only: standalone price × component quantity × bundle line quantity. Exact ratios, minor-unit rounding, largest remainder, ties by sequence then stable id. Final sum matches the bundle price or total discount exactly. No negative allocation. Posted snapshot stores bases, ratios, remainders, and finals. Tax rate resolves at post (ADR 0006).
12. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Tax Model                       # rate tables, fiscal sublines, explicit weights
Invoices and Fiscalization
Payments
Partial return / refund
POS layout
Nested bundles
Modifiers on expanded component lines
Coupon / manager override / ticket-level discount
```

The next domain ADR should define the **Tax Model**. Rate tables stay there. This ADR already owns when classification vs rate binds for allocated bundle amounts.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)
- [ADR 0005: Recipes and Production](0005-recipes-and-production.md)
- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)
- [ADR 0007: POS Modifiers](0007-pos-modifiers.md)

## Out of scope

This ADR does not define:

- POS screen layout, buttons, or KDS protocol
- application models, migrations, or activation of `BUNDLE` in code
- payments, tenders, or FX
- fiscal device protocol, JIR, or fiscal XML
- e-račun
- accounting journals
- partial return or refund documents
- tax rate tables or different-class fiscal sublines beyond the locked allocation semantics
- coupon codes, manager override, or ticket-level discount
- nested bundles
- modifiers on expanded component lines
- production substitutions (ADR 0005)
- loyalty or membership

## Amendment — 2026-08-15: Manual discount, Comp, and ticket-level discount owned by ADR 0016

The original Decision that v1 promotions are automatic, line-level, and do not stack with each other remains in the original text.

ADR 0016 owns ticket-level discount, manual discount, price override, Comp, and approval.

```text
0008 automatic promotions still do not stack with each other.
0016 applies after those automatic effects:
manual line discount, ticket-level allocation, then Comp.
```

This amendment does not change bundle structure, `BUY_X_GET_Y`, tax allocation, or “stock from delivered quantity”.
