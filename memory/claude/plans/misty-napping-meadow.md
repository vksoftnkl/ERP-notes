# Sale Bill — Qt (NexERP) vs React (`features/sales/salebill`): functional differences

## Context
You asked for a full functional comparison of the Qt screen
`/home/vk/VKNexERP/NexERP/src/modules/sales/salebill` (≈14.7k lines) with the
React port `features/sales/salebill` (≈24.5k lines). Three passes read the Qt
code line by line (entry 1–2400, 2370–5100 plus re-tender, 5100–7805 plus the
whole tender dialog) and traced each behaviour into React.

**Where React stands against the port plan** (`docs/plan-react-sale-bill.md`
REV 2): phases 1–4 are committed. Phase 5 (imports, carry, shipping, GST) and
phase 6 (re-tender, lists) are not built in `salebill`. Most of the "missing"
list below is those two phases.

**Cite legend**
- Qt: `E` = `sale_bill_entry.cpp`, `T` = `sale_bill_tender.cpp`, `R` = `sale_bill_retender.cpp`.
- React, relative to `features/sales/salebill/`:
  - `hook` = `use-sale-bill-draft.ts`
  - `view` = `components/sale-bill-entry-view.tsx`
  - `pay` = `salebill.payload.ts`
  - `val` = `salebill.validate.ts`
  - `slice` = `store/slices/saleBillSlice.ts`
  - `pricing/*` = `domain/pricing/*`

---

## 1. Bugs in React

### 1a. I checked these five in the code myself
1. **Tender rows add the change into "received" a second time.** `pay:329,364-369`
   computes total = base + surcharge, with the over-tender still inside, and then
   received = total + change.
   - A 470 bill paid with a 500 note sends amount 500 / total 500 / received 530 / change 30.
   - Qt and the plan (§15.10) send amount 470 / total 470 / received 500 / change 30 (`E:1087-1090`).
2. **CGST and SGST are rounded separately, which can leave the bill ₹0.01 short.**
   React rounds each half from the taxable amount (`pricing/lines.ts:123-124`,
   `pricing/document.ts:58-61`). Qt rounds the total GST once and makes
   SGST = GST − CGST (`E:1841-1844`). On an inclusive ₹4,000 at 18% React totals
   3999.99, which the server refused as SALES_AMOUNT_MISMATCH (`E:1826-1830`).
3. **Function keys with Ctrl or Shift run the plain key's action** (`view:1358-1430`, no modifier check):

   | Keys | Runs in React | Should do (Qt) |
   |---|---|---|
   | Ctrl+F5 | Save | Challan import |
   | Ctrl+F6 | Save/Post & Print | Re-tender |
   | Shift+F2 | Edit | Open the e-Invoice register |
   | Shift+F3 | Delete | Open the e-Way register |

