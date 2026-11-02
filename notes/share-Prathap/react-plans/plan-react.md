# Quotation Entry — React port plan

Scope: **one screen**, `QuotationEntry` (`src/modules/sales/quotation/`), ported to
React/TypeScript against the unchanged backend. The **create** path is the
deliverable (§1–§10); load / edit / delete is specified but scheduled after it
(§11), because everything it needs is the create path plus a paint routine.

Companion documents: `plan.md` (Qt), `db/quotation_list_grid.sql`. The shared
transaction list (`TxnMainView`) and the badge palette are NOT in scope here —
this screen only *consumes* the list, as its F8 picker (§9.5).

---

## 1. What this screen actually is

Three things stacked, and they must be built in this order:

1. **A pure pricing engine.** ~600 lines of arithmetic in `quotation_entry.cpp`
   (`priceRowGoods` → `distribute` → `finalizeRowWithCharges` → totals) plus
   `ChargeGridController::distribute`. Zero React. Everything below is a view of
   its output.
2. **Two editable grids** — items (`tblQuoteItems`, 89 columns, grid layout 18)
   and additional charges (`tblCharges`, grid layout 21).
3. **A header/footer form** — customer block, voucher info, sales info, terms,
   and a display-only totals strip.

The Qt screen's hardest-won rule is "every handler ends in exactly **one**
`recalcDocumentTotals()`". In React that rule disappears: the engine is a
`useMemo` over the draft, so it cannot be called twice or forgotten. Do not port
the discipline; port the arithmetic.

---

## 2. Module layout

```
src/features/quotation/
  QuotationEntry.tsx          // the screen: layout + wiring, no arithmetic
  components/
    CustomerBlock.tsx         // grpCustomer   — ddCustomer + snapshot fields + POS
    QuoteInfoBlock.tsx        // grpQuoteInfo  — refno, dates, validity pair, status
    SalesInfoBlock.tsx        // grpSalesInfo  — beat/agent/salesman/contact, 4 flags
    ItemGrid.tsx              // tblQuoteItems
    ChargeGrid.tsx            // tblCharges
    TotalsStrip.tsx           // grpTotals     — display only, both modes
    TermsBlock.tsx            // grpTerms      — remarks / payment / delivery / T&C
  state/
    draft.ts                  // QuotationDraft + reducer (the ONLY mutable state)
    useQuotationDraft.ts      // reducer + derived pricing + dirty tracking
  api/
    quotation.ts              // get / create / delete
    lookups.ts                // itemPrice, switchUom, customerDetail, freightRates
  payload/
    buildSavePayload.ts       // draft -> POST body (pure)
    parseLoadedDocument.ts    // GET body -> draft (pure, §11)
  validate.ts                 // validateSaveInputs, pure -> Violation | null

src/domain/pricing/           // shared, framework-agnostic (see §3)
  types.ts  lines.ts  charges.ts  document.ts  index.ts
```

`src/domain/pricing` has **no React import and no API import**. It is the piece
that gets unit-tested against the Qt client's numbers, and later reused by sales
invoice / purchase, which is exactly what the Qt code did by sharing
`ChargeGridController`.

---

## 3. The pricing engine — port these exactly

### 3.1 Types

