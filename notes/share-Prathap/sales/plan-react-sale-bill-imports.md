# Sale Bill imports (quotation · sales order · challan) — React plan

Scope: **the three ways lines reach a sale bill from another document**.
- Quotation import (Ctrl+F3), which replaces the bill.
- Whole sales-order import (Ctrl+F4 on an empty bill), which replaces the bill.
- The Open Sources append for orders and challans (Ctrl+F4 on a bill with
  lines, Ctrl+F5).

It also covers the stock-only refresh all three now share, and the key-by-key
maps from each source line to a bill line.

It is written against the Qt screen **as changed on 2026-09-26**, against the
live server the same day (after notes 51), and against the NestJS source in
`nexerp-api-ro`. Three live checks back it (§8).

Qt source:

    src/modules/sales/salebill/sale_bill_entry.cpp
      applyImportedQuotation()      quotation → bill (replaces)
      applyImportedOrder()          whole order → bill (replaces)
      fillRowFromOrderLine()        one order line → one row; shared by the two order paths
      appendPickedSourceLines()     Open Sources OK → fetch each picked document
      applyPickedSourceLines()      … then fill each pick from its own source line
      fillRowFromDcLine()           one challan line → one row
      refreshImportedStock()        the stock-only item-price refresh (§4)
    src/modules/sales/common/open_sources_dialog.{h,cpp}   the chooser (PickedLine)
    tests/salebill/test_full.cpp  modes quoteimport / orderimport / pickimport

Companion documents this plan does **not** repeat:

- **`plan-react-sale-bill.md` (REV 2, 2026-09-25)** is the whole bill screen.
  This file **overrides its §13.1–§13.4, Q4 and Q5, and the AllowNegative
  lines of §19 and §32** (§1 lists exactly what changes). Everything else there
  stands: the source trail (§13.5), the source chip (§13.6), the charge carry
  (§12.4), the advance carry, the qty rules (§8.6), the `stored | priced`
  union (§5.2), stale responses (§5.4) and the item-price keys (§9.1).
- **`../react-plans/plan-react.md`** owns the pricing engine. The one recompute
  after an import prices totals and never rewrites a stored rate.
- `notes-51-import-godown-negative-stock.md` (this folder) holds the two server
  asks, both closed on 2026-09-26.

> **The whole feature in one line:** an imported line is the source document's
> promise, so it is **filled from its own source line and never re-priced**.
> The only thing fetched fresh is its stock context, looked up **on the godown
> the line will actually draw from**.

---

## 1. What this changes in `plan-react-sale-bill.md` REV 2

| REV 2 says | Now |
|---|---|
| §13.1 "Stock and batch are not on a quotation; they fill on the next touch" | Nothing fills them on a "touch". Take the godown and flag the server **stamps** on `/quotations/get`, then run the stock refresh (§3.1, §4) |
| §13.2 "godown and godown name" + "Do not port: AllowNegative hard-coded" | Godown from the order line; negative stock from **`soiAllowNegativeStock`** (live since notes 51), else the refresh; then the refresh (§3.2) |
| §13.4 a new picked line gets `rate`, `taxPerc`, "level = the document's, then the lookup for the rest" | **Fetch each picked document's `/get` and fill from the source line.** The picker row is a chooser, not a line. No lookup re-price (§3.4) |
| §28 Q4, AllowNegative | **Decided:** the server's effective flag → else the refresh → else true. A challan line is always true (§3.4, §4) |
| §28 Q5, does the lookup overwrite the source rate | **Decided: no.** The lookup writes stock fields only (§4) |
| §19 "Qt hard-codes `"Y"` on the load path" | Qt reads `sbiAllowNegativeStock` since 2026-09-26; `null` → true |
| §2 module layout | adds `domain/imports/*` and `api/sources.ts` (§2 here) |

---

## 2. Module layout (additions to the bill plan's §2)

