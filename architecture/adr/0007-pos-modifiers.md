# ADR 0007: POS Modifiers

## Status

Proposed

## Date

2026-08-15

## Context

ADR 0002 locked one canonical `Product` and Sale Actions as the way one product is sold. ADR 0003 locked the stock ledger, the posting contract, and 12-decimal `ROUND_HALF_EVEN` conversion of exact portion ratios. ADR 0005 locked SALE recipes, freeze of `recipe_version_id` when a POS line is added, and the rule that a required component cannot disappear without an explicit authorized explanation. ADR 0006 locked Sale Action types, freeze-on-add-line, tax classification on add and tax rate at post, and left modifiers as a hook that must not silently rewrite a frozen recipe.

Without this ADR, oat milk would mutate the published cappuccino recipe, a waiter note would deduct stock, a required side would block adding the dish, editing a modifier group would rewrite an already published Sale Action, two transforms would multiply the substitute instead of the original, and a surcharge with a different tax class would silently inherit the parent rate.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the POS modifier domain **before** `apps.pos`. Physical schema details belong in a later implementation. The semantics below must not change.

The governing rule:

```text
Sale Action defines the default commercial and stock behavior of a sale line.
Modifier records an explicit customer or operator choice that changes that line.
Modifiers never mutate the published Sale Action or Recipe version.
Only the posted POS Ticket turns frozen modifier effects into stock movements.
```

A modifier is predefined, backend-authorized, and frozen on the ticket line. Free text is never inventory authority.

The client selects allowed modifiers on one draft line. The backend applies the frozen base Sale Action and the frozen modifier versions in a fixed order, then writes `SALE` movements through the ADR 0003 writer. The client must not submit a movement list.

```text
A Modifier is a predefined, versioned change to one frozen ticket line.

It is not a Sale Action, not a Recipe edit, and not a guest note.
It cannot invent arbitrary stock movements.
It cannot mutate a published Sale Action or Recipe version.
```

## Decision

### 1. Not a Sale Action

A modifier exists only on one POS ticket line. It cannot replace the base Sale Action, cannot change other lines, and cannot mutate published Sale Action or Recipe versions.

Removing a modifier from a **draft** line restores the frozen base effect plus any remaining selected modifiers. After the ticket is posted, selected modifiers are not edited or deleted.

### 2. Not a guest note

**Structured modifier.** May affect price, stock, and kitchen or bar display. In this ADR the price delta inherits the parent line’s tax classification. A modifier with a different classification cannot be activated.

**Free note.** Snapshot text only. No stock, no automatic price, no tax change, no hidden inventory command. An ingredient or quantity change requires a structured modifier.

### 3. Four types

**`ADD`.** Extra predefined stock effect and optional price. Example: burger + cheese → extra `SALE −1` cheese portion. `ADD` has no `target_recipe_component_id`. It does not rewrite an existing recipe component.

**`OMIT`.** Prevents the planned `SALE` of a targeted frozen recipe component. It does not add stock back.

**`SUBSTITUTE`.** Explicit original component, substitute Product, actual quantity, unit, conversion rule, price effect, and display name. `120 ml` is not automatically `120 g`.

**`MULTIPLY`.** Allowed positive decimal factor from the **modifier version**, not free client input (`×2`, `×0.5`, extra shot `+8 g` as a defined option). Never binary float.

v1 applicability:

- `OMIT`, `SUBSTITUTE`, and `MULTIPLY` apply only to frozen **RECIPE** components. They target `recipe_component_id`, not a product name.
- `DIRECT` and `PORTION` allow **`ADD` only**.
- `NON_STOCK` and `BUNDLE` allow no stock modifiers. `BUNDLE` remains fail-closed per ADR 0006.

### 4. Deterministic apply order

```text
Frozen Sale Action
→ frozen Recipe / PORTION / DIRECT effect
→ OMIT / SUBSTITUTE targeting
→ MULTIPLY
→ ADD
→ canonical SALE movements
```