```ts
type Line = {                    // one item row, engine-visible fields only
  itemId: string;
  caseQty: number; billQty: number; lengthQty: number; toBaseFactor: number;
  rate: number; isInclusiveTax: boolean; isFree: boolean;
  discPerc: number; discPerQty: number; discAmt: number;         // one of three
  splDiscPerc: number; splDiscPerQty: number; splDiscAmt: number;
  schPerc: number; schPerQty: number; schAmt: number;
  billSchDiscPerc: number;
  gstPerc: number; cgstPerc: number; sgstPerc: number; igstPerc: number;
  cessPerc: number; cessPerUnit: number;                          // CessUom column
  mrp: number; minPrice: number; actualPrice: number;
  costPrice: number; costBeforeTax: number; weight: number; loyaltyPv: number;
  freightPerQty: number; loadingPerQty: number; hasFreight: boolean;
};

type PricedLine = Line & {       // everything the engine derives — never stored
  netQty: number; rateBeforeTax: number; grossAmt: number; netGross: number;
  discAmt: number; splDiscAmt: number; schAmt: number; billSchDiscAmt: number;
  freightAmt: number; loadingAmt: number;
  chrgBeforeTax: number; chrgAfterTax: number; taxableAmt: number;
  cgstAmt: number; sgstAmt: number; igstAmt: number; gstAmt: number; cessAmt: number;
  total: number; netPrice: number; netPriceBeforeTax: number;
  savingsPerc: number; profit: number; profitBeforeTax: number; rateDiff: number;
};

type VoucherPolicy = {           // SNAPSHOT — never read from the session again
  freightCalcType: string; loadingCalcType: string;
  discountAlterBaseRate: boolean; roundOffStep: number;   // step: no backend column yet, 1
};
```

`PricedLine` is derived, so it is **not** part of the draft. This is the whole
reason the React port is smaller than the Qt one: in Qt, derived values live in
grid cells and have to be written back; here they are recomputed.

### 3.2 `lines.ts`

```
netBillableQty(caseQty, billQty, lengthQty, toBaseFactor)
  = caseQty * toBaseFactor + billQty * (lengthQty > 0 ? lengthQty : 1)
```

Two Pass-1 orders, chosen by `policy.discountAlterBaseRate`:

- **discountFirst** (`priceRowDiscountFirst`): discount `netQty*rate` first, then
  extract tax from the *discounted* rate.
  `discountedRate = base/netQty`; `baseRate = incl ? (discountedRate - cessPerUnit)/(1+(gst+cess)/100) : discountedRate`;
  **`grossAmt === netGross === netQty*baseRate`** on this path — deliberately, and
  documented in the Qt source: the discount columns hold tax-inclusive money, so no
  tax-exclusive gross reconciles with them. Consequence: savings (`gross − netGross`)
  and `freeSchemeAmount` read 0 here. Do **not** "fix" it in the port.
- **standard** (`priceRowStandard`): extract tax from the keyed rate first
  (`baseRate` same formula, off `rate`), `grossAmt = netQty*baseRate` **undiscounted**,
  then `netGross = applyLineDiscounts(grossAmt)`.

Both extract tax only when the line's own `isInclusiveTax` says so — the policy is
document-level, the inclusive flag is per line, so a **mixed document is normal**.

`applyLineDiscounts`, in order, each applied to the running balance:
`disc` (perc else perQty) → `splDisc` → `sch` → `billSchDisc` (percent only).
A keyed `*Amt` wins when neither perc nor perQty is set — that's what the Qt code
does by writing the Amt cell only in the `perc>0` / `perQty>0` branches and always
subtracting the cell. In the port, `applyLineDiscounts` returns the three computed
amounts alongside the balance so the grid can display them; the reducer's
"clear the alternates" rule (§9.2) is what makes the three-way choice unambiguous.

`applyGstAndCess(amount, …)`:
`cess = amount*cessPerc/100 + netQty*cessPerUnit`; then **local** → cgst+sgst at
their own percentages, **inter-state** → igst; `gstAmt = cgst+sgst+igst`;
`amount += gstAmt + cess`.

### 3.3 `charges.ts` — `distribute(lines, chargeRows, {roleOverrides, netTotalOf})`

Port of `ChargeGridController::distribute`. Non-negotiables, each one a bug that
was already fixed once:

1. **Method → amount**: `FIXED` (lump = rate), `QTY` (billQty×rate),
   `NET_QTY` (netQty×rate), `KG|QTL|TON` (weight×rate), `PERCENT` (basis×rate/100).
   `applyOn` (QTY/VALUE/WEIGHT/FLAT) matters **only for FIXED** — it is the
   distribution basis for the lump; FLAT = equal shares, residue on the last line.