```
src/features/saleBill/
  domain/
    imports/                       // pure: no React, no fetch
      fromQuotationLine.ts         // §5: sqi* → Partial<BillLine>
      fromOrderLine.ts             // §5: soi* → Partial<BillLine>, (billQty, orderQty, wholeLine) from the caller
      fromDcLine.ts                // §5: sdi* + the pick's names → Partial<BillLine>; per-qty discount rebuild
      stockContext.ts              // §4: merge an item-price reply into a line, stock fields only
      pickPlan.ts                  // §3.4: group picks by document, top-up vs new, missing source lines
  api/
    sources.ts                     // quotations/get, sale-orders/get, delivery-challans/get, bills/open-sources
  state/
    useBillDraft.ts                // thunks: importQuotation, importOrder, appendPicked, refreshStockContext
```

`domain/imports/*` holds every source key spelling. The thunks only fetch,
call these, and dispatch.

---

## 3. The three imports

> **2026-09-26.** Every import path now follows the same rule:
> **fill a line from its own source line at the document's figures, then run a
> stock-only refresh.** Nothing is re-priced. The Qt screen was changed the
> same day, and three live checks back it: `quoteimport`, `orderimport`,
> `pickimport` (§8). Server notes (51) closed the two gaps it met.

The three imports share one shape:

```
source /get  ──►  fillFrom{Quotation,Order,Dc}Line()  ──►  line (stored figures)
                   (pure, §5)                              │
                                                              ▼
                                   refreshStockContext(line) ─ item-price WITH the line's godown_id
                                   (§4, writes stock fields only, never a price)
```

A **re-price** of an imported line is a defect, not a convenience. It would:

- bill an order or quotation at **today's** rate instead of the promised one;
- move the line to the **price row's** godown instead of the one the source
  named;
- for a challan line, judge stock the challan **already moved**, and replace
  the batch it shipped.

### 3.1 Quotation import (Ctrl+F3). It **replaces**

1. On a bill with lines, confirm *"This bill already has items. Discard them
   and import a quotation?"*.
2. Picker = the quotation list. GET `/quotations/get?sqId&sqCompanyId&sqBranchId&sqAccYear`.
3. Reset, then:
   - **Trail:** `srcDocType QUOTATION`, `srcDocId sqId`, **`srcDocYear =
     sqAccYear`** (the quotation's own year, because every transaction table is
     partitioned by year), refno, date.
   - **Customer:** the `sqCust*` snapshot. **PIN cleared** (a quotation has
     none; the lookup fills it). POS from `sqCustStcd` → `sqPosStcd`.
   - **Header:** option ticks (no confirm), salesman (element 1), agent, level,
     terms text, policy snapshot. **Term is not taken from a quotation.**
   - **Lines:** every non-deleted line through `fillFromQuotationLine` (§5),
     **at its quoted figures**. Size is set verbatim **before** the
     quantities, so nothing recomputes it.
   - **Godown and negative stock.** A quotation **stores** no godown. The
     server **stamps** one on every line of `/get`: `sqiGodownId`,
     `sqiGodownName`, and the effective `sqiAllowNegativeStock` for that
     godown. Take both. `sbiGodownId` is **required** on a bill line. Before
     2026-09-26 Qt left it blank, and every quotation-imported bill failed to
     save with a 400 (*"sbiGodownId is required for a new bill line"*).
   - **Charges** replaced, with their ids cleared (the bill writes its own
     charge_detail).
4. One recompute under the bill's context, then customer-detail and
   party-context.
5. **Stock refresh** for every line (§4). `sqiAvailableStock` is the stock
   **on the quotation's date**, so it is only a placeholder until the
   refresh lands.

**Which godown the server stamps** (notes 51, `src/common/utils/sale-line-godown.utils.ts`):
this is ONE rule, shared with `/master-lookups/item-price`:

1. an explicit godown (the lookup's `godown_id`, a line's own column), else
2. the item's **price-row** godown (`item_price_master.ipm_godown_id`, the
   legacy per-item sale godown; a branch-specific row beats a branch-less
   one), else
3. the branch default (`branch_master.br_default_godown_id`), if that godown
   is still live.