The same line, the same frozen versions, and the same quantities produce the same `SALE` set. The client does not send the movement list.

v1: at most **one** of `OMIT`, `SUBSTITUTE`, or `MULTIPLY` per `target_recipe_component_id`. Combining any two transforms on the same component is rejected. Apply order must not be used to bypass this. `ADD` is applied separately because it has no existing-component target.

```text
Rejected:
  SUBSTITUTE Milk → Oat Milk
  plus MULTIPLY Milk ×2

Unclear whether ×2 applies to milk or oat milk.
v1 refuses the combination.
```

### 5. Modifier is versioned

```text
Modifier
--------
id
tenant_id
name
type                    # ADD | OMIT | SUBSTITUTE | MULTIPLY

ModifierVersion
---------------
modifier_id
version
status                  # DRAFT | ACTIVE | RETIRED
snapshots
```

- At most one `ACTIVE` version per modifier. Activate and retire happen in one transaction. `(modifier_id, version)` is unique. Version number is not an authorization key.
- `ACTIVE` and `RETIRED` are immutable. Clone `RETIRED` into a new `DRAFT`.
- A published version snapshots type, targets, quantity or factor, substitute product when present, price delta and basis, and display name.
- In v1 a modifier version that declares a tax classification different from the Sale Action version it is offered on cannot become `ACTIVE` and cannot be attached to that group. The priced effect inherits the parent line’s classification at sale time.

Activating a new Modifier version does **not** update groups on already published Sale Action versions.

### 6. Modifier Group is frozen inside the Sale Action version

Which modifiers a line may offer is part of the Sale Action version, not a live catalog lookup.

```text
ModifierGroup  (snapshot on SaleActionVersion)
-------------
tenant_id
sale_action_version_id
name
sort_order
min_selections
max_selections
exclusive              # true ⇒ max_selections = 1
offered modifier versions
```

```text
0 ≤ min_selections ≤ max_selections
exclusive ⇒ max_selections = 1
required group ⇒ min_selections ≥ 1
```

Group name, `min_selections`, `max_selections`, sort order, exclusivity, and the offered **Modifier versions** freeze with the Sale Action version.

- After a Sale Action version is `ACTIVE`, its groups and options cannot be added, removed, or edited.
- Changing a group requires a **new Sale Action version**.
- An existing draft line keeps the old frozen group configuration.
- A modifier not offered on that frozen Sale Action version cannot be selected.
- Tenant-scoped. Ids alone do not authorize.

**Selection count** is the number of distinct selected modifier options in that group. Line quantity does **not** increase the count.

```text
2 cappuccinos + oat milk
selection count = 1
stock effect scales for 2 cappuccinos
```

v1: the same option cannot be selected twice. Option quantity and a per-option maximum are a later hook.

### 7. Required groups validate at post

A waiter may add the dish first and choose the side afterward.

- A draft line may be temporarily `incomplete` when a required group is unsatisfied.
- An incomplete line cannot post.
- The ticket cannot post until every line satisfies its frozen groups.
- The post API re-validates every group from frozen configuration and current selections. It must not trust a UI completeness flag.

Line completeness (`incomplete` / `complete`) is not a ticket lifecycle state. Ticket states remain those in ADR 0006: `DRAFT → POSTED → REVERSED` and `DRAFT → CANCELLED`.

### 8. Freeze on select

Selecting a modifier on a draft line freezes `modifier_version_id` and these snapshots:

- type
- `target_recipe_component_id` when the type is `OMIT`, `SUBSTITUTE`, or `MULTIPLY`
- original Product snapshot for the targeted component
- substitute Product and quantity when the type is `SUBSTITUTE`
- exact quantity, factor, or PORTION ratio parts
- price delta and basis
- display name
- selecting operator
- selected-at time

Deselect on a draft line removes that freeze and restores the base effect plus remaining modifiers.