2. **`beforeTax` alone** decides whether a share moves the taxable base. `taxApl`
   only governs after-tax rows. A charge row carries its own GST **iff
   `taxApl && !beforeTax`**; every other combination zeroes its whole tax block
   *including the tax %*. Charges never carry cess.
3. **Basis by row**: every row is measured against `netGross` **except a DEDUCT row
   that is not before-tax** — a discount off the already-taxed bill — which is
   measured against each line's **net total**. That is what `netTotalOf(shares)`
   provides: the engine prices the goods-basis rows, calls back with the shares so
   far, the host (here: `document.ts` itself) tags and totals the lines, and returns
   each line's pre-discount net total. Multiple after-tax discounts all measure the
   *same* pre-discount total — no compounding, order-independent.
4. **Manual amount override**: `rate === 0 && amount !== 0` means the operator priced
   the charge by total — spread that amount by `applyOn`, don't recompute from the
   method. (In the grid, typing Amount zeroes Rate; keying a Rate flips back.)
5. **`roleOverrides`**: `role -> number[]` (per line). A charge row whose role is
   present uses those shares instead of its method computation. This is how
   weight-band freight and item loading reach the bill without being re-flattened.

Returns `{ lines: {beforeTax, afterTax}[], chargeTaxable, cgst, sgst, igst, cess }`
plus, unlike the Qt version, the **priced charge rows** (amount + tax block) — in Qt
`distribute()` wrote them back into `tblCharges`; here nothing is written back, the
charge grid renders what the engine returned.

### 3.4 `document.ts` — `recalcDocument(lines, charges, policy, flags)`

The whole pipeline, single direction, no circularity:

```
Pass 1   priceRowGoods per line              -> netGross, grossAmt, rateBeforeTax,
                                                netQty, freightAmt, loadingAmt
basis    LineBasis[] = {netGross, billQty, netQty, weight*billQty}
override FREIGHT / LOADING / UNLOADING per-line amounts, gated on
         flags.hasFreight/hasLoad/hasUnload AND policy calc type !== 'MANUAL'
distribute(...) with netTotalOf
Pass 2   finalizeRowWithCharges per line:
           taxable = netGross + chrgBeforeTax
           applyGstAndCess(taxable)
           total   = taxed + chrgAfterTax
           netPrice = total/netQty ; netPriceBeforeTax = taxable/netQty
           savingsPerc = mrp>0 ? (mrp-netPrice)/mrp*100 : 0      // 2 dp, stored
           profit = netPrice-costPrice ; profitBeforeTax = netPriceBeforeTax-costBeforeTax
totals   see §3.5
```

`netTotalOf` runs Pass 2 on a *copy* — in Qt it wrote the cells and re-read them,
which is only safe because the real Pass 2 immediately overwrites them. In TS it
must be a pure function returning `number[]`, not a mutation.

### 3.5 Totals (`DocumentTotals`)

Per-line sums: gross, item/spl/scheme/billScheme discount, taxable, cgst, sgst,
igst, cess, gst, line total, qty (billQty), weight×billQty, cost×netQty,
profit×netQty, profitBeforeTax×netQty, loyaltyPv×netQty, rateDiff×netQty,
freeSchemeAmount (grossAmt of `isFree` lines).

Then:

```
docTaxable = Σ taxableAmt + charge.chargeTaxable
docCgst/Sgst/Igst/Cess = Σ line + charge.*
docTax     = cgst + sgst + igst + cess
chargeOwnTax = charge.cgst + charge.sgst + charge.igst + charge.cess
amount     = Σ lineTotal − freeSchemeAmount + chargeOwnTax
bill       = step>0 ? floor((amount + step/2)/step)*step : amount
roundOff   = bill − amount
```

Two stat labels, and they measure **different things** — keep them different:

