# Sale Bill Entry ("sales entry") — React port plan

Scope: **one screen**, `SaleBillEntry` (`src/modules/sales/salebill/`), the
biggest in the application — 5,026 lines of `.cpp` against the quotation's
3,751 — and the only one that takes money. The **create** path is the
deliverable; load / edit / cancel is §16.

Qt source:

    sale_bill_entry.{h,cpp,ui}     the screen
    sale_bill_tender.{h,cpp,ui}    the settlement dialog — its own plan
    sale_tender_delegate.{h,cpp}

Companion plans, and this document does **not** repeat them:

- **`plan-react.md`** (Quotation Entry) owns the **pricing engine**. Every
  arithmetic rule — `priceRowGoods`, the discount ladder, GST/cess, charge
  distribution, document totals — is specified there and is **imported, not
  copied** (§3).
- **`plan-react-sale-tender.md`** owns the settlement dialog in full.
- **`plan-react-sale-order.md`** for the order this bill imports from.
- **`plan-react-printing.md`** for the print pipeline that does not exist yet.

> **The whole screen in one line:** the quotation promises and the order
> commits, but the bill *moves stock, takes money, and posts to the ledgers* —
> so every rule that is merely advisory upstream becomes a gate here, and
> everything it writes has to survive a power cut at the counter.

---

## 1. What the bill is, that the quotation and the order are not

Four things, and they are the whole reason this plan exists separately:

1. **Settlement.** Cash, card, UPI, cheque, credit, gift voucher, loyalty
   points — the tender dialog (§9). No other sales screen has one.
2. **Credits the customer already holds** — advances and credit notes, set off
   against this bill (§10). **An adjustment is never a tender**; conflating the
   two is the single easiest way to get this screen wrong.
3. **It closes things.** Billing an order line consumes it, and removing that
   line asks whether to cancel it on the order too (§8).
4. **It must survive the counter.** Hold, autosave and crash recovery (§12) —
   and that section is where the Qt design does **not** port.

---

## 2. Module layout

```
src/features/saleBill/
  SaleBillEntry.tsx
  components/
    CustomerBlock.tsx        // SHARED with quotation/order + the source chip
    BillInfoBlock.tsx        // refno, date/time, term, due days/date, category
    PeopleBlock.tsx          // salesman, agent, driver, loadman, packed, supervisor
    ItemGrid.tsx             // SHARED — grid 22 (§7)
    ChargeGrid.tsx           // SHARED — the charge grid + auto-apply (§6)
    TotalsStrip.tsx          // SHARED + tendered / adjusted / balance / refund
    CreditPanel.tsx          // §4.2
    AdjustPanel.tsx          // §10 — one component, two mount points
    QuickStrip.tsx           // import quotation / order, hold, pick held
  tender/                    // plan-react-sale-tender.md — SHARED with the order
  state/
    draft.ts  useSaleBillDraft.ts
  persistence/               // §12 — the part that is NOT a file on disk
    holds.ts  autosave.ts
  api/
    saleBill.ts              // create / get / delete(cancel)
    lookups.ts               // customerDetail, partyCredit, itemPrice, freight,
                             // openCredits, quotationForImport, orderForImport
  payload/
    buildSavePayload.ts  buildAdjustmentPayload.ts  parseLoadedBill.ts
    importQuotation.ts  importOrder.ts              // §13
  validate.ts
```

`domain/pricing/` is **imported**. If the bill needs a pricing change, it is a
change to the shared engine with a golden case, or it is not a pricing change.

---

## 3. The pricing engine is imported, and the bill adds exactly two things

Everything from `priceRowGoods` through `finalizeRowWithCharges` to
`recalcDocumentTotals` is the quotation's, unchanged. The structure in the Qt
file is deliberately identical — same function names, same order — precisely so
that they can be one module in React.

The bill adds two of its own, and neither is arithmetic:

- **the negative-stock gate** (§7.2) — a quantity rule, not a price rule;
- **the order-quantity cap** (§7.3) — likewise.

Resist the pull to fork the engine "just for the bill". The two screens
producing different money for the same goods is the defect this structure
exists to prevent.

