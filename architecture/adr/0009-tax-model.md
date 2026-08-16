# ADR 0009: Tax Model

## Status

Proposed

This ADR may be documented and merged while `Proposed`. The tax engine must not be implemented or activated from this text.

This ADR must not become `Accepted` until an accountant and the applicable fiscal rules confirm both the simultaneous extraction formula and the concrete rounding mode. After that confirmation, amend the published `calculation_rule_version` with the chosen rounding fields. Do not silently copy inventory `ROUND_HALF_EVEN` (ADR 0003).

## Date

2026-08-15

## Context

ADR 0002 locked tax classification on Product and forbade a raw rate. ADR 0006 freezes classification when a POS line is added and resolves the rate at post. ADR 0007 keeps modifier price on the parent line’s classification. ADR 0008 allocates bundle GROSS amounts among delivered components and left rate tables and fiscal sublines to this ADR.

Without this ADR, a Product would store `vat_rate = 25`, a waiter terminal would send a municipality, a rate change mid-shift would rewrite an open ticket, a lunch menu would invent one VAT class for the parent bundle, and `VAT_EXEMPT` would appear as a silent `0%`.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the tax domain **before** any tax engine. Physical schema details belong in a later implementation. The semantics below must not change once accepted. They do not authorize implementation while the rounding mode is unset.

The governing rule:

```text
Tax Classification defines which taxes may apply to a commercial line.
Tax Rate defines the effective rate for one jurisdiction and period.
Tax Calculation resolves and snapshots the applicable taxes when a document posts.
Fiscalization reports the resulting tax facts but does not calculate or mutate them.
```

A tax rate is never hard-coded on Product, Sale Action, or a POS button. A tenant does not invent legal rates.

```text
Products and sale lines freeze tax classifications, never live tax rates.

At document posting, the backend resolves exactly one applicable rate for
each required tax type from the trusted location jurisdiction and legally
relevant instant.

The complete tax result is stored on the posted document and is never
recomputed from current configuration.
```