- **Saving** is the customer's, vs MRP: `Σ(mrp−netPrice)*netQty` over lines with
  `mrp>0` only, as a % of `Σ mrp*netQty` over those same lines. A line with no MRP
  is outside **both** sides.
- **Margin** is the company's, pre-tax: `Σ profitBeforeTax*netQty` as a % of
  `Σ taxableAmt`. Charge revenue is outside both sides.

Percentages that get **persisted** (`marginPerc`, `mrpSavingsPerc`, per-line
`savingsPerc`) are rounded to 2 dp *inside the engine*, not at paint time — the
cell, the print and the stored column must all show the same figure.

**Invariant, asserted on every golden test (§12):**
`bill === docTaxable + docTax + afterTaxFlatCharges === Σ lineTotal + chargeOwnTax`
(before round-off).

### 3.6 Money

Postgres is `numeric(15,2)`; JS numbers are doubles. Engine arithmetic in
`decimal.js-light`, `number` only at the API boundary and for display. Round where
the Qt client rounds or the two clients disagree on the same document:

| value | dp | why |
|---|---|---|
| currency cells | 2 | matches `NexTable` Currency storage |
| `rateBeforeTax`, `netPriceBeforeTax` | 12 | **divisors, not amounts** — the Qt code makes these `Number`, not `Currency`, for exactly this reason |
| `marginPerc`, `mrpSavingsPerc`, `savingsPerc` | 2 | persisted |
| bill | round-off step (§3.5) | |

Display is Indian 3-2 grouping: one `formatCurrency(value, precision, showZero)`
mirroring `NexNumberFormat::formatCurrency`, zero-suppression included.

---

## 4. Draft state — the only mutable thing

```ts
type QuotationDraft = {
  mode: 'entry' | 'browse';
  // tenant context — the VOUCHER's, seeded from session for a new document and
  // overwritten from the record on load. Every lookup and the save read THESE.
  companyId: string; branchId: string; accYear: string;
  // identity — server-owned, held so an update hands them back untouched
  docId: string; quoteSlno: string; revisionNo: number; docType: string;
  isNewEntry: boolean; status: string;                       // new -> 'DRAFT'
  policy: VoucherPolicy;                                     // §5
  customer: CustomerSnapshot;                                // §6
  header: { refno, quoteDate, validUntil, validityDays, contactPerson, contactNo,
            beatId, agentId, salesmanId, posStateCode, posStateName,
            hasFreight, hasLoad, hasUnload, hasPromo, priceLevel };
  terms:  { remarks, paymentTerms, deliveryTerms, termsConditions };
  lines: Line[];            // NO trailing blank row — see §9.1
  charges: ChargeRow[];
  isLocalSale: boolean;
  freightRates: FreightBand[];   // cache, keyed by the customer's distance
};
```

Everything else on screen is `useMemo(() => recalcDocument(...), [lines, charges, policy, flags, isLocalSale])`.

One reducer, one `dispatch`. The reducer never calls the API; thunks in
`useQuotationDraft` do the fetching and dispatch the result.

---

## 5. Policy is a snapshot, not a setting

`seedDocumentPolicy()` copies `freightCalcType`, `loadingCalcType`,
`discountAlterBaseRate` out of the session **exactly twice**: when the screen
mounts for a new quotation, and on Clear. From then on `draft.policy` is the only
source; `recalcDocument` takes it as an argument and must never read the session.
An operator can change the company setting between two quotations, and a document
has to keep pricing the way it was created.

`roundOffStep` has no session source and no header column — it stays at 1, in one
place, marked as such (§13.4).

On load, the policy comes off the document (`sqFreightCalcType`,
`sqLoadingCalcType`, `sqDiscAlterBase`); absent/legacy → session default. That is
the only reason those columns are being persisted at all.

---

## 6. Customer snapshot

Picking a customer (`ddCustomer`, dropdown 39) fires
`GET master-lookups/customer-detail?customerId&companyId&branchId&regional=true`
and fills a `CustomerSnapshot` **held on the draft**, not re-read from the master
later: a saved quotation must print the customer it was raised for even after the
master is edited.

