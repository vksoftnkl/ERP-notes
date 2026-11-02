# Textile lot tag — gaps against industry textile ERPs (addendum)

Written 2026-10-17. **This is a NEW file. It edits no existing plan.** It reads
`plan-textile-lot-tag.md` (same folder, "the base plan") and lists what textile retail
software (Ginesys, GoFrugal textile, Logic ERP, Wondersoft, Marg garment) does that the base
plan does not. Each gap ends in a decision **G1..G19** with a default.

**Nothing of the base plan is built yet** (checked 2026-10-17: no tag table in the live DB, no
`textile` code in NestJS or Qt). So gaps that change the table cost nothing now and should be
settled before T-1.

| base plan section | what this file does to it |
|---|---|
| §1 table | proposes column changes (§6 summary). The base plan text is not edited |
| §7 flows | adds flows the base plan does not list (§2) |
| §10 reports | adds the textile reports (§3) |
| §14 build order | proposes a revised order (§7) |
| §15 T5 | **G1 supersedes T5** if accepted |
| §15 T3 | **G5 supersedes T3** if accepted |

Live facts used here (2026-10-17): `inventory.item_unit_master.unit_decimal_count` exists; no
size, colour or markup table exists anywhere.

---

## 1. Settle before T-1 (they change the table or a rule)

### G1 — GST slab is decided at billing, not on the tag

The garment slab depends on the **value charged per piece**. Base plan T5 resolves `ilt_tax_id`
from the tag price at reserve and markdown. But a line discount, a bill discount or a promotion
moves the value: a ₹2,799 tag (18%) sold at 10% off is ₹2,519, and at 12% off it is ₹2,463
(5%). Every textile package computes the slab on the net line value at billing.

- **Default:** drop `ilt_tax_id`. The tag stores no tax. The sale bill resolves the rate per line
  from the item's tax (flat-rate items: sarees, fabric) or the company's slab rule
  (`statutory_limits`, per company, never a constant) on the **net per-piece value after every
  discount**, and re-resolves when a discount changes. The server's bill validation applies the
  same rule.
- The tag screen and the label only **show** "tax at this price" (computed, not stored).
- This supersedes base plan T5 and §8.2 "tax (T5)".

### G2 — Cost code on the label

Indian textile shops print the cost on the tag as a letter cipher: a 10-letter word with no
repeated letter maps the digits 1-2-3-4-5-6-7-8-9-0 (e.g. `KRISHNAMUT` → cost 1320 prints
`KRIT`). The salesman bargains from it; the customer cannot read it.

- Setting `stock.textile_cost_code` (per company, in `app_setting_def`; blank = no cost code
  printed). Validation: 10 distinct uppercase letters.
- **Which cost.** The label is printed from the draft, before the lot exists, so the lot's cost is
  not there yet. **Default:** a new column `ilt_label_cost numeric(18,6) NULL` — the cost the
  label prints, **print-only**. It is never used for valuation or COGS; the lot stays the only
  cost for stock (base plan decision 5 holds). Filled from the line's rate at reserve.
- Label dataset field `cost_code` = `translate()` of the rounded cost (a built-in, not a DB
  function, so the SQL rule holds).

### G3 — Selling price from cost by a markup rule

Keying 200 prices by hand is slow and error-prone. The packages fill the price from cost by a
rule, and the user overrides.

- New table `inventory.item_price_markup` (prefix `ipmk_`, because `ipm_` is the price master):
  company, item group (NULL = any), brand (NULL = any), `markup_pct`,
  `price_base` (`MRP` = MRP from cost, `SALE` = selling price from cost), `round_rule`
  (`NONE` / `END_9` / `END_99` / `NEAREST_10` / `NEAREST_50`), priority = most specific wins
  (group+brand > brand > group > any).
- Route `GET /textile-tags/suggest-price?companyId&itemId&brandId&cost` → `{mrp?, salePrice}`.
  The client fills the line; the user can change it. Reserve does not apply it on its own.
- **Default:** build it in T-1; one screen tab "Markup" beside Series.

### G4 — Fabric sold by the metre

Base plan §7: scan adds qty 1. Running fabric is one roll, one label, sold in cuts (2.25 m).