A quantity change on the same line keeps modifier freezes. **All modifier price and stock effects scale with the base line quantity.** `once-per-line` is **not supported** in this ADR. A later ADR may introduce it only with explicit split and merge rules.

A new select after a newer modifier was activated uses that new `ACTIVE` version only if the **frozen Sale Action version** already offers it. Otherwise the published group still offers the old modifier version.

Missing modifier freeze is fail-closed. Posted line modifiers are immutable.

ADR 0006 merge still applies: two lines merge only if all freezes match, including modifier freezes.

`OMIT` and `SUBSTITUTE` snapshots satisfy ADR 0005: omitting a required component has an explicit authorized explanation — display name, target component, original Product, substitute Product and quantity if any, change type, operator, and time.

### 9. Price, tax, and stock posting

- Price delta is decimal, ticket currency, frozen on select. Price `0` still applies the defined stock effect.
- **v1 tax.** The modifier price inherits the parent line’s frozen tax classification. A modifier with a different classification cannot be activated. A different-class extra (separate fiscal subline or allocation) waits for the Tax / Fiscal ADR. Tax **rate** still resolves at ticket post (ADR 0006) and is then stored. A posted ticket never recomputes tax.
- All resulting `SALE` movements post atomically with the ticket through the ADR 0003 writer. Reversal copies stored movement quantities. It does not recompute ratios.
- Semi-finished extras are not exploded (ADR 0005).

**`ADD` of a measured divisible COUNT extra** freezes all of:

```text
product_id
portion quantity and unit
declared-content quantity and unit
exact numerator and denominator
```

The backend first scales the exact ratio by the ticket line quantity, then canonicalizes **once** (ADR 0003: 12 decimals, `ROUND_HALF_EVEN`). It must not use an already-rounded per-portion quantity. A canonical result of `0` blocks post. Reversal copies the stored ledger quantity.

Other `ADD`s use an explicit base-unit quantity on the modifier version.

## Rejected alternatives

- Client-sent movement lists or targeting “omit milk” by product name.
- Free-note stock or price effects.
- Mutating the published recipe or Sale Action when a modifier is chosen.
- Editing groups on an `ACTIVE` Sale Action version.
- Auto-updating published groups when a new Modifier version activates.
- Blocking add-line because a required group is empty. A draft line may be incomplete.
- Counting line quantity toward `min_selections` / `max_selections`.
- Selecting the same option twice in v1.
- Two of `OMIT`, `SUBSTITUTE`, or `MULTIPLY` on the same `target_recipe_component_id`.
- Using apply order to stack transforms on one component.
- Activating a modifier with a tax classification different from the parent Sale Action / line.
- Treating a different-class surcharge as a silent increase of the parent line in v1.
- `once-per-line` in v1.
- Free client `MULTIPLY` factor.
- `OMIT` as a reversing `+` movement.
- Automatic substitute equivalence (`120 ml` = `120 g`).
- Applying `OMIT`, `SUBSTITUTE`, or `MULTIPLY` to `DIRECT` or `PORTION` in v1.
- Stock modifiers on `NON_STOCK` or `BUNDLE`.
- Freezing modifiers only at post, or silently taking current `ACTIVE`.
- Selecting a modifier not offered on that frozen Sale Action version.
- Rounding a divisible `ADD` per portion, then multiplying by line quantity.
- Recomputing a stored `ADD` ratio on reversal.

## Consequences

### Positive

- A published Sale Action version cannot be changed by later group or modifier-version edits.
- A waiter can add the dish and choose the side before post, without posting an incomplete line.
- `min` / `max` stay about choices, not about how many cups are on the line.
- Conflicting transforms cannot multiply a substitute by accident.
- A different tax class cannot silently ride the parent rate.
- Measured extras use the same exact-ratio ledger rule as PORTION.
- Every omit or substitute keeps an authorized audit snapshot.

### Negative

