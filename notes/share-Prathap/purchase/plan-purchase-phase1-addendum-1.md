# Purchase phase 1 — addendum 1 (mockup review)

Date: 2026-10-27. Changes `plan-purchase-phase1.md` rev 1. That file is not edited; read the two together.

## A1. Damaged qty on the Purchase Invoice (user: "damage col not found in purchase invoice")

The GRN had a Damaged column; the invoice did not. A **direct** bill (goods arrive with the bill, no GRN) needs it just as much.

**Change:**
- `pur_invoice_item` gets **`pii_damaged_qty`** beside billed / received / short.
- Rule on every invoice and GRN line: **billed = received + damaged + short**.

| Qty | Stock | Accounts / GST |
|---|---|---|
| Received + free | SALEABLE | Normal: Dr Stock-in-Hand, ITC taken |
| **Damaged** | **DAMAGED bucket** (in stock, not saleable) | Same as received: the goods arrived, so ITC is taken (sec 16(2)(b) is met). Valued at the bill cost. |
| Short | none | SHORTAGE_RECEIVABLE + INPUT_TAX_PENDING (§3.4) |

- **What happens to damaged stock next** is a Purchase Return row on that bill (Ctrl+F11 from the invoice): Return / Claim — goods sent / Claim — destroyed here / Replace (§3.5). The ITC is reversed there, not at receipt.
- **From a GRN:** the invoice shows the GRN's damaged qty, locked (as received is).
- **Allocation:** the F12 split carries the DAMAGED bucket rows (as the GRN popup does). Σ alloc = received + damaged + free.
- **Register entries touched:** ENT-34's QC hold bucket stays phase 2 (that is inspection after receipt; this is damage seen at receipt).
- **Mockup:** `purchase_invoice_ui_mockup.png` now has the Dmg column (line 3: 48 billed = 46 received + 2 damaged) and callout 4 rewritten.

## A2. Textile columns on the purchase line (user: "why we didn't add textile cols right now")

The rev-1 mockups showed only FMCG rows; textile appeared only inside the F12 popup and a callout. That was an omission. The user decided on 23-10 that **size / colour / model belong on the purchase LINE** (`textile/plan-textile-auto-batch-2.md`), and Opening Stock already shows them (OpeningStockCols 39–44, notes-149).

**Change (GRN, Purchase Invoice, PO, Return):**
- Line columns **Model / design · Size · Colour · Batch**, the same rule as Opening Stock (`takesBatchDetail`): shown on a line when `stock.auto_batch_no` is on **and** the item is tracked by batch. Other items on the same bill: blank and locked. Hidden for a company with no such items.
- **Batch:**
  - blank while typing; **given by the server on post** (CBE26-00041 …);
  - size / colour / model are written to `stock_batch_detail` against it;
  - picking an EXISTING batch tops it up;
  - a typed new batch with a size is refused (as notes-149).
- **One line = one design × size × colour = one batch = one label per received piece.** Damaged pieces get no label.
- **Ratio pack (Ctrl+P):** design + colours + size ratio × packs → one line per size × colour. The lines stay editable.
- **Value-based GST slab** per piece (P15) shown in the Check column.
- **Selling price + MRP on the batch** (preset TEXTILE_BATCH); a new price = a new batch.
- **PO** (textile): Model / Size / Colour optional on the PO line — an order may be "Cotton Shirt CS-Slim 402, 36 pcs, sizes by ratio"; the ratio pack runs at GRN. **Return:** the row carries the exact batch, so size / colour / model show read-only.
- **Mockup:** `purchase_textile_lines_ui_mockup.png` (the lines grid of a textile company + ratio pack + label).
- **Register:** ENT-13 stays phase 1 and now has its mockup.

## A3. Missing fields (user: "check is there any missing fields") + serial no + batch on the row

**Batch / serial entry (user: "in some cases user may want to enter in item line… optional in row / batch popup"):**
- **On the row:** Batch / Serial, Expiry (and Mfg under More columns), MRP, Godown. This is the one-allocation shortcut, for the common single-batch line.
- **In F12:** several batches, godowns or serials. The row then shows "2 batches (F12)" / "5 serials (F12)", read-only.
- **Both write the same `pur_item_alloc` rows.**
- **Required or not = the item's track preset** (batch / expiry / serial). The screen does not decide. Untracked items: cells blank and locked.

**Serial numbers — ENT-14 moved from phase 2 to phase 1:**
- qty 1: typed / scanned on the row;
- qty > 1: F12 scan mode, one row per piece (qty 1, its own lot — serial is part of the lot identity);
- count must equal received (REFUSE at post);
- duplicates refused within the bill and against stock on hand;
- optional warranty-to date.

**Purchase Invoice — header:**
- **Entry date** (our posting date: period + ITC month), separate from the supplier's invoice date;
- **Goods received on** (from the GRN, typed on a direct bill: ITC + MSME due date run from it);
- **Bill from** = the supplier's GSTIN / branch (several per supplier; another state → IGST);
- **E-way bill no / date** and **IRN / Ack** (recorded; QR fill + IRN check phase 2).

**Purchase Invoice — buttons:** Attach (Ctrl+A), Hold (F9).

**Purchase Invoice — totals:** Supplier TCS line; Cash mode tender strip (Cash / UPI / Bank, settled at once).

**Purchase Invoice — lines:**
- Barcode (supplier's code also finds the item), Batch / Serial, Expiry;
- ITC column (Elig. / Inelig. / Blocked / RCM / Pend.);
- More columns via the column chooser (hidden by default, saved per user): Gross, Disc ₹ (3.0 discount by %, per-qty or amount), Cess %, Cess / unit, Mfg, Godown, Weight, Sell B / C / D, Old prices, PO / GRN ref, Remarks.

**GRN:**
- vehicle + driver, e-way bill no, packages received vs sent (per-bale status = Gate Inward, phase 2), received by, remarks;
- lines: barcode, Batch / Serial + Expiry on the row, damage reason, serials via F12.

**PO:**
- credit days (supplier default);
- terms & conditions (small text master, printed);
- **approval by amount** (reuses the approval rules; setting `purchase.po_approval_above`, 0 = off; an unapproved PO cannot feed a GRN);
- line MRP + tax; ordered value incl. tax.

**Purchase Return:**
- place of supply (as the bill);
- supplier CN no / date if handed over at once;
- e-way bill (needed ≥ ₹50,000, recorded; generation phase 2);
- vehicle / driver.

**Claims:**
- **Claim against** = supplier / transporter / insurer: the receivable sits on that party. Transit breakage is the transporter's.
- **Received as** = supplier CN / cash / bank / adjust in next bill.

**Shortage Receipt:**
- header: receipt no, received date-time, godown, challan no, received by, loadmen, remarks;
- lines: Batch / Serial + Expiry.

**New mockup `purchase_masters_settings_ui_mockup.png`:**
- **Supplier**: MSME category / activity / written agreement / agreed days, drug licence expiry, FSSAI no + validity, default godown, show-notes-on-entry, import flag, bill-from GSTINs. Drug licence no and notes already exist on the NexERP supplier form.
- **Item card**: purchase unit, min shelf life %, ITC class, shortage %, default markup %.
- **14 purchase settings** (per company, snapshotted on the document like the sales policy).

**Register updates:**
- ENT-14 → phase 1;
- MST-01 / MST-03 detailed as above;
- new IDs: ENT-35 entry date + goods-received date, ENT-36 bill-from GSTIN, ENT-37 e-way / IRN recorded, ENT-38 column chooser, DOC-01a PO approval by amount, DOC-07a claim against / received as.