Fields consumed: `cust_id/name/ename/phone1/eadd1..3/place/pin`, `gst_no`,
`ecommerce_gstin`, `gst_type`, `state_code`, `state_name`, `area_id`, `area_name`,
`distance_km`, `allow_discount`, `debit_allowed`, `debit_days`, `debit_limit`,
`overdue_billing`, `local_sales`, `price_level` (**1-based** → combo index − 1;
passing it through unshifted put a level-1 customer on level 2), and the four flags
`freight_charge` / `cooly` / `unloading_charge` / `allow_promotion` → the header
checkboxes.

Side effects, in this order: set `isLocalSale`; write the POS dropdown; apply the
price level; and **only if area or distance changed and the policy isn't MANUAL**,
refetch freight bands (`GET master-lookups/freight-charge?distanceKm`).

Not yet wired (no widget): `cust_disc_perc`, `cust_points`, `billed_date`,
`tcs_company`.

---

## 7. Place of supply → which tax the document carries

`ddPosStcd` (dropdown 21, default `33`/Tamil Nadu). `isLocal = stateCode === session.companyStateCode`.
It drives two things and nothing else: `applyGstAndCess`'s cgst+sgst vs igst branch,
and the own-GST split of after-tax taxable charge rows. In React it is one boolean
in the draft feeding the engine — the Qt dance of "flag, then unconditional
recompute, with the controller's own signal blocked" is not needed.

An empty state code is ignored (dropdown cleared ≠ re-tax the document).

---

## 8. The two grids

Both are server-configured: `useGridColumns(18)` for items, `useGridColumns(21)`
for charges. The column *layout* (order, width, visibility, heading) comes from
`/grid-columns/list`; the column *meaning* is the `QuoteItemCols` enum, which
becomes a TS const map keyed by the same backend column names.

### 8.1 Item grid

- ~89 columns, most display-only. **Editable**: BatchNo (when `batchConfig`),
  OrderQty, CaseQty, BillQty, LengthQty, SchemeFlag, IsFree, PriceLevel, Rate,
  DiscPerc/PerQty/Amt, SplDiscPerc/PerQty/Amt, SchPerc, BillSchDiscPerc,
  CashDiscPerc, FreightPerQty (when HasFreight), LoadingPerQty, Remarks, UnitId.
- **Do not port the `"Y"`/`"N"` text flags.** They caused a silent always-true bug
  in Qt (`QVariant::toBool()` reads `"N"` as true; `isInclusiveTax` was permanently
  on in both pricing paths *and* in the save payload). In TS they are `boolean` and
  the bug class is gone at compile time. Convert at the API boundary, both ways.
- Item picker: typing in Description opens popup grid **71**; picking calls the
  price lookup (§8.3). Warn — don't block — when the item is already on another row.
- Virtualise rows (TanStack Virtual). 89 columns × N rows is where a naïve table dies.

### 8.2 Charge grid

Popup grid **82** on the charge-name column; picking snapshots the whole charge
from `charges/get?chgId`. Editable: Rate, Unit, Amount, Remarks. Typing Amount
zeroes Rate (§3.3.4). Rate is not editable on a FIXED row. Rate-driven rows paint a
unit suffix (`%`, `/Qty`, `/NetQ`, `/Kg`, `/Qtl`, `/Ton`) **at paint time only** —
the cell value stays a bare number.

### 8.3 Item price lookup