4. **An opened bill shows freshly recomputed totals instead of its saved ones.**
   `storedPricing` is never filled (`pay:963` points to a `storedPricingOf` that
   doesn't exist), so `hook:825-828` always falls back to a live recompute. The
   title bar still says it is showing the saved figures. Qt paints the stored
   totals (`E:4661-4815`).
5. **The party-context reply can be thrown away.** It is requested before
   customer-detail sets `custId` (`hook:1266-1286`). If it answers first,
   `slice:430` drops it and nothing asks again: the credit panel stays blank
   and the credit popup never appears.

### 1b. Reported by the comparison passes, with code cites, not yet re-checked by me
6. **A new bill ignores the company's pricing policy.** `createBillDraft` gets
   no policy (`salebill.state.ts:229`), so every new bill prices as
   manual / manual / discount-first off / round-off step 1. The settings keys
   exist in `salebill.constants.ts:253-257` but nothing reads them.
7. **Ctrl+± adds or deletes a row instead of stepping the line's price level.**
   The bill never passes `onStepPriceLevel` to the grid (`view:1671-1698`), and
   a plain line is removed without asking. Qt asks: "Remove "%1" from this bill?"
   (`E:2293-2299`).
8. **Importing onto a loaded or amending bill keeps that bill's identity**
   (id, status, rights, people, term), because `salebill.import.ts:148-225`
   copies the current draft. The next save overwrites or amends that bill with
   the imported lines. Qt clears the screen first (`E:5157, 5532`).
9. **A quotation import keeps the previous customer's credit and loyalty
   figures.** React doesn't re-read customer-detail or party-context afterwards
   (`hook:2388-2420`); Qt does (`E:5334-5337`).
10. **Adjusting after settling creates change nobody handed back.** Example:
    settle 1000 in cash, then adjust 300 with F4. React saves with
    `tdChangeAmt` 300 (`val:161-184`). Qt refuses: "Re-open settlement (F5)"
    (`E:6708-6723`).
11. **When a credit tender switches the term to Credit, promotions run again**
    (`hook:2685` together with `hook:1587-1589`). The tender then no longer
    matches the bill. Qt deliberately does not re-run them (`E:265-282`).
12. **F9 Hold works on a read-only or already-saved bill** (`hook:2478-2501`).
    Qt refuses both (`E:4966-4978`).
13. **F7 Clear and closing the screen leave the autosave behind**, so a bill you
    cleared is offered for recovery next time (`hook:1398`). Qt deletes it on
    clear and close.
14. **Some keyboard shortcuts skip the checks their buttons make:**
    - Alt+D, Ctrl+F3 and Ctrl+F4 work on a read-only bill.
    - Alt+Y copies without the create right and without the confirm or the
      "nothing to copy" warning (`view:1432`).
    - Ctrl+Enter skips the `rights.post` check (`hook:2000-2079`).
15. **The header price-level combo only changes the header.** There is no
    "Apply to Selected / All" prompt, no re-price and no edit-price check
    (`bill-header-blocks.tsx:349-357`). The waiting blank row also keeps the old
    level (`slice:1026`).
16. **Freight and loading charges don't follow their ticks.** With Freight or
    Load unticked, the charge still applies; Qt forces it to 0 (`E:1955-1966`
    vs `pricing/document.ts:146-151`). Unload is never set to the loading
    amount (`E:1957`).
17. **When the server leaves out a right, React allows it; Qt refuses it**
    (`salebill.verbs.ts:74,110,129` vs `E:7552,7566,7621`).
18. **Fields not filled from the item price lookup:**
    - HSN (`E:2443-2444`).
    - `stock_id`, batch number and batch dates (`E:2467-2470`).
    - A level of 0 or null from the server drops the line to level 1
      (`quotation.state.ts:370`); Qt keeps the level it asked for.
19. **Pressing Enter on a barcode cell that already has an item replaces it**
    with a new scan lookup (`grid:443-449`). Qt only looks up a scan on an
    empty row.
20. **A sales order's pending quantity has no fallback.** React reads
    `soiPendingQty` only (`salebill.import.ts:435,533`). Qt falls back to
    ordered − delivered − cancelled (`E:5615-5620`).
21. **Holds:**
    - Deleting a held bill doesn't ask first.
    - The empty list says "No quotation is on hold."
    - There's no "Suspend this bill?" confirm.
22. **A failed auto-post leaves the draft behind.** When an auto-post of a
    loaded draft fails, Qt deletes the draft (`E:7008→7409`); React keeps it.
    Also, `postNow` reads `draftFromAutoPost` right after a dispatch
    (`hook:1830,1878`); that needs a test.

---

## 2. Features missing from React (Qt has them)

### 2a. Whole areas (plan phases 5–6)
- **Transport band:**
  - There's no band UI and no vehicle picker.
  - PUT `/bills/transport` exists in the API but is never called.
  - There are no posted-bill locks for the band.
  - A `SALES_EWAY_TRANSPORT_MISSING` refusal doesn't open the band.
- **Shipping:** the shipping dialog, the ship-to address manager, the live
  summary line and use of the parsed `shipTo[]` are all missing (`E:3494-3575`).
- **Challan and open-source import (Ctrl+F5):**
  - The import itself: per-document, all or nothing.
  - Topping up a line from the same source, capped at the order quantity.
  - The challan line fill.
  - Appending several orders line by line (`sales.multi_order_bill` is read
    but never used; an order import always replaces the whole bill).
- **Re-tender (Ctrl+F6, the `SaleBillRetender` dialog):** none of it exists:
  - reading `/bills/tender-context`;
  - void ticks with reasons, and "voided = new" before OK is allowed;
  - the remark;
  - POST `/bills/retender` with `tdRowNo` numbered after every existing row.
- **GST actions:**
  - e-Invoice, e-Way and Part-B (the endpoints exist; the buttons are always
    disabled).
  - The changing labels (Retry / Fetch e-Inv) and the detail shown on the badges.
- **Freight bands:** they are fetched and stored but never applied to the lines
  (`slice:505`; Qt `E:3903-3927`).
- **Charges carried from the source:** quotation and order charges aren't
  carried, and the `/validate` `proposals.charges` reply isn't applied
  (`E:5323, 5638, 7227`).
- **Stock refresh after an import** (`refreshImportedStock`, `E:5347-5440`):
  today's stock, batch, loyalty PV, reorder level and the negative-stock flag
  on imported lines.
- **F11 print last bill**, and remembering which bill was last (`E:5968`).

### 2b. Smaller items
- **Keys:**
  - Plain `+` / `-` to insert or remove a row.
  - Ctrl+± to step a line's price level.
  - After a quantity edit, the cursor jumps past a free line (`E:1543`).
- **Price levels:**
  - The starting level from `sales.default_price_level`.
  - The company's own level names in the grid (React shows a fixed A/B/C/D…).
- **Missing confirms:**
  - "Apply/Remove %1 on this bill?" on the Freight / Load / Unload / Promo ticks.
  - Removing a plain line.
- **Grid:** the Σ totals footer row under the item grid (`E:1899-1932`).
- **Validation:** a quick delivery must have a remark (`E:6674-6679`). React
  has no quick-delivery field at all.
- **Adjustments:** the notice "Some credits changed since this bill was put
  aside". The helper exists (`salebill.adjust.ts:132-165`) but nothing calls it.
- **Loyalty in the tender dialog:**
  - Typing points to get the amount.
  - The Value/Point, Min points and Points left fields (`T:1167-1173, 1241`).
- **Temp credit:** the hint line "max N days · cap · name + mobile mandatory"
  (`T:1393`).
- **Order import doesn't bring across:**
  - A credit order switching the bill to Credit.
  - The salesman, agent and packed-by.
  - The customer PIN.
  - The message "This order holds an advance of %1…".
  - A read-only Order Qty cell.
- **Quotation import doesn't bring across:**
  - The salesman and agent.
  - Item facts (group, brand, …).
  - Read-only Item and Description cells.
  - The "That quotation could not be found." message.
- **Bill-to:** the PIN field, and the walk-in-only "Temp Name" row.
- **Bill fields not sent:**
  - `sbBillRefno`, `sbUsrRefdate`.
  - Agent commission % and amount (always null; not even read on load).
  - Line-level company / branch / year, `sbiLotId` and created/modified by.
- **Quick-add customer:** Qt's small customer + ledger dialog. React opens the
  full customer master with Alt+C.
- **Autosave recovery:** isn't gated by `auto_save_temp_bill`.

---

## 3. Same feature, different behaviour (not covered above)

### Keys and close
- **Ctrl+1..N:** React has a fixed 4 levels mapped to level numbers; Qt uses
  min(the level-count setting, 4) and positions in the combo.
- **Close / Clear / F8 / import prompts:**
  - React asks only when the bill has unsaved changes, with one generic text.
  - Qt asks whenever the grid has lines, with its own wording for each case
    (amend, clear, open-another, import).

### Term and credit
- **Credit with 0 days:** Qt sets the due date to the bill date; React sends null.
- **Due date:** editable in React and works the days back out; in Qt only the
  days are entered.
- **Credit-limit confirm on save:**
  - React counts amount or bill-count exceeded only.
  - Qt also counts overdue days, a used-up limit and a fallback when the
    context is missing, and skips the question when `allowCreditDue` is on.
- **While amending:** React turns the tender route off. Qt keeps Tender, so the
  payment can change during an amend.

### Party panel
- **Address source:** React shows `cust_address` / `cust_place`; Qt shows
  `cust_eadd1` / `cust_eadd2`.
- **GSTIN:** React falls back to the e-commerce GSTIN; Qt keeps it separate.
- **Credit panel:** Qt keeps Advances, Credit notes, Cash today and Temp credit
  as four separate cells with counts and colours, and its "available" figure
  can go negative. React merges them into two cells and never shows a negative
  available.
- **Facts line:** React always shows it, even with no customer.
- **Credit popup:** Qt shows a modal box; React shows a toast.
- **Freight bands:** Qt re-reads them when the area or the distance changes;
  React only on a distance change.

### Grid and pricing
- **Minimum rate:**
  - Qt warns even on a typed 0, clears the cell and moves focus back.
  - React skips 0, doesn't move focus, and also warns above MRP.
- **Negative stock:** Qt warns whenever qty > stock and moves focus back.
  React warns only after a live lookup.
- **Order-quantity cap:** React applies it only when `orderQtyLocked` is set.
- **RateDiff:** React gives the whole rate when the line has no list price;
  Qt gives 0.
- **Other discount inputs:** React clears them only when the new value isn't 0;
  Qt always clears them.
- **Free line:** a rate typed into a free line is kept in React; Qt forces it
  to 0 on every recalc.
- **Size entry:**
  - Qt requires exactly 4 parts, divides cft by 144 to 4 decimals, rewrites the
    text in standard form, and clears and warns when it's invalid.
  - React accepts any number of factors, rounds to 3 decimals, and ignores
    invalid text silently.
- **Per-line freight:**
  - Qt uses bill qty × rate, on every line.
  - React uses net qty × rate, only for items marked for freight.
- **Charge GST:**
  - Qt works at 4 dp and splits it exactly in half.
  - React rounds each half to paise.
  - A role override reaches only one row in React; every row with that role
    in Qt.
- **Bill totals:**
  - **Margin:** Qt = profit ÷ gross; React = profit before tax ÷ taxable.
  - **Scheme box:** Qt shows scheme + bill-scheme; React shows scheme only.
  - **Item count:** Qt counts every row; React counts rows with an item.
- **Promotions:**
  - React re-runs only when the number of schemes changes, not when their
    rules change.
  - React shows a single hint with ⓘ whenever promotions are on.

### Scans and units
- **Barcode cell after a scan:** React writes the item's master barcode; Qt
  keeps the scanned label.
- **Unit switch:** React prices with `line.itemId`; Qt uses the `item_id` the
  server returns.
- **Auto-pop qty:** React applies it only on pick or scan.

### Imports
- **Order picker:** React shows every order and refuses after the pick; Qt
  shows only pending orders.
- **Order lines:**
  - **Quantity:** React divides by length.
  - **Weight:** React works per unit.
  - **`sbiSrcItemQty`:** React sends the ordered qty; Qt sends the pending qty.
- **Advance:** React pre-fills credits at their full value and lets the clamp
  trim them. Qt sets `soAdvanceBalanceAmt`.

### Save payload values
- **Header charge totals:** `sbFreightAmt` / `sbLoadAmt` / `sbUnloadAmt` /
  `sbCashDisc` / `sbOtherAmt1` come from the charge rows in React; Qt sums the
  item columns, with unload = load, cash discount 0 and other 0.
- **Margins:** `sbMarginAmt` and `sbMarginAmtWot` are swapped.
- **Counts:** `sbTotBags` and `sbTotItems` are worked out differently.
- **`sbBillDatetime`:** React uses when the draft was created; Qt uses the
  time of the save.
- **`sbBalanceAmt`:** React clamps it to ≥ 0.
- **Tender and adjustment arrays:** React omits `tenders` when there are none
  and sends `adjustments` once the panel has been touched; Qt always sends
  `tenders` and sends `adjustments` only when authoritative.
- **Tender fields:** `tdConversionRate` and `tdUnitsUsed` go on loyalty rows
  only; `tdCardLast4` on card rows only.
- **Adjustment remarks:** React sends the credit's narration; Qt the
  adjustment's own remark.

### Save checks
- **Customer:** React accepts a typed name without a customer id; Qt requires
  a picked customer.
- **Accounting year:** React checks by the year label; Qt by the session's
  begin and end dates.
- **Negative qty or rate:** refused by React, accepted by Qt.
- **Above MRP:** React skips free lines.
- **Stock:** React asks an "unavailable" question even for service lines.
- **Draft save when `/validate` is unreachable:** Qt stops; React saves anyway.

### Lifecycle
- **After cancel:** Qt starts a new bill; React reloads the cancelled bill.
- **Amend order:** Qt goes check → remark → `/amend`; React goes remark →
  check → `/validate` → confirm → `/amend`.
- **Edit while an IRN or e-way bill is live:** Qt shows the button greyed with
  a reason; React hides it.
- **F8 list:** Qt offers View or Alter; React opens every bill in entry mode, so
  a posted bill goes straight to the amend confirm.

### Tender dialog
- **Change routing:** React needs the change tender to cover the whole change.
- **F1:** does nothing on the ADJUST or LOYALTY rows.
- **Amount cell:** React accepts any value and takes its absolute value.
- **Bank dropdown and labels:** React also shows the bank dropdown on UPI and
  BANK rows, and uses different reference labels.
- **Panel amounts:** update on blur in React, live in Qt.
- **Loyalty:** React always rounds down to a whole multiple.
- **Temp credit:**
  - **Notes:** 250 characters in React, 200 in Qt.
  - **Default days:** 30 in React; Qt reads the setting.
  - **Cap:** React compares the base amount; Qt compares the gross.
- **Panel check on Save:** missing in React; Qt runs it when the dialog saves.

### Hold and autosave
- **Debounce and trigger:** 1500 ms on any change in React; 1000 ms on a
  recompute in Qt.
- **What's worth saving:** React needs an item with qty > 0; Qt saves whenever
  the grid has lines.
- **When recovery is offered:** React only on a fresh bill; Qt on every open.
- **Wording:** the recovery prompt reads differently.

---

## 4. Deliberate differences: leave these alone
These follow plan §28/§32 or your own decisions:
- **Tender without a check first:** the tender opens with no client check and
  no `/validate`. That was your decision on 2026-10-06; every check runs when
  the dialog saves.
- **Holds and autosave:** holds live on the server (`/txn-holds`, with a
  device lease) and autosave in IndexedDB. Qt uses local JSON files (Q2).
- **No "Put on credit anyway"** (D2/Q1). React refuses at the credit row instead.
- **Change goes only to a tender that can cover it** (D3).
- **Tender payload fields:**
  - `tdIsPdc` is keyed on the type id (D9).
  - Loyalty uses the scheme rate (D4).
  - TEMP_CR skips the reference gate (D1).
  - TEMP_CR counts as credit in the roll-ups (D5).
  - `tdPartyLedgerId` is the customer id (house rule 1).
- **Temp credit mobile match:** the same mobile is matched on exact digits,
  leaving out this bill (D8).
- **Status and bucket:** `sbStatus` is never sent, and `sbiBucket` is always
  SALEABLE.
- **Allow-negative:** taken from the item, not forced to Y (Q4).
- **Tender list date:** offerable tenders are judged on the bill date.
- **Delete vs cancel:** delete is for drafts only, and cancel goes through
  `/bills/cancel`.

## 5. React-only additions (not in Qt)
- **Keys:**
  - F1 walks the panels.
  - Alt+R copies a row.
  - F3 / double-click opens the multi-size dialog, which also opens by itself
    after a pick.
  - Enter walks through the header.
  - The browser's close-tab warning.
- **Header and lines:** an editable due date, the above-MRP warning, and a
  per-line unit dropdown.
- **Refusals:**
  - Scanning an item with `allowSales=false`.
  - Importing a quotation that's already converted, cancelled or expired.
  - Posting from a browser with no registered device.
  - Missing unit or godown at save.
  - IFSC / MICR in the wrong format.
  - Its own message for `SALES_REVISION_STALE`.
- **Lists and printing:** printing selected bills from the F8 list.
- **Credits:** open credits are re-read every time a panel opens.

---

## Next step, if you approve
I'll fix the five verified bugs in §1a, then re-check items 6–22 in §1b, each
with a vitest case:
- **§1a #1:** `pay` sets received = base + surcharge and total = received − change.
- **§1a #2:** round the total local GST once and make SGST = GST − CGST, in
  `domain/pricing/lines.ts` and `domain/pricing/document.ts`. This is shared
  with quotation and order, so their tests need to pass too.
- **§1a #3:** check modifiers in the F5 / F6 / F2 / F3 cases of the view.
- **§1a #4:** build `storedPricing` from `/bills/get` in `parseLoadedBill`.
- **§1a #5:** set `custId` before the reads start, or re-read party-context
  after customer-detail lands.

The missing features in §2 would be ported as plan phases 5–6 only once you
say go.

**Verification:** `npx vitest run features/sales/salebill domain/pricing features/sales/quotation features/sales/sale-order`, `npx tsc --noEmit`,
then a browser check on the bill:
- the ₹4,000 inclusive 18% line totals 4000.00;
- a 500 note on a 470 bill sends received 500 / change 30;
- an opened bill shows its stored totals.