- Rule: when the item's unit has `unit_decimal_count > 0`, the scan does **not** add 1; it opens
  the qty cell for the cut length (the weight-label idea, typed instead of scanned). Pieces
  (decimal 0) keep "scan = 1".
- `ilt_label_qty` = roll length; one label per roll (`labels` = 1 by default for these items).
- **Remnant:** setting `stock.textile_remnant_qty` (per company, default 1.0). The register shows a
  "Remnant" filter: ACTIVE tags of decimal units with 0 < on hand ≤ that qty. A remnant is
  marked down through the normal markdown. No new label, no new lot.
- **Default:** as above, in T-3.

### G5 — Size and colour from lists, not free text

Free text (T3) gives "Maroon / maroon / MRN" and "XL / X-L"; size-wise reports are the main
textile report and break on it.

- New `inventory.item_size_master` (`isz_`): company (NULL = shared), code, name,
  **`isz_sort_no`** (S < M < L < XL < XXL; 28 < 30 < 32), `isz_size_set` (e.g. `ALPHA`, `WAIST`,
  `KIDS`) for the size grid (G11).
- New `inventory.item_colour_master` (`icl_`): company (NULL = shared), code, name.
- Tag columns become `ilt_size_id`, `ilt_colour_id` (FKs) instead of `ilt_size` / `ilt_colour`.
  `ilt_model` stays free text (designs never repeat; that is the point of the plan).
- Quick-add: typing a name not in the list on a tag line asks "Add colour 'Peacock blue'?" and
  creates it.
- **Default:** lists with quick-add, built in T-1. Supersedes T3.

---

## 2. Flows the base plan does not list (add to base §7)

| # | flow | industry | default here | when |
|---|---|---|---|---|
| G6 | **Purchase return by scan** | scan tags into a return to the supplier; the debit note is built from it | purchase-return line takes the tag's lot (`MANUAL` already forces a lot). Status follows on hand as for a sale | T-5 |
| G7 | **Consignment / sale-or-return** | weaver sarees: pay only for what sold; periodic supplier statement | the purchase gets a `CONSIGNMENT` flag; a supplier statement lists sold tags whose source doc is consignment, then raises the payable. The tag already carries supplier + source doc, so **no tag change** | T-5 |
| G8 | **Goods on approval** | pieces go to a customer's home, come back or get billed | DC out + DC return (both carry the lot: `sdi_lot_id`, `sdri_lot_id`) cover it; **no new document**. The DC scan uses the tag scan (base §8.2 already says so) | T-3 |
| G9 | **Hold for a customer** | a piece held for a wedding order | new columns `ilt_hold_party_id`, `ilt_hold_until`, `ilt_hold_note`; status machine unchanged. A scan for another party **warns, never blocks** (counter-never-blocked rule); expired holds clear themselves on read | T-1 columns, T-3 warning |
| G10 | **Exchange at the counter** | return + new sale in one bill | waits on the sale-return module; the tag scan already returns the lot both ways. Recorded here so the return design includes it | sale return |
| G11 | **Size grid entry** | one design row, one qty box per size → one tag per size | client only: a "size set" (G5) row expands to N tag lines before reserve. The reserve route does not change | T-2 (+ purchase in T-5) |
| G12 | **Same product bought again** | option to keep one barcode for standard stock | **No.** Reusing a barcode means the same batch → the same lot → mixed cost. Standard repeat stock (shirts at a fixed price) stays on normal EAN items, not the textile policy. Say so on the item group screen | — |

## 3. Reports (add to base §10, build in T-4)

| # | report | basis |
|---|---|---|
| G13a | **Sell-through** % sold by 30/60/90 days, by purchase doc / supplier / brand | tag source doc + lot inward qty vs outward qty |
| G13b | **Size-wise / colour-wise** stock and sales | needs G5 |
| G13c | **Profit per design / supplier / brand** (margin per rupee of stock) | lot cost × on hand vs sale lines on the lot |
| G13d | **Ageing with value** (at cost, at tag price, at MRP) | base §10 ageing + tag price |
| G13e | **Salesman incentive** per tag / brand, extra for slow stock | sale-bill salesman array; incentive rules are a later screen — **report only now** |

## 4. Smaller items