Until 2026-09-26 the quotation used step 3 alone and the lookup used step 2,
so a hand-picked line and the same item from a quotation landed in different
godowns. The client never re-derives this. It **takes what `/get` stamped**.

**Defect not to port:** the picker has **no status filter**, so the same
quotation can be billed twice (bill plan §28 Q3). The React picker shows open
(non-converted, non-cancelled) quotations only, or the server must refuse a
second conversion.

### 3.2 Sales order, whole-document import (Ctrl+F4 on an empty bill). It **replaces**

- Picker = pending sales orders. GET `/sale-orders/get?soId&soCompanyId&soBranchId&soAccYear`.
- Refuse on `soStatus ~ CANCEL` or `soFulfilStatus ~ COMPLETED`: *"This order
  is cancelled / fully delivered — there is nothing left to bill on it."*
- **Header:**
  - trail `SALES_ORDER` / soId / **soAccYear** / refno / date;
  - the customer snapshot, **including the PIN**, and POS from `soCustStcd`;
  - option ticks, salesman, agent, **packed-by**, level;
  - **Term = CREDIT iff `soOrderType === 'CREDIT'`**, then due;
  - terms text and policy.
- **Lines, at the PENDING qty**, through `fillFromOrderLine` (§5):
  - `pending = soiPendingQty > 0 ? soiPendingQty : max(0, orderQty −
    deliveredQty − cancelledQty)`. Skip `pending ≤ 0` lines and count them;
  - `billQty = pending` and `orderQty = pending` (the cap);
  - `wholeLine = deliveredQty ≤ 0 && cancelledQty ≤ 0`;
  - godown = **the order line's own** `soiGodownId` / `soiGodownName`;
  - negative stock = **`soiAllowNegativeStock`**, which the server derives on
    that line's godown (notes 51, live 2026-09-26). `null` means "not
    derived": leave it unset, and the refresh fills it.