---

## 4. Customer, and the two panels that hang off it

### 4.1 The snapshot

Same as the quotation: `fetchCustomerDetail` fills name/address/place/phone/
GSTIN, the state code, freight bands, price level, and the loyalty flags. The
snapshot is written onto the document — a bill names the customer *as they were
when it was raised*.

### 4.2 The credit panel

`fetchPartyCredit` → `applyPartyCredit`. `CreditState` carries
`checkEnabled`, `amtExceeded`, `billExceeded`, `availableAmount`, and the
previous bill total. Two independent limits — an **amount** limit and a **bill
count** limit — and either can be exceeded.

Whether exceeding one *blocks* the save is a setting, not a rule this screen
invents. Port the flag; do not hard-code the verdict.

### 4.3 The customer lock

Once a bill carries tendered amounts or adjustments, changing the customer is
not a field edit — it invalidates both. `confirmCustomerChange` asks, and on
confirmation clears them. Port this as an explicit guard in the draft reducer,
not as a `useEffect` that reacts to a customer id changing: the difference is
whether the operator is asked *before* or *after* their settlement disappears.

---

## 5. Place of supply

`sbCustStcd` and `sbPosStcd` are **two different facts**, since 2026-09-11:

- `sbCustStcd` ← the customer master's own state, from the lookup;
- `sbPosStcd` / `sbStateName` ← the POS dropdown (21), operator-overridable for
  a ship-to across a state line.

`isLocalSale` derives from the **POS**, never from the lookup's `local_sales`
flag — that flag is only the fallback when the customer master has no state at
all. Loads read `sbCustStcd` with a fallback to `sbPosStcd`, for records written
before the split.

The default POS is the **company's** state (`companyStateCode` /
`companyStateName`), not a hard-coded 33 / Tamil Nadu.

Whenever the POS changes, the charge grid's inter-state flag must be re-set.
The Qt bug worth not reproducing: `ChargeGridController::clear()` drops the rows
but **keeps** the flag, so a reset after an inter-state bill left the next one's
own-GST charge rows on IGST.

---

## 6. Charges, and the auto-apply seed

The charge grid is shared (`plan-react.md` §8.2). The bill adds
`chg_auto_apply`: on a **new or cleared** document the grid is seeded from
`charges/get?chgModule=S` — the module's charges plus the `B` (both) ones —
keeping the rows flagged auto-apply, ordered by `chgDispOrder` then name.

Four guards make it safe, and all four must port:

1. no-op once the grid holds any charge row;
2. every load / restore / import path clears the grid first, so a seed can
   never survive onto a loaded document — **including one that legitimately has
   no charges**;
3. the module list is cached per session, so from the second document on the
   fill is synchronous and there is no race;
4. lookup failure is silent — a document without its standing charges is worse,
   not broken.

The filter-and-order step is a pure function in Qt purely so it can be tested
without a server. Keep it pure.

---

## 7. The item grid

Grid **22** `SALE BILL - LINES` — 94 columns registered, 89 visible. Shared
component with the quotation and order; virtualise it.

### 7.1 What the bill fills that the others don't

`GodownId` / `StockId` / `BatchNo` / `BatchDate` / `ExpiryDate` / `SerialNo` —
the batch allocation. Single batch per row by design (user decision): the
lookup's godown/stock identify the row the quantity comes from.

### 7.2 The negative-stock gate — and the incoherence to fix, not port

`AllowNegative` is a per-item master flag. The gate compares BillQty against
StockQty at entry time and again at save time.

In Qt it is **hard-coded `"Y"` on the bill's own load path and on the order
import**, and left **empty** on the quotation import — so the gate is silently
*off* for every reopened and every order-imported line, and *on* against a
month-old stock snapshot for a quotation-imported one. Three behaviours for one
rule.

**Do not port any of the three.** The flag belongs to the item, and the stock
figure must be today's: re-resolve both from the item lookup when a line is
loaded or imported, and if the stock context cannot be established, say the gate
is unavailable rather than inventing an answer in either direction.

### 7.3 The order-quantity cap