| # | item | default |
|---|---|---|
| G14 | **Markdown by filter** ("brand X, older than 180 days, 20% off") | client only: the Register's filter + "select all filtered" → the existing `/textile-tags/price`. No route change. T-2 |
| G15 | **Design photo** on the tag (high-value sarees, WhatsApp to customers) | later; leaves room for `ilt_image_ref`. Not in T-1 |
| G16 | **One label per piece** (each silk saree its own tag and lot, so a double scan or a return is exact) | item group flag `textile_label_per_piece` (default off): reserve splits a qty-N line into N tags of qty 1. Off = base plan T2 behaviour. T-1 |
| G17 | **Handheld stock count by scan** | the physical stock screen (menu 45) uses the tag scan; base §7 called it optional — make it **required** for textile. T-3 |
| G18 | **HSN per fabric** | setup, not code: textile item groups are split by HSN (silk / cotton / man-made / garments knitted / garments woven). Note on the go-live checklist |
| G19 | **Alteration / stitching jobs** (blouse, fall & pico) | **out of scope** here; a service-job module if wanted |

## 5. Already better than the packages (keep)

- Per-device series: a till issues numbers offline. Most packages need the server.
- Price on the tag, cost on the lot: never stored twice.
- No stock-table change; one scan = one lot, enforced by `ux_ilt_barcode`.

## 6. Summary of table changes, if G1–G5, G9 are accepted

`inventory.item_lot_tag`:
- **drop** `ilt_tax_id` (G1)
- **add** `ilt_label_cost numeric(18,6) NULL`, print-only (G2)
- **replace** `ilt_size`, `ilt_colour` by `ilt_size_id`, `ilt_colour_id` (G5)
- **add** `ilt_hold_party_id uuid NULL`, `ilt_hold_until date NULL`, `ilt_hold_note varchar(100) NULL` (G9)

New tables: `item_price_markup` (G3), `item_size_master` (G5), `item_colour_master` (G5).
New settings: `stock.textile_cost_code` (G2), `stock.textile_remnant_qty` (G4).
New item group flag: `textile_label_per_piece` (G16).
New route: `GET /textile-tags/suggest-price` (G3).

## 7. Revised build order

| step | base plan | added here |
|---|---|---|
| T-1 | tables, series, reserve / get / update / price / print / void | G1 column drop, G2, G3 table + route, G5 lists, G9 columns, G16 flag |
| T-2 | tag screen + label dataset | Markup tab (G3), size grid (G11), markdown by filter (G14), cost code on the label (G2) |
| T-3 | scan + sale bill / DC / return | bill-time slab (G1), metre qty (G4), hold warning (G9), DC approval (G8), count by scan (G17) |
| T-4 | nightly job + textile stock report | G13a–e |
| T-5 | purchase lot-first | purchase return (G6), consignment (G7), size grid on purchase (G11) |

The tag screen mockup (`textile_tag_ui_mockup.png`) predates this file: it shows free-text
size/colour, a Tax column stored on the line, and no Markup tab. A new mockup is needed after the
decisions below, before any client code.

## 8. Decisions

| # | question | default if nobody answers |
|---|---|---|
| G1 | slab at billing on net value; tag stores no tax? | yes; supersedes T5 |
| G2 | cost cipher on the label; print-only `ilt_label_cost`? | yes; blank setting = not printed |
| G3 | markup rules table + suggest route? | yes, T-1 |
| G4 | decimal unit → scan asks the cut length; remnant filter? | yes; remnant ≤ 1.0 |
| G5 | size/colour lists with quick-add instead of free text? | yes; supersedes T3 |
| G6 | purchase return by scan | yes, with purchase |
| G7 | consignment purchase + supplier statement | yes, with purchase |
| G8 | approval = DC + DC return | yes, no new document |
| G9 | hold columns; warn, never block | yes |
| G10 | exchange in the sale-return design | yes |
| G11 | size grid expands to tag lines (client only) | yes |
| G12 | reuse a barcode for repeat stock | **no**; use EAN items |
| G13 | the five reports in T-4; incentive as report only | yes |
| G14 | markdown by filter (client only) | yes |
| G15 | design photo | later |
| G16 | label per piece, item group flag | yes, default off |
| G17 | count by scan required for textile | yes |
| G18 | HSN split by item group | checklist note |
| G19 | alteration jobs | out of scope |