`GET master-lookups/item-price-detail` with `itemId, unitId, companyId, branchId,
accYear, priceLevel (= comboIndex+1), regional, loadingCalcType, freightCalcType`
— company/branch/**accYear from the draft**, never the session. Fills ~35 columns
(see `fetchItemPrice`): ids, names, stock, weight, base factor, all four rates,
MRP/min/cost/cost-pre-tax, disc defaults, the full tax block, and the three flags.

`params.customerId`, `godownId`, `enableLoading` have no UI yet and go unset —
which risks the wrong price tier (§13.7).

F4 on a row → `GET master-lookups/item-switch-uom?itemId&unitId`, then re-run the
price lookup with the returned unit. **The backend owns the cycle**; nothing is
scaled client-side and nothing is written until the lookup returns.

---

## 9. Interaction rules worth porting deliberately

### 9.1 No trailing blank row
Qt keeps a blank last row as the "add item" affordance, and *every* loop on the
screen has to remember `rowCount() - 1`. React renders an explicit "add row" affordance
instead: `lines` contains only real lines. This deletes the off-by-one from
`buildSavePayload`, `changePriceLevel`, `applyFreightRates`, `validate` and the
totals loop at once. A line with no `itemId` is still skipped at save.

### 9.2 One-of-three discount inputs
Editing `discPerc` clears `discPerQty` + `discAmt` (and the two symmetric cases);
same for the spl-disc trio. One reducer case, `SET_LINE_FIELD`, with a small
`ALTERNATES` map — not six handlers.

### 9.3 Validity pair
`validUntil` and `validityDays` are two views of one period counted from
`quoteDate`. Editing days (or moving the quote date) re-derives the date; editing
the date re-derives the days, floored at 0. A `validUntil` **before** `quoteDate` is
left standing as keyed and rejected at save (§10) — that is deliberate.
Controlled inputs mean **no `QSignalBlocker` equivalent is needed**; the Qt code
needs six of them and one on load.

### 9.4 Price level
`comboPriceLevel` is filled async (`priceLevels`). In Qt an index set before the
items arrive is silently lost, which needed `m_pendingPriceLevelIndex` and a
completion callback. In React the select renders from query data — there is nothing
to overwrite, so **drop the pending-index machinery entirely**.

Keep: `CTRL+1..4` shortcuts scoped to the item grid (count from
`session.priceLevelCount`, capped at 4); the "Apply to Selected / Apply to All"
prompt; the `session.editPrice` gate; the refusal in browse mode (in Qt a QShortcut
bypasses the grid's read-only branch, so it needs its own guard — in React, gate the
hotkey scope on `mode`); and the rule that only "Apply to All" commits the
document's level.

### 9.5 F8 → the shared list as a picker
`TxnMainView` in pick mode, built from the quotation descriptor, so the grid, the
date range, the search and the status badges are identical to the menu's list.
Returns the row's whole `TxnDocKey` — **company/branch/accYear included**, which is
what lets another branch's document load correctly. One promise-returning hook
replaces the signal + `QPointer` guard:

```ts
const key = await pick(quotationDocType);   // TxnDocKey | null
```

Guarded by the same unsaved-work confirm the close path uses.

### 9.6 Hotkeys
`CTRL++` insert line above current, `CTRL+-` remove current (confirm, then
recompute), `F4` switch UOM, `CTRL+1..4` price level, `F8` open list, `Enter` next
cell, barcode-commit jumps to the next row's Barcode cell, `↓` in the barcode editor
bails out to the item picker. All scoped to the item grid, all disabled in browse
mode by gating the scope once — not per handler.

### 9.7 Read-only
`mode: 'browse'` disables the four header groups, the charge grid and the item
grid's editors **together**, through one gate. The totals strip is display-only in
both modes. Save stays clickable and is refused by validation with a reason, rather
than being a dead button.

---

## 10. Validation (`validate.ts`, pure)

In order, first violation wins, returning the field to focus:

1. mode is `entry` (else "this quotation is read-only — press Edit").
2. a customer is picked.
3. `quoteDate` is a real calendar date.
4. `validUntil`, when non-empty, is a real date and **not before** `quoteDate`.
5. per line with an `itemId`: `billQty !== 0`; `rate !== 0` unless the line is free;
   `rate >= minPrice` when `minPrice > 0`; `rate <= mrp` when `mrp > 0` unless
   `session.skipMrp`.
6. at least one line has an `itemId`.
7. every populated charge row has a posting ledger (`cd_ledger_code` is NOT NULL).
8. **role gate** — for each of FREIGHT / LOADING / UNLOADING: if the checkbox is on
   *and* the policy calc type isn't MANUAL *and* the lines compute a positive
   amount, a charge row with that role must exist. An override whose role has no
   charge line is dropped on the floor, so without this gate the operator saves a
   quotation whose freight the customer is never billed for.

There is also a live, non-blocking warning: keying a rate below `minPrice` warns
immediately and reverts the cell (Qt does this in `recalcAfterRateChange`).

---

## 11. Load / edit / delete (after create works)

`GET quotations/get?sqId&sqCompanyId&sqBranchId&sqAccYear` returns header +
`items[]` + `charges[]` in one payload. `parseLoadedDocument` maps it into a draft.
Three rules carry over intact:

1. **Nothing is repriced on load.** Every figure shown is the one that was saved, so
   a document keeps the numbers it was quoted with even after the masters and the
   policy have moved on. In Qt this needs the `m_loading` guard at the top of
   `recalcDocumentTotals()`. In React it's a *type*:

   ```ts
   type Draft = { mode: 'stored'; doc: StoredDocument }   // painted as saved
              | { mode: 'priced'; lines; charges; policy }
   ```
   The first real edit transitions `stored → priced`, and only then does the engine
   run — under **this document's** policy.
2. **Tenant context is the voucher's.** `sqCompanyId/BranchId/AccYear` overwrite the
   draft's; an admin can open another company's or year's document, and every
   subsequent lookup and the save must quote that one.
3. **Server-owned fields are handed back untouched**: `sqQuoteSlno` (string end to
   end — the GET returns it quoted, `"1"`, and a non-numeric series like `QT/0001`
   must survive the round trip; never `parseInt` it), `sqRevisionNo`, `sqDocType`.
   Re-sending 0 / `"Quotation"` renumbers or retypes the voucher.

Soft-deleted lines (`sqiIsDeleted`) are dropped. The tax split of a loaded document
is derived from its **own stored amounts** (cgst+sgst vs igst), falling back to the
POS comparison only for a zero-tax document — the POS alone can't settle it, since
the state it would be compared against is the *voucher's* company's, and only the
session company's state code is on the client.

Open read-only; Edit switches to `entry` after a confirm. Delete targets the loaded
or last-saved `sqId` only, gated on `canDelete`.

---

## 12. Tests

- **Vitest, `domain/pricing` — the priority, and written before any component.**
  Golden cases: inclusive vs exclusive tax; `discountFirst` vs `standard` (they
  **agree** on percentage discounts without per-unit cess and **diverge** for
  per-qty discounts and cess lines — assert the divergence, don't "fix" it); cash
  discount as a before-tax DEDUCT; an after-tax taxable charge carrying its own GST;
  an after-tax DEDUCT measured on net totals with two such rows (assert
  order-independence); a FIXED charge spread by FLAT with the residue on the last
  line; a manual-amount override; a mixed inclusive/exclusive document; the §3.5
  invariant on **every** case.
- **Vitest, `buildSavePayload`** — round-trip: `parse(build(draft)) ≡ draft` for
  every field the payload carries, and the three server-owned fields survive
  verbatim.
- **Vitest, `validate`** — one case per gate in §10, especially the role gate.
- **Playwright** — pick customer → pick item → key qty/rate → totals match a fixture;
  add a cash-discount charge → taxable moves; save → payload snapshot; F8 → load →
  read-only; Edit → first cell edit reprices.
- **Cross-client check** — load one quotation in the Qt client and the React client
  and diff the rendered totals. Cheapest possible guard against decimal drift.

---

## 13. Backend gaps this screen inherits

1. `sqiToBaseFactor` isn't persisted — a loaded line's base factor is back-derived
   from the stored quantities (`netQty = caseQty*toBase + billQty*length`) and is
   **unrecoverable on a line with no case quantity** (loads 0, and the case half of
   any newly keyed NetQty then comes out 0).
2. The GET carries no **area / agent / salesman name**, so those three dropdowns load
   id-only and show blank until re-picked. (charge_detail already joins
   `cdLedgerName` — same treatment is wanted here.)
3. The GET carries nothing for per-line SearchCode, AliasName, Godown, DecimalCount
   (qty precision), BatchConfig, AllowNegative, Reorder, LoyaltyPv, Salesman,
   OrderQty. Loaded lines show no godown and edit at default precision.
4. `sq_round_off_step` doesn't exist and there is no session setting either — the
   step is hard-coded to 1 (§5).
5. The **create response key is unconfirmed** — the Qt code guesses `data.sq_id` and
   says so. Confirm before the React client depends on it.
6. Save is one route for create and update; `sqId`'s presence selects which.
7. `customerId` / `godownId` / `enableLoading` are missing from the item price
   lookup call (§8.3) — risks the wrong price tier.
8. No print pipeline anywhere. Save-and-Print currently only *marks* the request.
   For React the answer is a server-rendered PDF endpoint per voucher/format, not
   client printing; keep both print routes stubbed in one place.
9. No barcode → item endpoint, so the Barcode column can't resolve a scan yet.
10. `applyFreightRates` in Qt ends **without** a recompute, so freshly fetched
    freight doesn't reach the bill until some later edit. In React this cannot
    happen — freight bands are draft state and the engine is derived — so the bug
    is fixed by construction. Note it, don't reproduce it.
11. `sqHasComm` (commission), `sqiAcess*` (additional cess), `sqiAddlDisc1/2`,
    `sqiFreeType`, `sqiSrc*` (PO/SO conversion), `sqCustEmail`, `sqTenantId` have no
    UI and are sent null.

---

## 14. Phases

| # | Deliverable | Done when |
|---|---|---|
| 1 | `domain/pricing/*` — types, lines, charges, document | every golden case in §12 passes, invariant holds; **no UI exists yet** |
| 2 | `state/draft.ts` + `useQuotationDraft` + `recalcDocument` memo | a fixture draft renders correct totals in a test harness |
| 3 | Header blocks: customer + quote info + sales info + terms | customer pick fills the snapshot, POS flips the tax split, validity pair stays in step |
| 4 | `<ItemGrid>` on grid 18 + item picker (71) + price lookup + F4 | picking an item and keying qty/rate prices the line and the document |
| 5 | `<ChargeGrid>` on grid 21 + picker (82) + distribution wired | cash disc moves the taxable base; an after-tax taxable charge adds its own GST |
| 6 | `<TotalsStrip>` + the two stat labels + round-off | totals and both stats match the Qt client on the same fixture |
| 7 | `buildSavePayload` + `validate` + Save / Save-and-Print / Clear | **a new quotation saves and the Qt client opens it with identical numbers** |
| 8 | F8 picker + load + read-only + Edit + Delete (§11) | load → save unchanged → identical stored figures |

Phase 1 before everything else is deliberate, and phases 1–7 are the "create only"
deliverable. The engine is the part that must be right and the only part that is
fully testable without rendering anything.

---

## 15. Do not port

- The trailing blank row (§9.1) and every `rowCount() - 1` that follows from it.
- `"Y"`/`"N"` text flags (§8.1) — booleans, converted at the boundary.
- `m_loading` (§11.1) — a discriminated union, not a mutable guard.
- The pending-price-level index and its completion callback (§9.4).
- `QSignalBlocker` on the validity pair, the POS dropdown and the load path (§9.3).
- Manual recalculation discipline — `useMemo` derives; there is nothing to call
  twice or forget.
- Per-handler read-only guards (§9.7) — gate the hotkey scope and the editors once.
- Writing derived values back into grid cells and re-reading them (`netTotalOf` in
  §3.4) — return values instead.