Unlike the stock gate, this one **puts the value back**: negative stock is a
condition an operator may knowingly accept, over-billing an order line is not
(subject to `allowBillOverOrderQty`). On an imported line, `OrderQty` holds what
is still **pending**, not what was ordered, and the cell is read-only — a cap the
operator can raise by hand is no cap at all.

### 7.4 Size → quantity

`"L*W*T*Pcs"` → cubic feet → BillQty, at **4 decimals** rather than the item's
`DecimalCount`: timber items routinely carry a DecimalCount of 0, and rounding a
10.4375 cft board to 10 quietly gives the customer three quarters of a foot.

### 7.5 Promotions and price levels

The promotion binder is shared (`shared/promotions` + `sales_promotion_binder`)
and is already wired here. Price levels: `changePriceLevel` / `stepRowPriceLevel`
with keyboard shortcuts, and a per-row level that can differ from the document's.

---

## 8. Removing a line that came from an order — a three-way question

If the row carries a `SrcDocId` and the document's source is `SALES_ORDER`,
removing it is not one action but three, and the operator must choose:

- **Cancel on Order** — `PUT /sale-orders/cancel-lines` with a mandatory
  `soiCancelReason`; the row goes only *if the server agrees*. **One line — the
  selected row's — never the order.**
- **Remove from Bill** — comes off this bill, stays pending on the order.
- **Cancel (abort)** — the row was picked by mistake; nothing happens.

Port the prompt wording; this is one of the few places where an ERP's exact
phrasing is load-bearing. Note the ordering: the row is dropped in the success
callback, never optimistically — a refused cancellation that had already removed
the row would leave the order and the bill disagreeing.

---

## 9. Settlement

The dialog is `plan-react-sale-tender.md` in full. What this screen owns is the
contract around it:

- it hands the dialog the document total, the credit standing, the loyalty
  points and the allowed flag;
- it receives back `tender`, `credit`, `refund`, `surcharge`, the tender lines,
  and whether to print;
- `updateSettlementDisplay()` is the single place where tendered, adjusted,
  balance and refund are reconciled — one function, called from every path that
  can change any of them.

Two rules that belong here rather than in the dialog:

- **a CREDIT tender (type 9) posts no accounting leg at all** — the party debit
  simply stays open;
- **surcharge is a charge on the payment instrument**, not on the goods: it must
  not re-enter the pricing engine.

---

## 10. Adjustments — F4, and the rule that governs them

Advances and credit notes the customer already holds, set off against this bill.
**An adjustment is never a tender.** They are different rows in different
tables, they post differently, and a screen that merges them will produce a bill
that balances on screen and not in the ledgers.

- One panel component, two mount points: inside the tender dialog and outside it
  on the bill. Same state, one source of truth (`m_adjustmentsAuthoritative`
  marks which side last wrote).
- `advanceAdjusted()` / `noteAdjusted()` / `totalAdjusted()` feed
  `updateSettlementDisplay()`.
- Importing an order pre-fills the panel with the credits **that order** raised,
  read in the SOURCE document's accounting year — a March advance really does
  settle an April invoice, and every txn table is partitioned by `acc_year`.
- Changing the customer re-seeds the context (§4.3).

---

## 11. Loyalty — port what exists, and know what doesn't

What exists: `allow_loyalty` and `cust_points` come from the customer lookup,
the points are shown, `LoyaltyPv` is a per-line column, the document total of
points is computed, and the tender dialog can redeem them.

What does **not** exist: accrual. The payload carries `sbHasLoyalty` — a
boolean — and nothing else. The points history screen (Alt+H in 3.0) is not
built. Do not design a React accrual path on the assumption the server is
recording one; see `12/13/14` in the schema share for the model it will follow
(statistical-only points, FIFO expiry lots, cooling-period claw-back, and a gift
voucher that is **not** a tender).

---

## 12. Hold, autosave, crash recovery — the section that does not port

This is the one place where the Qt design cannot be transliterated.

**What Qt does:** writes JSON files under `AppLocalDataLocation/salebill/` —
`holds_<deviceId>.json` and `autosave_<deviceId>.json` — keyed by device id.
Autosave is debounced, requires at least one real item row, and is offered back
on next open.

