# Audit: Purchase Return (menu 25) vs purchase_return_ui_mockup.png

Audited 2026-10-29 by qt-tester. Screen: `src/modules/purchase/return/purchase_return_entry.{h,cpp}` + `purchase_return_cols.h`, grid ui_table 59.
Mockup: `/home/vk/Shared/Prathap/purchase/purchase_return_ui_mockup.png` (1560 wide). Snapshots rendered at 1560 x 1000, theme v41, live :3011 as VKPOS, ZT data (cancelled afterwards).

Legend. Layer: **Q** = Qt only, **S** = needs a server route / field / DTO change, **D** = needs a DB change (seed or column). Evidence short names: RE = `src/modules/purchase/return/purchase_return_entry.cpp`, RC = `purchase_return_cols.h`, snapshots are in `/home/vk/Shared/Prathap/purchase/audit/`.

## Summary

Rows audited: 62 (61 judged, 1 not rendered).

| Verdict | Count |
|---|---|
| YES | 18 |
| PARTLY | 15 |
| DIFFERENT | 9 |
| NO | 19 |

The screen is function-first: the money/stock behaviour (4 outcomes per row, exact lot, split with +, claim party, post, cancel) is built and tested. The LOOK is not the mockup: no section headings, no outcome legend table, no 6-figure totals strip, no Reason / Billed / Returned / Credit to / Writes columns, 3 of 5 header rows missing fields, the button row lacks From bill / Batches / Show List.

### The 10 biggest gaps