- Changing sides or limits on a live offer requires a new Sale Action version.
- v1 cannot price an extra at a different VAT class than the dish.
- v1 cannot apply two transforms to one recipe component, even when a kitchen would understand the intent.
- Operators cannot mark a surcharge as once-per-line.

### Neutral

- First implementation can ship the four types, frozen groups, and post-time completeness without KDS, option quantity, or fiscal sublines.
- The Tax / Fiscal ADR will own different-class extras. This ADR owns the fail-closed inherit rule.

## Invariants

1. A modifier belongs to one ticket line. It never mutates a published Sale Action or Recipe version. Free text is never stock, price, or tax authority.
2. Only four types exist: `ADD`, `OMIT`, `SUBSTITUTE`, `MULTIPLY`. `OMIT` / `SUBSTITUTE` / `MULTIPLY` target `recipe_component_id` on `RECIPE` lines only. `DIRECT` and `PORTION` allow `ADD` only. `NON_STOCK` and `BUNDLE` allow no stock modifiers.
3. At most one of `OMIT`, `SUBSTITUTE`, or `MULTIPLY` may apply to one `target_recipe_component_id`. Any combination is rejected. Apply order cannot bypass this. `ADD` has no existing-component target.
4. Apply order is fixed: frozen base → `OMIT` / `SUBSTITUTE` → `MULTIPLY` → `ADD` → canonical `SALE`. Same frozen inputs produce the same movements. The client does not send the movement list.
5. Group name, `min_selections`, `max_selections`, sort order, exclusivity, and offered Modifier versions are immutable inside a published Sale Action version. Changing a group requires a new Sale Action version. Activating a new Modifier version does not rewrite published groups. A draft line keeps the frozen group configuration it was added with.
6. `min_selections` and `max_selections` count distinct selected options. Line quantity does not increase the count. v1: the same option at most once.
7. A draft line may be `incomplete`. An incomplete line cannot post. A ticket cannot post until every line satisfies its frozen groups. The post API re-validates groups and must not trust a UI flag.
8. Select freezes `modifier_version_id` and audit snapshots, including operator and time for `OMIT` and `SUBSTITUTE`. Missing freeze is fail-closed. Posted modifiers are immutable. Lines merge only when modifier freezes also match.
9. All modifier price and stock effects scale with the base line quantity. `once-per-line` is not supported.
10. v1 modifier price inherits the parent line’s tax classification. A modifier with a different classification cannot be activated. Tax rate resolves at ticket post (ADR 0006).
11. A measured divisible COUNT `ADD` freezes product, portion, declared content, and exact numerator/denominator. Scale by line quantity, then canonicalize once. Canonical `0` blocks post. Reversal copies the stored ledger quantity.
12. Only a posted POS Ticket turns frozen modifier effects into `SALE` movements, through the ADR 0003 writer, atomically with the ticket. `OMIT` does not write a reversing `+` movement.
13. Tenant isolation. Ids alone do not authorize.

## Follow-up ADRs

```text
Bundles and Promotions          # required before BUNDLE can activate
Tax Model                       # different-class modifier extras
Invoices and Fiscalization
Payments
Partial return / refund
POS layout
Option quantity / once-per-line
```

The next domain ADR should define **Bundles and Promotions**. `BUNDLE` remains reserved and fail-closed until that ADR exists.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)
- [ADR 0005: Recipes and Production](0005-recipes-and-production.md)
- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)

## Out of scope

This ADR does not define:

- POS screen layout, buttons, or KDS protocol
- bundle structure, nesting, or promotions
- payments, tenders, or FX
- fiscal device protocol, JIR, or fiscal XML
- e-račun
- accounting journals
- partial return or refund documents
- tax rate tables or different-class fiscal sublines
- option quantity or `once-per-line`
- nested modifiers
- modifier availability by location or time
- production substitutions (ADR 0005)
- allergen derivation from selected modifiers