**Why it cannot port as-is:** there is no such directory in a browser, and the
device id is not a file name. More importantly the quotation already does this
**the other way** — `TxnHoldController`, server-side, shared, and identical on
every screen that uses it. The bill is the outlier, and it is the screen where
losing a held sale matters most.

**Port to:**

- **Holds → the server**, via `TxnHoldController`'s endpoints. A hold taken at
  one counter must be resumable at another; a file on one machine cannot do
  that, and a counter that dies takes its held sales with it.
- **Autosave → IndexedDB**, per device, as crash recovery *only* — never as a
  second source of truth. Keep the guard that it needs a real item row, or
  every abandoned blank screen offers itself back on the next open.

Keep from the Qt version: the autosave snapshot carries the adjustments **with
their display fields**, which is what lets a recovered bill name the credits it
had set off even when the open-credits endpoint is unreachable at recovery time.

---

## 13. Importing a quotation (Ctrl+F3) or an order (Ctrl+F4)

Two entry points, one shape: pick → fetch by id **with the source's own
accounting year** → paint. Both carry the source trail (`sbSrcDoc*`, and
`SrcDocId` per line).

Rules that are not obvious:

- **Quoted figures are kept.** A quotation is a promise; the bill opens on the
  numbers the customer was quoted, and the operator reviews quantities anyway.
- **An order imports the PENDING quantity**, never the ordered one, and a line
  with nothing pending is skipped with a count shown. Billing the full quantity
  a second time is exactly the mistake this screen must not make.
- **The order's `Size` crosses only on a whole first delivery** (nothing
  delivered, nothing cancelled): `"12*6*2*10"` describes all ten boards, and on
  a part delivery it would print dimensions for ten against a quantity of six.
- **The order picker is the PENDING list**; a cancelled or fully delivered order
  is refused with a reason. **The quotation picker has no such filter and no
  status check — so the same quotation can be billed twice.** Fix this in the
  port; do not carry it over.
- Charges ride across with their ids cleared — the bill inserts its own charge
  rows, never the source's.
- The POS the source was priced under carries over, still overridable.

**Known losses, and they are schema gaps rather than client bugs** — do not
spend a sprint "fixing" them in React:
`sale_quotation_item` has **no** godown, salesman or stock column at all, and
`sale_order_item` has **no** batch columns. The fields cannot cross because they
do not exist on the source.

---

## 14. Validation (`validate.ts`, pure)

In order, and the first failure wins:

1. a customer;
2. at least one item row;
3. per row: quantity > 0; rate not below the minimum selling price; rate not
   above MRP unless `skipMrp`; the negative-stock gate (§7.2); the order cap
   (§7.3);
4. bill amount > 0;
5. term/credit coherence — a credit bill needs due days or a due date;
6. settlement: tendered + adjusted + credit ≥ the bill total, unless the term is
   credit;
7. the charge-role gate: a LOADING/FREIGHT charge row must exist when
   `chkHasLoad` / `chkHasFreight` is on. (Auto-apply usually satisfies this
   already — §6.)

Every message names the **row number**. Nothing here may be a server round trip.

---

## 15. Save payload

`POST /api/v1/bills/create`, one object: header + `items` + `charges` +
`tenders` + `adjustments`.

Points worth stating because they are easy to get wrong:

- **`sbBillSlno` is server-assigned** per counter and year and is **never
  sent**. The refno is the client's.
- `sbId` present = update.
- `sbAccYear`, `sbCompanyId`, `sbBranchId`, `sbSessionId`, `sbDeviceId`,
  `sbDeviceType` are all part of identity — every txn table is partitioned by
  accounting year and the counters are offline-first.
- `sbBillType` is CASH or CREDIT, from the term control; due days/date are sent
  only for CREDIT.
- Header discount totals (`sbItemDisc`, `sbSplDisc`, `sbSchDisc`,
  `sbBillSchDisc`, `sbAddlDisc1/2`) are **already netted inside**
  `sb_taxable_amt`. They are reported, not re-applied — see §17.