[Porezna uprava – Porez na potrošnju](https://www.porezna-uprava.hr/HR_porezni_sustav/Stranice/porez_potrosnja.aspx) establishes that the consumption-tax base is the sales price of drinks **without VAT included**. The versioned shared-base extraction formula is the v1 implementation rule and must be validated against the applicable accounting and fiscal reporting requirements before implementation. The official page does not by itself specify the simultaneous algorithm or rounding.

## Decision

### 1. Four concepts

**Tax type.** `VAT`, `CONSUMPTION_TAX`. Later types do not change the Product model.

**Tax classification.** Treatment of a product or commercial line. Names are conceptual. Do not embed Croatian percentages in enum names.

```text
VAT_STANDARD
VAT_REDUCED_A
VAT_REDUCED_B
VAT_ZERO
VAT_EXEMPT
CONSUMPTION_TAX_ELIGIBLE
OUTSIDE_SCOPE
```

Each classification declares the required `jurisdiction_level`:

```text
VAT classification            → COUNTRY
CONSUMPTION_TAX_ELIGIBLE      → MUNICIPALITY/CITY
```

ADR 0002’s consumption-tax hook (`NONE` / `ALCOHOLIC_BEVERAGE` / `NON_ALCOHOLIC_BEVERAGE`) remains a Product hook. This ADR owns the classification catalog used at freeze and resolve time.

**Tax rate.** A decimal percentage from the legal tax catalog, for type + classification + jurisdiction + period. Source, publication date, and legal reference are required.

**Tax result.** What was actually calculated and frozen on the posted document, at grain `posted_ticket_line + allocated_component + tax_type`:

```text
tax_type
classification
jurisdiction
rate
taxable_base
tax_amount
semantic_outcome            # STANDARD | ZERO_RATE | EXEMPT | OUTSIDE_SCOPE
exemption_reason_code       # required when EXEMPT
legal_reference snapshot
effective_at
calculation_rule_version
```

### 2. Product stores classification, not rate

A Product may store one or more classifications (VAT and consumption tax independently). It never stores the live percentage. A legal rate change does not require editing every product.

Classification freezes when the POS line is added (ADR 0006). Rate resolves when the document posts. The posted document stores both. Later Product or catalog edits do not rewrite history.

A Sale Action version already snapshots tax classification (ADR 0006). Modifier price inherits the parent line (ADR 0007). Bundle components keep their own frozen classifications.

### 3. Legal catalog — not tenant-defined rates

VAT and consumption tax both come from a controlled legal tax catalog. A municipal consumption-tax rate is still a legal fact (a local decision), not a tenant business setting.

```text
TaxRate
-------
catalog_scope               # legal catalog, not tenant
tax_type
tax_classification
jurisdiction_id
valid_from                  # required
valid_until                 # open-ended allowed
rate                        # decimal, never float
status
source
published_at
legal_reference
```

```text
valid_from <= effective_at < valid_until
```

- A tenant cannot create or edit legal rates. The tenant only maps a business location to jurisdictions.
- A tenant override of a legal rate is forbidden.
- Administrative rate entry requires audit and controlled approval.
- Two active rates for the same type, classification, jurisdiction, and instant are forbidden. Overlap blocks activate.
- An unknown rate for the required jurisdiction blocks post. No silent default.
- A published rate is not edited retroactively. A fix is a new version or a controlled dated correction.
- An expired rate stays readable for posted documents.

### 4. Jurisdiction level — no fallback

```text
TaxJurisdiction
---------------
country
administrative_level
code
name

BusinessLocation
→ country jurisdiction
→ local municipality/city jurisdiction
```

- The resolver looks up a rate **only** at the classification’s declared `jurisdiction_level`.
- A local rate cannot override VAT.
- A country rate cannot fill a missing municipal rate.
- There is no city→country fallback and no “nearest found rate”.
- A zero local rate must be an **explicit** legal catalog result, not a missing row.
- Jurisdiction comes from backend location configuration. The POS client does not send a municipality or a rate.
- A missing location jurisdiction for a required tax blocks post.
- An address change does not rewrite history. The posted result freezes jurisdiction code and name.

### 5. Legally relevant instant

```text
tax_effective_at = legally relevant commercial instant chosen by the document type
```

For a POS ticket in v1:

```text
tax_effective_at = server-controlled posting instant
```

The client does not choose the tax instant. Store the UTC instant, the timezone ID, and the derived local business date (location timezone, ADR 0003 / 0006). A rate change while a ticket is open uses the rate valid at post. All lines of one POS ticket share the same tax effective instant. A later fiscal submission does not change the already calculated rate.

### 6. Resolve is fail-closed; exemption needs a reason

For each posted allocated component and each applicable tax type:

```text
0 matching rates → block post
1 matching rate  → use and snapshot
>1 matching rates → configuration error; block post
```

`ZERO_RATE`, `EXEMPT`, and `OUTSIDE_SCOPE` stay distinct. They are not merely `0%`.

- `EXEMPT` requires `exemption_reason_code` and a legal-basis / legal-reference snapshot.
- A tax result with amount `0` is still stored when needed for reporting.
- A missing required exemption reason blocks post.

A tax type that does not apply uses rate `0` in the extract formula and does not create a positive tax amount. An applicable type with a catalog gap still blocks post.

### 7. GROSS commercial prices and input validation

The frozen POS unit price, or the ADR 0008 allocated component amount, is **GROSS**. `base` is derived. It is not the hospitality button source of truth.

- Final GROSS after discount must not be negative.
- GROSS must match ticket currency precision. Excess decimals are rejected.
- `0` is allowed for a genuine free sale. Applicable rates and semantic tax results still resolve.
- Negative amounts are not used as a return. Reversal copies original results with the opposite sign.

v1 does not add different-class modifier fiscal sublines. ADR 0007 remains fail-closed.

### 8. Simultaneous extract from GROSS — shared base

Calculation lives on a versioned `calculation_rule`. Posted results store `calculation_rule_version`.

The locked hospitality formula:

```text
base = GROSS / (1 + VAT_rate + consumption_tax_rate)

VAT = base × VAT_rate
consumption_tax = base × consumption_tax_rate

GROSS = base + VAT + consumption_tax
```

- The calculation is simultaneous from the GROSS amount.
- VAT and consumption tax do not enter each other’s base.
- Math is decimal. Never binary float.
- The exact extract runs first. Rounding happens only under Decision 9.
- After remainder adjustment it must hold: `base + VAT + consumption_tax = GROSS`.

If only VAT applies, `consumption_tax_rate = 0` in the formula. If only consumption tax applies, `VAT_rate = 0`. If neither applies, `base = GROSS`, tax amounts are `0`, and the semantic outcome is `EXEMPT`, `OUTSIDE_SCOPE`, or `ZERO_RATE` as required.

Rejected for v1: putting consumption tax inside the VAT base, or extracting VAT from GROSS first and then consumption tax on the remainder.

### 9. Rounding lives on `calculation_rule_version`

Inventory `ROUND_HALF_EVEN` (ADR 0003) is not automatically the tax rounding mode. Two implementations must not pick different cents.

Each published `calculation_rule_version` must contain:

```text
calculation formula
intermediate precision
money scale
rounding mode
remainder algorithm
bucket order
```

The concrete v1 rounding mode is **not chosen in this ADR**. It is locked only after accountant and fiscal-rule review, then stored on the published rule. Missing any of these fields blocks activate of the rule and blocks post. Do not copy inventory rounding silently.

Remainder **inside one allocated component**, for buckets `BASE`, `VAT`, and `CONSUMPTION_TAX`:

```text
largest fractional remainder
→ tax bucket order from the calculation rule
→ stable tax-type code
```

Bundle `sequence` is only for ADR 0008 GROSS allocation among components. It is not the tax-bucket tie-break.

Document totals are the sum of stored component tax results. Do not re-extract tax from the document grand total.

### 10. Bundle tax grain

- ADR 0008 first assigns a final GROSS to each delivered component.
- This ADR calculates tax separately for each allocated commercial component.
- Each result links to the ticket line and the bundle component snapshot.
- Results are then summed into document tax totals.
- The parent BUNDLE line does not receive one invented tax classification.

```text
unique grain = posted_ticket_line + allocated_component + tax_type
```

A non-bundle line is one allocated component: the line itself.

### 11. Reversal

A full ticket reversal (ADR 0006) copies stored tax results with the opposite sign. It does not re-resolve rates, rerun the extract, or rerun ADR 0008 allocation. A partial refund waits for its own ADR and must not edit the original tax snapshot.

## Rejected alternatives

- `vat_rate = 25` on Product, Sale Action, or a POS button.
- Tenant-created or tenant-overridden legal rates.
- A client-sent rate, municipality, or tax instant.
- City→country or “nearest rate” fallback.
- Treating a missing local rate as `0`.
- Copying inventory `ROUND_HALF_EVEN` onto tax without review.
- Using ADR 0008 component `sequence` as the intra-component tax-bucket tie-break.
- One invented tax classification on the parent BUNDLE line.
- `VAT_EXEMPT` without `exemption_reason_code`.
- Freezing the rate when the line is added.
- Recomputing tax on a posted or fiscalized document.
- Treating `VAT_EXEMPT` as only `0%`.
- Claiming the Porezna uprava page specifies the full simultaneous algorithm and rounding.
- Putting consumption tax inside the VAT base, or sequential extract.
- Negative GROSS as a return.
- Reopening ADR 0008 GROSS allocation.
- Implementing or activating a tax engine from this Proposed ADR.
- Amending ADR 0002–0008 in this change.

## Consequences

### Positive

- A legal rate change does not require editing the catalog of products.
- An open ticket keeps its frozen classifications and takes rates at the server posting instant.
- A tenant cannot invent 25% VAT or a municipal drink tax.
- A lunch menu with mixed classes keeps one tax result per allocated component.
- Exempt sales stay reportable, with a reason, instead of disappearing as `0`.

### Negative

- The tax engine cannot ship until rounding mode and the formula are confirmed.
- Operators cannot post if the legal catalog or location jurisdiction is incomplete.
- v1 cannot put a modifier on a different tax class than its parent line.

### Neutral

- First documentation can merge without fiscal XML or an implemented calculator.
- The Invoices and Fiscalization ADR will report stored tax facts. This ADR owns how those facts are produced — after acceptance.

## Invariants

1. Classification, rate, calculation, and fiscalization are four concepts. Fiscalization does not calculate or mutate tax. A rate is never stored as a live percentage on Product, Sale Action, or a POS button.
2. Product and sale lines freeze classifications. At post, exactly one rate is resolved per required tax type from the legal catalog, the trusted location jurisdiction, and `tax_effective_at`. The complete result is stored and never recomputed from current configuration.
3. VAT and consumption-tax rates come from the legal catalog. A tenant only maps location to jurisdiction. No tenant override. Catalog entry requires source, published-at, legal reference, audit, and approval. Unknown rate blocks post.
4. A classification declares `jurisdiction_level`. The resolver searches only that level. No fallback. An explicit zero is a catalog row, not a missing row.
5. POS v1 `tax_effective_at` is the server-controlled posting instant. One instant per ticket. The client does not choose it.
6. Resolve is fail-closed: 0 or many matches block post. `ZERO_RATE`, `EXEMPT`, and `OUTSIDE_SCOPE` stay distinct. `EXEMPT` requires `exemption_reason_code` and a legal snapshot. A zero-amount result is stored when reportable.
7. GROSS is non-negative and currency-precise. `0` still resolves. Negative amounts are not returns. Reversal copies stored results with the opposite sign.
8. v1 extract is simultaneous: `base = GROSS / (1 + VAT_rate + consumption_tax_rate)`. Neither tax enters the other. The formula is the implementation rule pending fiscal validation. Decimal only.
9. `calculation_rule_version` must include formula, intermediate precision, money scale, rounding mode, remainder algorithm, and bucket order. The concrete rounding mode is unset until review. Missing fields block activate and post. Inventory rounding is not copied.
10. Intra-component remainder: largest fractional remainder, then bucket order from the rule, then stable tax-type code. ADR 0008 `sequence` allocates GROSS only.
11. Tax grain is `posted_ticket_line + allocated_component + tax_type`. The parent BUNDLE line has no invented classification. Document totals are sums of stored results.
12. Tenant isolation. Ids alone do not authorize. This ADR stays `Proposed` and does not authorize a tax engine until the formula and rounding mode are confirmed.

## Follow-up ADRs

```text
Invoices and Fiscalization      # reports stored tax facts; JIR / fiscal XML
Partial return / refund
Payments
POS layout
```

After accountant and fiscal confirmation, a dated amendment to this ADR (or the published `calculation_rule_version`) must record the concrete rounding mode. Then, and only then, may this ADR become `Accepted`.

The next domain ADR should define **Invoices and Fiscalization**. It must not become a second tax calculator.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0002: Canonical Product domain](0002-canonical-product-domain.md)
- [ADR 0003: Warehouse and Inventory](0003-warehouse-and-inventory.md)
- [ADR 0004: Procurement and Goods Receiving](0004-procurement-and-goods-receiving.md)
- [ADR 0005: Recipes and Production](0005-recipes-and-production.md)
- [ADR 0006: POS Sales and Sale Actions](0006-pos-sales-and-sale-actions.md)
- [ADR 0007: POS Modifiers](0007-pos-modifiers.md)
- [ADR 0008: Bundles and Promotions](0008-bundles-and-promotions.md)
- [Porezna uprava – Porez na potrošnju](https://www.porezna-uprava.hr/HR_porezni_sustav/Stranice/porez_potrosnja.aspx)

## Out of scope

This ADR does not define:

- implementation or activation of a tax engine
- the concrete v1 rounding-mode enum value
- POS screen layout, buttons, or KDS
- payments, tenders, or FX
- fiscal device protocol, JIR, or fiscal XML
- e-račun
- accounting journals
- purchase / input VAT
- partial return or refund documents
- different-class modifier fiscal sublines
- explicit bundle tax weights beyond ADR 0008 GROSS allocation

## Amendment — 2026-08-15: Supplier input VAT recorded by ADR 0023

The original Decision that this ADR owns the tax model, and that purchase / input VAT was out of implementation scope here, remain in the original text.

ADR 0023 records input VAT evidence, deductible and non-deductible parts, reverse charge, exemption, jurisdiction, and rule version on the supplier invoice. This ADR still owns the tax model and recoverability math.

This amendment does not change sales-tax calculation, rounding ownership, or fiscal XML.

## Amendment — 2026-08-15: Inbound eInvoice TAX_VALIDATION recorded by ADR 0024

The original Decision that this ADR owns the tax model, and that fiscalization does not calculate or mutate tax, remain in the original text.

ADR 0024 records inbound `TAX_VALIDATION` on a received eInvoice. This ADR still owns the tax model and recoverability math. Intermediary technical ACK is not tax calculation.

This amendment does not change sales-tax calculation, rounding ownership, or outgoing fiscal XML.

## Amendment — 2026-08-15: Accounting export uses stored tax snapshots owned by ADR 0025

The original Decision that this ADR owns the tax model, and that fiscalization does not calculate or mutate tax, remain in the original text.

ADR 0025 exports stored tax snapshots on accounting journal lines. This ADR still owns the tax model and recoverability math. Export must not recalculate tax.

This amendment does not change sales-tax calculation, rounding ownership, or outgoing fiscal XML.

## Amendment — 2026-08-16: Stored-value tax class frozen by ADR 0030

The original Decision that this ADR owns the tax model, and that fiscalization does not calculate or mutate tax, remain in the original text.

ADR 0030 freezes `voucher_tax_class` and the other classification axes. This ADR still resolves rates. A monetary card may be tax SPV or MPV. There is no generic gift-card tax flag.

This amendment does not change sales-tax calculation, rounding ownership, or outgoing fiscal XML.