1. **No "THE FOUR OUTCOMES" legend table** (Outcome / Goods / Money / ITC / Closes when). Whole panel missing. Qt only.
2. **Totals strip is one line of text, not the 6-cell strip** (Rows, Return now, Claims raised, Replace qty due, ITC reversed, Supplier balance now). ITC reversed and Supplier balance now are not shown before save at all. Qt (+ server: `prItcReversed` counts destroyed goods only, the mockup's 107.00 is every outcome).
3. **Reason column is missing** (the per-row Stock Reason from menu 293). The DTO takes `reasonId` and the DB has `pri_reason_id`, but Qt never sends it and the DB has **no purchase-return reason set** (`stock.stock_reason_master` has no row allowing a purchase return). Needs D (seed the set: expiry, breakage, damage, wrong item, excess, quality, rate) + Q.
4. **Credit to / Writes columns missing** (the per-row consequence: "Supplier now / Claim receivable / — (shortage row)" and "Dr Sup / Cr Stk ...", "ITC reversed", "Replace due 5"). Derivable client-side from Outcome. Qt only.
5. **Billed and Returned columns missing**; only a net "Can Return". Source lines carry `returnableQty` but billed / returned are not shown. Qt (check `source-lines` returns billedQty / returnedQty; if not, S).
6. **Header fields missing: Godown, Place of sup, E-way bill (with the 50,000 rule), Our ref, Transport.** Of 14 mockup header fields, 8 are present. E-way bill and Driver are in the DTO (`ewbNo`, `driverName`) so Qt can add them; Godown is derived by the server from the lot (no DTO field); Transport name / LR has no column on `pur_return` (the shared `txn_transport_detail` table exists but this module does not write it); "Our ref" is a free-text claim memo in the mockup but the DB column `pr_our_ref` is the source bill's refno.
7. **Button row**: From bill · Ctrl+L, Batches · F12, Show List · Ctrl+O missing; "Print debit note / claim memo" is a plain Print. An extra "Claims..." button stands in for the whole Claims screen. Qt only.
8. **In-cell treatment missing**: Bucket in amber when DAMAGED, Outcome / Reason as boxed (red-bordered when not Return) combos with full wording ("Claim — goods sent"), "Credit to" in amber / blue, grey "Writes" text. The built cells are plain upper-case codes ("CLAIM SENT"). Qt only.
9. **No section headings** (THE HEADER / THE ROWS with its sub-caption / THE FOUR OUTCOMES), no big title, no company / branch sub-title, status is plain text, not a coloured chip. Qt only (theme v41 would need the heading / chip classes).
10. **Blank prompt row** ("pick rows from the bill (Ctrl+L) - or start from the bill: Ctrl+F11 all / Ctrl+F12 selected") and the Ctrl+F12 "selected rows" route missing. Ctrl+F11 (all rows from the bill) exists on the invoice screen only.

## Table

| # | Mockup element / callout | Built? | What is there now | What is missing | Layer | Evidence |
|---|---|---|---|---|---|---|
| **Title / status** | | | | | | |
| 1 | Page title "Purchase Return - PRN/2026-2027/D1/00003" (large, bold) | DIFFERENT | Small plain label "Purchase Return - new / - PRN/..." top left | Large heading style | Q | return_01_new_empty.png; RE:64 |
| 2 | Sub-title "Acme Foods - Coimbatore" | NO | Not shown | company / branch line | Q | return_01_new_empty.png |
| 3 | Status chip (amber pill "DRAFT") | DIFFERENT | Plain text "DRAFT / POSTED" top right | Pill chip, status colours | Q | return_01..07; RE:65 |
| 4 | Date format dd/MM/yyyy | DIFFERENT | Date field shows `29-10-2026` (dd-MM-yyyy) | slash format as the date rule | Q | return_01_new_empty.png (Date *) |
| **Header** | | | | | | |
| 5 | Section heading "THE HEADER" | NO | Fields sit in an unlabelled card | heading | Q | return_01_new_empty.png |
| 6 | Row 1: Supplier * (combo) | YES | Supplier * dropdown 35 | - | - | return_02; RE:115 |
| 7 | Row 1: Against (Invoice / GRN) | PARTLY | "Against *" combo: "Bill" / "GRN (not yet billed)" | value text "Invoice"; callout 1 wording | Q | RE:116 |
| 8 | Row 1: Bill * (combo "PUR/.../00012 - SMA/1452") | YES | "Bill / GRN *" combo "refno - supplierRef - date - amount" | label is "Bill / GRN" | - | return_02; RE:117 |
| 9 | Row 1: Return No (disabled, grey) | PARTLY | Read-only, "given on save" placeholder; looks enabled (white) on a loaded doc | greyed disabled look | Q | return_07_reopened_posted.png |
| 10 | Row 2: Date * | YES | NexDateEdit, defaults to app date | slash format (row 4) | Q | RE:119 |
| 11 | Row 2: Godown (combo "Main Warehouse") | NO | No field. Server derives `prGodownId` from the lot | field + DTO `godownId` | S | `ReturnHeaderDto` has no godownId (swagger); RE:115-130 |
| 12 | Row 2: Settlement (combo "Adjust in next bill / payment") | PARTLY | "Set off against the supplier's bills" / "Keep as an advance with the supplier"; seeded from `purchase.return_settlement_default` | plan lists a third choice, cash refund (code comment says cash goes through a Receipt - needs your decision); wording | Q | RE:126-130 |
| 13 | Row 2: Our ref (text, "Claim memo 17") | NO | No field. `pr_our_ref` holds the source bill refno and is filled by the server | free-text memo ref (new column or reuse remarks) | S+D | `ReturnHeaderDto`; `pur_return.pr_our_ref`; purchase-return.ts:354 |
| 14 | Row 3: Place of sup (disabled "Tamil Nadu (33) - local - as the bill") | NO | No field. `pr_pos_stcd` is in the DB and `/get` | read-only field filled from /get + bill | Q (+S if `/source-lines` does not return the bill's state) | `pur_return.pr_pos_stcd` |
| 15 | Row 3: Supplier CN ("SMA/CN/97 - 27/10 (if given now)") | YES | "Supplier CN" text + separate "CN Date" date | mockup has one field; ours two | - | return_01; RE:120-121 |
| 16 | Row 3: E-way bill (disabled, "needed >= 50,000 - not needed (1,284.86)") | NO | Not shown. DTO `ewbNo` exists, never sent | field + the 50,000 rule note computed from the total | Q | `ReturnHeaderDto.ewbNo`; callout 7 |
| 17 | Row 3: Vehicle ("TN38AB1234 - driver Mani") | PARTLY | Vehicle text only | driver (DTO `driverName` exists) | Q | RE:122 |
| 18 | Row 4: Transport ("Back with KPN Parcel - LR 90233", span 2) | NO | No field | transporter + LR on the return (no column on `pur_return`) | S+D | `pur_return` columns; shared `txn_transport_detail` exists |
| 19 | Row 4: Remarks (span 2) | YES | Remarks text (row 3 of our card) | - | - | RE:124 |
| 20 | Extra (not in mockup): "Claim on (other)" dropdown of ledgers | DIFFERENT | Header-level transporter / insurer ledger for TRANSPORTER / INSURER claim rows | mockup shows no such field; claim party per row is not modelled in the mockup either | Q (decision) | return_04 |
| **Rows grid** | | | | | | |
| 21 | Heading "THE ROWS" + caption "from the bill - the exact batch it came in as - qty <= billed - already returned" | NO | A one-line hint under the grid | heading + caption | Q | return_03 |
| 22 | Col #: row number | YES | Row gutter | - | - | return_04 |
| 23 | Col Item | YES | Item name | - | - | return_04 |
| 24 | Col "Batch - Exp" (one cell "SX1025 - -", "MG1026 - 15/02") | PARTLY | Batch only; Unit column sits before it | expiry in the cell; `/get` returns `expiryDate` per line | Q | return_04, return_07; `/returns/get` line field `expiryDate` (purchase-return.ts:494-515) |
| 25 | Col Bucket (amber when DAMAGED) | PARTLY | Column titled "From", plain text, editable by popup | title "Bucket", amber colour | Q | return_04 |
| 26 | Col Billed | NO | Not shown | billed qty | Q (S if `/source-lines` omits it) | RC:15 enum has no Billed |
| 27 | Col Returned (already returned) | NO | Not shown | returned qty | Q (S if omitted) | RC:15 |
| 28 | Col Qty (white input box) | YES | Qty editable (capped only by the server: RETURN_QTY_EXCEEDS) | box look | Q | return_04 |
| 29 | Col Rate, Value, GST (read-only, in this order after Qty) | DIFFERENT | Rate, GST %, Value, Tax, Total come AFTER Outcome / Claim On, and GST shows a % not the amount (36.36 in the mockup) | order; GST amount; Total is extra but off-screen (scroll) | Q | return_04 (Total cut at the right edge) |
| 30 | Col Outcome (boxed combo, red border when not Return, full wording) | DIFFERENT | Upper-case codes "RETURN / CLAIM SENT / CLAIM DESTROYED / REPLACE", popup editor | wording "Return (debit note)", "Claim - goods sent", "Claim - destroyed here", "Replace"; boxed look | Q | return_04; RC:25-30 |
| 31 | Col Reason (boxed combo from Stock Reasons 293) | NO | Not present; `reasonId` never sent | column + dropdown + seed | Q + D | DTO has `reasonId`; `stock_reason_master` has 0 purchase-return rows |
| 32 | Col "Credit to" (Supplier now / Claim receivable amber / "- (shortage row)" blue) | NO | "Claim On" (SUPPLIER / TRANSPORTER / INSURER) only on claim rows | the consequence text; claim-party still needed for non-supplier claims | Q | return_04 |
| 33 | Col "Writes" (Dr Sup / Cr Stk; ITC reversed red; Replace due 5) | NO | Not present | per-row posting preview | Q | return_04 |
| 34 | Blank prompt row ("pick rows from the bill (Ctrl+L) - or start from the bill: Ctrl+F11 all / Ctrl+F12 selected") | NO | Grid is empty until a bill is picked; hint line below the grid | prompt row; Ctrl+L picker; Ctrl+F12 | Q | return_01 |
| 35 | `+` on a row = split into another outcome (house key) | YES | `lineCreateRequested` -> `splitRow` | - | - | RE:178-186; test_purchase_return R2 |
| 36 | Grid width: the full row fits in 1560 | DIFFERENT | Horizontal scroll bar appears; Total column cut | column widths (ui_table 59) | Q/registration | return_04 (scroll bar, Total clipped) |
| **Outcome legend / totals** | | | | | | |
| 37 | "THE FOUR OUTCOMES" legend table (5 cols x 4 rows) | NO | Nothing | whole panel (static text) | Q | return_04 vs mockup |
| 38 | Strip: Rows | PARTLY | "4 row(s) going back" in a text line | strip cell | Q | return_04 bottom line |
| 39 | Strip: Return now (debit note) 264.56 | PARTLY | "Debit note 236.00" bold in the text line (taxable+tax of RETURN rows) | strip cell | Q | return_04 |
| 40 | Strip: Claims raised (amber) | PARTLY | "Claims 218.00" | strip cell, amber colour | Q | return_04 |
| 41 | Strip: Replace qty due (blue) "5 pcs" | PARTLY | "Replacement due 3" only when > 0 | strip cell, unit, blue | Q | return_04 |
| 42 | Strip: ITC reversed (red) 107.00 | DIFFERENT | Before save: nothing. After save: "GST reversed on destroyed goods 12.00" (server `prItcReversed` = destroyed goods only; the mockup is every outcome) | live preview of ITC on all outcomes; server field semantics | Q + S | return_04, return_07; purchase-return.ts:345 |
| 43 | Strip: Supplier balance now "- 264.56" | NO | Not shown | outstanding after this return; `/purchase/lookup/supplier-context` already returns outstanding | Q | return_04 |
| 44 | Totals in a framed box (the "total frame") | NO | Plain label under the grid, no frame | framed strip | Q | return_04 |
| **Buttons** | | | | | | |
| 45 | Left row: "From bill - Ctrl+L" | NO | Bill is picked only from the header combo | button + Ctrl+L bill / lines picker | Q | return_01 button row |
| 46 | Left row: "Batches - F12" | NO | No F12 on this screen (rows carry one lot; bucket editable) | button / dialog (when a bill line has several lots) | Q | grep: no F12 in return/ |
| 47 | Left row: "Print debit note / claim memo" | PARTLY | "Print - Ctrl+P" (one print) | two papers (debit note vs claim memo) | Q (+print templates) | PurchaseDocNav |
| 48 | Right row order: New, Save Draft, Post, Edit, Cancel, History, Show List, Close | DIFFERENT | New, Save Draft, **Claims...**, Post is before it, Edit F2, Cancel, History, Print, Close | **Show List - Ctrl+O missing**; Claims... is extra; Print is extra | Q | return_01 button row; RE:164-186 |
| 49 | Post primary (dark red) | YES | Primary button | - | - | return_01 |
| 50 | Edit - F2 / Cancel - Alt+X / History - Ctrl+H keys | YES | Bound through PurchaseDocNav + TxnKeys; Cancel disabled until a doc is saved | - | - | return_01; test_purchase_doc_keys |
| **Callouts** | | | | | | |
| 51 | (1) Against invoice or GRN; GRN = rejection at the gate, no GST | YES | Against = GRN limits Outcome to RETURN; server posts Dr GRNI / Cr Stock | wording only | - | RE:210-212 (`choices`); test_purchase_return |
| 52 | (2) Settlement of "Return" rows: next bill / Payment / cash / advance, company default | PARTLY | Adjust / Advance + company default; Payment already honours PURCHASE_RETURN held debit | cash refund choice (see row 12) | Q (decision) | RE:126-130 |
| 53 | (3) Outcome per row: four outcomes in one document | YES | Verified: RETURN + CLAIM SENT + CLAIM DESTROYED + REPLACE posted in one return | - | - | return_06_posted.png; return_modal_02_posted.png |
| 54 | (4) Exact batch: rows from the bill's own allocation, lot exact (expiry, MRP, cost); qty capped at billed - returned; reasons from 293 | PARTLY | Lot exact via bucket + lotId; batch shown; cap by server | expiry / MRP in the row; client-side cap; reasons | Q + D | return_04; RE:fillRow (~line 340) |
| 55 | (5) Value = bill rate; difference to PPV; ITC reversed at post | YES | Rate = bill rate; PPV posted by the server (test: voucher balances) | - | - | test_purchase_return F/R3 |
| 56 | (6) Edit / cancel: locked once adjusted or a claim is part-settled; cancel writes reversing rows | YES | Server refuses; the screen shows the refusal (tested: "cancel refused while a claim is settled") | - | - | test_purchase_return X |
| 57 | (7) Papers for goods going back: e-way bill at >= 50,000 recorded; Supplier CN no / date | PARTLY | Supplier CN no / date yes; e-way bill no | e-way field + rule | Q | row 16 |
| **Warnings / messages** | | | | | | |
| 58 | Refusals by code (qty exceeds, claim part-settled, etc.) | YES | `showRefusal` puts the plain text in the hint line and a warning box | - | - | RE:670; test_purchase_return |
| **Loaded document** | | | | | | |
| 59 | Reopened document shows the saved header | PARTLY | Supplier, bill (refno), date, CN no, vehicle, remarks, settlement load. **Claim on (other) comes back blank** (party not restored); the hint under the grid is stale ("Pick the supplier, then the bill the goods came on."); Can Return column blank | restore claim party; refresh hint; Returnable on a posted doc | Q | return_07_reopened_posted.png |
| 60 | Posted state locks the grid and header | YES | Against / Bill / Settlement disabled, Save Draft / Post disabled, Cancel enabled | - | - | return_07 |
| 61 | Cancelled state | YES | Status text CANCELLED, all locked | - | - | return_08_cancelled.png |
| 62 | Against-GRN mode snapshot | NOT RENDERED | Needs a posted un-billed GRN of the ZT supplier; logic read from code only | - | - | - |

## Server-dependent checks on :3011 (swagger /api/docs-json, 2026-10-29)

- `POST /purchase/returns/save` `ReturnHeaderDto`: has `ewbNo`, `driverName`, `vehicleNo`, `settlement`, `supplierCnNo/Date`, `remarks`. **No** `godownId`, `placeOfSupply`, `ourRef`, `transport`. `ReturnLineDto`: has `reasonId`, `freeQty`, `remarks`, `claimAgainst`, `claimPartyId`.
- `GET /purchase/returns/get` returns the whole `pur_return` row (`prGodownId`, `prPosStcd`, `prEwbNo`, `prDriverName`, `prOurRef`) and per line `expiryDate`, `claimValue`, `claimSettled`, `claimOpen`. So Place of sup, E-way, Driver, Batch-Exp, claim open are all Qt-only gaps.
- `purchase/returns/sources` + `source-lines`: not checked whether billed / returned qty are returned (Qt only reads `returnableQty`).
- DB: `stock.stock_reason_master` has zero rows allowing a purchase return (`srm_allowed_txn_types` never contains a PURCHASE_* type), so the Reason dropdown has nothing to show until a set is seeded.