- `sbHasLoad` / `sbHasUnload` / `sbHasFreight` / `sbHasPromo` / `sbHasComm` /
  `sbHasLoyalty` are flags the server uses to decide what to look for.

---

## 16. Load, edit, cancel

Load: `GET /bills/get` by `TxnDocKey` (id + company + branch + **acc year**),
then `applyLoadedHeader` → `applyLoadedItems` → `applyLoadedTotals`, with the
recalculation suppressed throughout and run **once** at the end. In React:
build the whole draft, then one derive pass. Never paint row by row through the
pricing engine.

**Cancel is `POST /api/v1/bills/delete`** — it takes a `CancelBillDto` with a
reason and answers with the order lines it released. It is a *cancellation*, not
a row delete. **The Qt screen does not implement it at all**; the React screen
should, and it is the natural home for the reason prompt.

---

## 17. What the server does with the bill, and what the screen must not assume

The screen does not post to the ledgers, and must not imply the amounts it
shows are the legs. For reviewers reading this plan alongside the posting work:
every amount is stored **positive**, with direction in a `dr_cr` flag; the party
is debited the bill amount, Sales and the output taxes are credited; and the
**before-tax charge trap** means a before-tax charge posted to its own ledger
must be subtracted from the Sales credit or it is credited twice. None of this
is client arithmetic. It is here so that nobody "helpfully" sends signed amounts
from React.

---

## 18. Backend gaps this screen inherits

1. **No print pipeline.** F11 and save-and-print are stubs
   (`// TODO: no print pipeline exists in this codebase yet`). Schema designed
   — `17_printing.sql`, `plan-react-printing.md`. The dialog's "print" flag
   already flows; there is nothing at the end of it.
2. **No godown or batch picker.** The columns exist and the payload carries
   them; nothing fills them. Needs the stock endpoints.
3. **No e-invoice / IRN path on this screen**, though the GST provider layer is
   designed (`11_gst_provider.sql`).
4. **Loyalty accrual** (§11).
5. **No cancel in the Qt client** though the route exists (§16).
6. **Slab pricing**, TCS, and the companion screens are unbuilt.

---

## 19. Tests

- **Golden documents** against the shared engine — the same cart priced as a
  quotation, an order and a bill must agree to the paisa. This is the single
  most valuable test in the sales module.
- `importOrder`: a line with 10 ordered / 8 delivered imports 2, is capped at 2,
  and carries `Size` **only** when nothing is delivered or cancelled; a line
  with nothing pending is skipped and counted.
- `importQuotation`: an already-billed quotation is refused (the fix in §13).
- Adjustments: advance + note + tender + credit reconcile to the bill total;
  changing the customer clears them after a confirmation, and not before.
- Auto-apply: seeds on new and on clear, never onto a loaded document —
  including a loaded document with no charges at all.
- POS: an inter-state bill followed by a reset leaves the next bill's own-GST
  charge rows on **CGST/SGST**, not IGST.
- Persistence: a hold survives being resumed at another counter; an autosave
  with no item rows is not offered back.

---

## 20. Phases

1. Header + item grid + the shared engine, priced and totalled. No money.
2. Charges + auto-apply + place of supply. The document is now arithmetically
   complete.
3. Validation + save + load. Shippable for credit sales only.
4. Tender (its own plan) + `updateSettlementDisplay`.
5. Adjustments (§10).
6. Imports (§13), with the already-billed guard.
7. Hold/autosave on the server + IndexedDB (§12).
8. Cancel (§16). Print when there is a pipeline.

---

## 21. Do not port

- **The three `AllowNegative` behaviours** (§7.2) — port the rule, not the
  incoherence.
- **The quotation import's missing already-billed guard** (§13).
- **File-backed holds** (§12).
- **`m_loading` as a global recalculation suppressor.** It exists because the Qt
  screen paints cell by cell through the same handlers the operator triggers. A
  React load builds a draft and derives once; a flag that means "ignore my own
  events" is a symptom of that shape, not a feature of it.
- **Hold implemented twice.** One controller, both screens.
- **Alt+R** meaning `hikePrices` here while 3.0 uses it for split-stock. Decide
  once, across all three screens.