- **Charges:** populate, **stamp carry facts**, clear ids.
- **Advance carry:** if `soAdvanceBalanceAmt > 0`, the adjust panel prefills
  from the source document (by `srcDocId`, in the **order's** year — *"a March
  advance really does settle an April invoice"*). Once the panel has loaded,
  say: *"This order holds an advance of X, which has been set off against this
  bill. Collect only the balance at settlement."*. The server also applies the
  order's own advance first at post.
- Info *"N line(s) were already delivered in full and were not imported."*.
  If nothing is left: *"Every line on that order has already been
  delivered."*.
- **Stock refresh** for every line (§4). `soiAvailableStock` is the stock
  on the **order's** date, and the order carries no batch.

**Size, and why `wholeLine` exists.** The order stores the size the customer
agreed to, verbatim (`soiSize`, e.g. `12*6*2*10`, meaning ten boards). Carry it
only on a first, whole delivery. On a part delivery it would print the
dimensions of ten boards against a quantity of six. Re-committing the cell
would also push `billQty` back to ten and trip the order-quantity cap. The
test uses delivered/cancelled, **not** pending-vs-ordered, because the two
are in different units: pending is in base units (from `soi_net_qty`), while
ordered is `soi_order_qty`, which may be cases.

### 3.3 Append from orders or challans (Ctrl+F4 on a bill with lines; Ctrl+F5)

- A second order needs `sales.multi_order_bill`. Otherwise: *"This company
  bills one order per bill. Clear the bill first, or turn on
  sales.multi_order_bill."*
- Needs a customer: *"Pick the customer first — only their open documents are
  offered."*
- The Open Sources dialog **appends, never replaces**.
- **The challan import (Ctrl+F5) is always this path.** There is no
  whole-challan import.

### 3.4 Open Sources dialog, and what happens after OK

GET `/bills/open-sources?companyId&branchId&partyId&kind=DC|ORDER&accYear?`
(BARE keys). DC lists POSTED challans with open qty. ORDER lists
CONFIRMED/PARTIAL orders with pending qty.

```
[{ docId, accYear, refno, date, ageDays, pastWindow, purpose (DC), convertRequired?,
   lines: [{ lineId, lineNo, itemId, itemName, unitId, unitName, lotId, batchNo, godownId,
             docQty, openQty, freeQty, rate, taxId, taxPerc, hsnCode }] }]
```

- Tree: document (tristate tick; **red + "— past the return window"** when
  `pastWindow`; header "refno · dd-MM-yyyy · N day(s) old · purpose") → lines
  (tick + Take 0…openQty, default openQty, 3 dp; typing > 0 ticks, 0 unticks).
- Summary: "N line(s) ticked · goods value X". An error is shown as status
  text, not a modal.

**These rows are for choosing, not for filling a line.** A row carries only
rate, tax % and HSN. It has no discounts, no CGST/SGST/IGST split, no
tax-inclusive flag, no MRP, cost, price level or base factor. Before
2026-09-26, Qt filled a picked line from the row, hard-coded `toBaseFactor =
1` and the **document's** level, then ran the full item-price lookup. That
**overwrote the rate with today's price and the godown with the price row's**.
So after OK:

1. **Fetch every picked document once.** Group the picks by `(kind, docId,
   accYear)`. GET `/sale-orders/get` or `/delivery-challans/get`
   (`sdcId, sdcCompanyId, sdcBranchId, sdcAccYear`), all in parallel.
   - **All or nothing:** if any fetch fails, show that error and **append
     nothing**. A half-imported pick is harder to see than none.
   - Ignore replies that arrive after the screen has moved on (bill plan §5.4).
2. **For each pick, in pick order:**
   - **Same source line already on the bill** → top up to `min(have + take,
     orderQty)`, then the qty rules (*"a double-bill waiting to happen"*).
   - **Find the source line** in the fetched document by id (`soiId` /
     `sdiId` = the pick's `lineId`). If it is gone, skip the pick and collect
     `"refno · itemName"`. At the end, warn once: *"These lines are no longer
     on their document and were not added:"* followed by the list.
   - **Fill:** an order line → `fillFromOrderLine(order, line, { billQty: take,
     orderQty: openQty, wholeLine: untouched && take ≥ openQty })`. A challan
     line → `fillFromDcLine(line, pick)` (§5).
   - **Then override from the pick:** `orderQty = openQty` (the cap),
     `billQty = take`, `srcLineNo = lineNo`. The picker's numbers are the
     truth for *this* bill.
3. The header trail is set only by the first document. Every document's refno
   is kept for the chip.
4. One recompute, then **the stock refresh for every appended line** (§4),
   then autosave.

**Challan lines do not move stock again** on the bill. The server skips them in
the stock step (`bill-lifecycle.service.ts`:
`i.srcDocType === 'DELIVERY_CHALLAN'` → `continue`) and in the post. So a
challan line's negative-stock flag is **always true**, and the refresh
**never touches its batch identity**.

---

## 4. The stock-context refresh (all three imports)

`refreshStockContext(lineId)` runs once per imported line, after the fill. It
is the item-price lookup (bill plan §9.1) with **one difference in the request and one
in the response**.

**Request:** the line's own `unit_id`, its own `price_level`, and
**`godown_id` = the line's godown**. Without `godown_id` the lookup falls back
to the price row's godown (§3.1 step 2). That may not be the godown the line
was stamped with, so the stock figure and the negative-stock answer would then
describe a godown the line never draws from.

**Response: write only these fields.**

| Response key | Line field | Rule |
|---|---|---|
| `stock` | stock | always (live beats the source's snapshot) |
| `item_code` | searchCode | always |
| `loyalty_pv` | loyaltyPv | always, then recompute (loyalty points ride on it) |
| `reorder_qty` | reorder | always |
| `godown_id, godown_name` | godown | **only if the line has none** |
| `allow_negative_stock` | allowNegative | **only if the source did not say** |
| `batch_config, decimal_count, item_group_id, item_brand_id, item_section_id, item_category_id` | same | **only if blank** (a challan's `/get` carries ids only) |
| `stock_id, batch_no, batch_date` (+ `expiry_date` if blank) | the batch identity, **as one unit** | only when the line is **not** from a challan **and** its batch is empty or the same `batch_no`. A stock row without its batch number is a line that saves one batch and describes another |

**Never written:** `sales_price`, `cost_*`, `max_price`, `min_price`,
`disc_*`, any tax key, `price_level`, `base_factor`, the freight and loading
keys. Those are the source's promise.

**Stale and failed replies:**

- Key the request by **line id**, never by row index. Drop the reply if that
  line has since been deleted or re-picked with another item. Qt checks the row
  still holds the same item id, which is a NexTable workaround; do not port it.
- **A failure is silent.** An item with no price row answers **404** *"Item
  price not found"*, and the line imported fine without it. Log it, keep the
  source's figures, and if `allowNegative` is still unknown set it to
  **true**.
  - Why permissive: the client stock gate only warns early. The **server**
    is the guard: `/validate` reports `SALES_STOCK_NEGATIVE` as INFO, and
    posting refuses under the item's negative-stock policy.
  - Before this rule, Qt raised one error popup per unpriced line on every
    import.
- While a refresh is in flight, the line shows the source's stock. The
  qty-entry warning may use the stale figure for a moment. That is
  acceptable, because the server decides at post.

## 5. The source-line maps (pure: `features/saleBill/domain/imports/*`)

Each is `(sourceLine, context) → Partial<BillLine>`, with no React and no
fetch. They are the only place these keys are spelled. **Stored** figures are
kept as they arrived (bill plan §5.2): the one recompute after import prices totals and
never rewrites a rate.

**Common to quotation (`sqi*`) and order (`soi*`)**, same suffix:

| Suffix | Line field | Suffix | Line field |
|---|---|---|---|
| `ItemId, ItemName` | itemId, description (**read-only**) | `Rate, ActPrice` | rate, actualPrice |
| `ItemUnitId, UnitName` | unitId, uom | `MaxPrice, MinPrice` | mrp, minPrice |
| `HsnCode, EanCode` | hsn, barcode | `CostPrice, CostPreTax` | cost, costPreTax |
| `GodownId, GodownName` | godown (§3.1 / §3.2) | `ItemDiscPerc, ItemDiscQty` | disc %, disc per qty |
| `AllowNegativeStock` | allowNegative (**only if boolean**) | `SplDiscPerc, SplDiscQty` | spl %, spl per qty |
| `PriceLevel, Remarks` | priceLevel, remarks | `SchDiscPerc, SchDiscQty, BillSchPerc` | scheme %, per qty, bill scheme % |
| `IsTaxIncl, IsPromo, IsFree, IsService, HasFreight` | booleans | `TaxPerc, CgstPerc, SgstPerc, IgstPerc` | tax |
| `Size` | size (**order: only if `wholeLine`**) | `CessPerc, CessPerUnit` | cess %, cess per unit |
| `WeightQty, AvailableStock, ToBaseFactor` | weight, stock (placeholder), toBaseFactor | `FreightQty, LoadQty` | freightPerQty, loadingPerQty |
| `DecimalCount, GroupId, BrandId, SectionId, CategoryId` | same | trail | `srcDocId` = the **document** id, `srcItemId` = the line id (`sqiId`/`soiId`), `srcDocType`, `srcDocYear` = the document's year |

Differences:

- **Quotation only:** `CaseQty, BillQty, LengthQty` as quoted; `BatchNo,
  ExpiryDate`; `BatchConfig`. There is no source line number.
- **Order only:** `billQty` and `orderQty` come from the caller (§3.2 /
  §3.4), **not** from `soiOrderQty`. `CaseQty` and `LengthQty` are **not**
  imported. `SalesmanId` per line; `LineNo` → `srcLineNo`.
- `orderQty` becomes **read-only** on an order or challan line (*"a cap the
  operator can raise by hand is no cap at all"*).

**Challan (`sdi*`)**. `/delivery-challans/get` carries **ids only**. Names come
from the pick (`itemName`, `unitName`); decimals and group/brand/section/
category come from the refresh.

| Key | Line field |
|---|---|
| `sdiItemId, sdiItemUnitId, sdiHsnCode, sdiEanCode` | itemId, unitId, hsn, barcode |
| `sdiGodownId, sdiLotId, sdiBucket, sdiBatchNo, sdiBatchDate, sdiExpiryDate, sdiSerialNo` | **the batch the challan shipped**; never re-allocated |
| `sdiPriceLevel, sdiRemarks, sdiSalesmanId` | priceLevel, remarks, salesman |
| `sdiIsTaxIncl, sdiIsPromo, sdiIsFree, sdiIsService` | booleans |
| `sdiWeightQty, sdiToBaseFactor` (`> 0` else 1) | weight, toBaseFactor |
| `sdiRate, sdiActPrice, sdiMaxPrice, sdiMinPrice, sdiCostPrice` | rate, actualPrice, mrp, minPrice, cost |
| `sdiItemDiscPerc` (**not** `sdiDiscPerc`), `sdiSplDiscPerc, sdiSchDiscPerc` | disc %, spl %, scheme % |
| `sdiTaxPerc, sdiCgstPerc, sdiSgstPerc, sdiIgstPerc, sdiCessPerc, sdiCessPerUnit` | tax |
| — | **allowNegative = true** (§3.4) |

- **`sdiDiscPerc` is the combined figure** the DC screen writes
  (`sdiDiscAmt` = item + special + scheme). `sdiItemDiscPerc` is the line
  discount. Reading the wrong one double-counts.
- **Per-qty discounts.** A challan stores each discount as % plus amount, never
  per unit. A per-qty discount therefore arrives as `0 %` with an amount.
  Carry it as `perQty = amount / sdiDcQty` (for item, special and scheme), so
  the bill's recompute lands on the same amount. Otherwise it is silently
  dropped.

---

## 6. Backend gaps this feature inherits

| id | What | Effect |
|---|---|---|
| **DC-NAMES** | `/delivery-challans/get` has ids only: no item, unit or godown names, no decimals, no item group keys | names from the Open Sources row, the rest from the stock refresh (§5) |
| **PRICE-404** | an item with no price row answers `/master-lookups/item-price` with **404** *"Item price not found"*, so an imported line of it gets no refresh | silent; stock stays the source's snapshot; negative stock falls back to true (§4) |
| **ORD-DRAFT** | the whole-order import can post lines from a DRAFT order (bill plan §27) | none client-side |
| **QT-TWICE** | a quotation can be billed twice: no status filter, and no server refusal (bill plan §28 Q3) | picker shows open quotations only |
| **SRC-SYNC** | on 2026-09-26 the local `nexerp-api-ro` copy did **not** match the deployed server: `sale-line-godown.utils.ts` present but unused, no `soiAllowNegativeStock` in `sale-order.service.ts` | **code to the live `/get` bodies**, not the copy; re-read once it is re-synced |

**Closed on 2026-09-26 (notes 51), do not re-raise:**
- `soiAllowNegativeStock` is on `/sale-orders/get`.
- ONE "default godown" rule is shared by `/quotations/get` and item-price
  (§3.1).

---

## 7. Decisions settled here

| # | Question | Decided |
|---|---|---|
| Q4 | Where an imported or loaded line's negative-stock flag comes from | the server's effective flag (`sqi`/`soi`/`sbiAllowNegativeStock`), else the refresh, else **true**; a challan line is always **true** |
| Q5 | Does the lookup overwrite an imported line's rate | **no**; it writes stock fields only |
| I1 | One picked document fails to fetch | append **nothing**, show the error |
| I2 | A picked line is gone from its document by the time of OK | skip it; one warning naming each *"refno · item"* |
| I3 | Item with no price row | silent; keep the source's figures |

---

## 8. Tests

- **Vitest, `domain/imports/*`.** Fixtures are live `/get` bodies saved from the
  box (quotation Q0016, order sor00119, challan dc00045 on Acme / branch
  `019c8ea7-…`).
  - Every map spells every key in its §5 table. A `null`
    `*AllowNegativeStock` leaves the flag unset.
  - Order: `wholeLine` false → no size. `billQty` and `orderQty` come from
    the caller. Case and length are not imported.
  - DC: reads `sdiItemDiscPerc`, not `sdiDiscPerc`. A 0 % discount with an
    amount → per-qty = amount / dcQty, for all three discounts.
    allowNegative is always true.
  - `stockContext`: never writes a price, tax, level or base-factor key.
    Godown and allowNegative are written only when unset. The batch identity
    is written as one unit, and never on a challan line.
  - `pickPlan`: a line already on the bill is topped up, capped at orderQty.
    Missing source lines are collected. Exactly one fetch per document.
- **Component tests (MSW):**
  - one of two document fetches fails → nothing appended;
  - a refresh answering 404 → no popup, allowNegative true;
  - a refresh reply for a line deleted meanwhile is dropped;
  - a quotation import sends `sbiGodownId` on every line (the old 400).
- **Live acceptance** mirrors `tests/salebill/run-full.sh <mode>`. Nothing is
  saved; every imported line is compared with the database. On 2026-09-26:
  - `quoteimport` 8/8: the godown is the one `/get` stamped; the flag is the
    server's; live stock refreshed; the refresh kept the godown; the quoted
    rate is kept; `sbiGodownId` is in the payload.
  - `orderimport` 7/7: the godown is the order line's; the flag equals the
    three-switch rule on that godown, computed in SQL; the order rate is
    kept; the stock row and batch travel together. Also: a missing refresh is
    proved to be a lookup refusal.
  - `pickimport` 17/17: one DC line and one order line. The picker is fed a
    **deliberately wrong rate (1.00)**. Checks: the document rate, discount %,
    tax split, the tax-inclusive flag, the godown, take billed vs open as the
    cap, the DC never stock-judged, the DC's own batch kept.
- **Known weakness of the live set:** the only open documents on the test
  branch are unpriced `E2E-` items at rate 100, 0 % discount and 0 % tax. The
  rate check is a real discriminator (the wrong picker rate). The discount and
  tax checks are not. Add a `ZT-` order and a `ZT-` challan with a discount
  and 18 % tax, both inclusive and exclusive, before calling the React port
  done.

---

## 9. Phases

This is bill-plan phase 5, split:

| Step | Builds | Done when |
|---|---|---|
| 5a | `domain/imports/*` + fixtures | the §8 Vitest suites pass |
| 5b | quotation + whole-order import thunks, the stock refresh | `quoteimport`, `orderimport` pass against the React screen |
| 5c | Open Sources dialog + `pickPlan` + the document fetch | `pickimport` passes, plus the MSW all-or-nothing test |
| 5d | the `ZT-` discount/tax documents (§8) | the discount and tax checks can fail |

---

## 10. Do not port

- **`AllowNegative = "Y"` hard-coded** on quotation, order or load lines.
  The only line that is always true is a challan line, and for a stated
  reason (§3.4).
- **Re-pricing an imported line.** No full item-price lookup on a quotation,
  order or challan line. Until 2026-09-26 that was the Open Sources path: it
  replaced the order's rate with today's price, the godown with the price
  row's, and a challan line's batch; it also hard-coded `toBaseFactor = 1` and
  the bill's price level.
- **Filling a picked line from the Open Sources row.** The row is a chooser,
  not a line.
- **The "fills on the next touch" assumption.** Nothing did the filling; the
  quotation import sent `sbiGodownId: ""` and every save was a 400.
- **One error popup per unpriced line** on import.
- **A stock refresh without `godown_id`**, or one that writes any price key.
- **Matching a refresh reply to its row by index plus an item-id check** (a
  NexTable workaround); key it by line id.
- **`m_loading` around the fill** and the trailing blank row the Qt loops
  step over. Dispatch one `IMPORT_*` action and let the reducer replace the
  lines.
- **Re-deriving the default godown or the negative-stock rule in
  TypeScript.** Both belong to the server (`sale-line-godown.utils.ts`), and
  the client takes what `/get` stamped. The one exception is the refresh's
  fill-if-unset, which only takes the lookup's own answer.
