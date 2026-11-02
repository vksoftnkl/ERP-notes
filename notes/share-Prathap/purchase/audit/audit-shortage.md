# Audit: Shortage Receipt (menu 195) vs section B of purchase_claims_shortage_ui_mockup.png

Audited 2026-10-29 by qt-tester. Screen: `src/modules/purchase/shortage/shortage_receipt_entry.{h,cpp}` + `shortage_receipt_cols.h`, grid ui_table 61. Mockup: `/home/vk/Shared/Prathap/purchase/purchase_claims_shortage_ui_mockup.png`, section "B. SHORTAGES & REPLACEMENTS - goods expected", its strip, buttons, callouts 3 and 4.
Snapshots in `/home/vk/Shared/Prathap/purchase/audit/`: `shortage_01..07*.png`, `shortage_modal_*.png`. Rendered at 1560 x 1000, live :3011 as VKPOS, ZT bill + ZT return with a REPLACE row (all cancelled afterwards).

Layer: **Q** Qt only, **S** server route / field / DTO, **D** DB. SE = `shortage_receipt_entry.cpp`, SC = `shortage_receipt_cols.h`.

## Summary

Rows audited: 44 (43 judged, 1 not rendered).

| Verdict | Count |
|---|---|
| YES | 9 |
| PARTLY | 10 |
| DIFFERENT | 9 |
| NO | 15 |

The core flow is built and tested (open shortages as rows, receive qty, batch / expiry, post, cancel, close by supplier CN or write-off, undo a part close). The mockup's header, the 16-column grid, the 5-figure strip, the chips and the F12 / "Close row" buttons are not.

### The 10 biggest gaps

1. **Grid has 11 of the mockup's 16 columns, with different content.** Missing: tick box, Date, Age, Recd, Closed, Open (the grid shows one "Short" = open qty), GST held, Outcome chip. The `/shortages/open` answer already returns `received`, `closed`, `qtyBase`, `openGst`, `gst`, so Recd / Closed / GST held are Qt only; Date and Age need a source-date field from the server. Extra columns not in the mockup: Unit, MRP, Value.
2. **No tick-box selection model.** The mockup ticks rows and the strip says "Ticked rows 2"; the build treats any row with a typed Receive qty as picked. Qt only.
3. **Totals strip is a text line**: "2 row(s) received - Qty 5 - Value 350.00 (of 2 short row(s))". The mockup strip has Ticked rows, Receive now (base), Stock in at original cost, **ITC released (green)**, **Still open after this (amber)**. ITC released is `openGst` x fraction (Qt can compute from the rows); "Still open after this" is a row count (Qt). Qt only.
4. **Header differs**: Received is a date only (mockup date + time "28/10/2026 11:20 AM"); the server stamps `psr_received_at = now()` and the DTO has no `receivedAt` -> S. "Received by (supervisor)" does not exist; the built "Unloaded by" fills the DTO `receivedBy` array, which the DB stores as one uuid[] column of loadmen. The mockup's two people fields (Received by + Loadmen) need a second column -> D + S. The mockup header has no Supplier field (supplier is the page title); the build has "Supplier *" in the header (reasonable, but a difference).
5. **"Close row... (CN / claim / write-off)" is three buttons, and CLAIM does not exist.** Built: Supplier CN..., Write off..., Undo close... The server `CloseShortageDto.kind` accepts only SUPPLIER_CN and WRITE_OFF; callout 4 says a transporter / insurer claim closes to CLAIM_RECEIVABLE. Needs S (new kind + party) then Q. The Qt close also always closes the whole open qty and never sends `cnDate` / `qty` (DTO supports both).
6. **Batches - F12 button and dialog missing**; serial items cannot be received (DTO has batchNo / expiry / mrp only; no allocs[]). Q + S for serials; Qt-only for batch rows.
7. **Reopened receipt loses the header**: after loading a posted receipt the Godown is blank (code passes an empty name: `setGodown(id, QString())`), Loadmen are blank, "Short" shows 0 and the hint line still says "Pick the supplier - every open shortage ... comes in as a row." (shortage_07). Qt only; `/receipt/get` needs to return the godown name and the loadmen (check) -> maybe S.
8. **No section heading / title / status chip**: no "B. SHORTAGES & REPLACEMENTS - goods expected" heading with caption, no "Supplier dues - {supplier}" title, no "3 OPEN" chip; status is plain text. Qt only.
9. **In-cell treatment**: Receive now as a boxed red-bordered input on the focused row, Batch / Serial and Expiry as boxes, Outcome chips ("Supplier sends" blue, "Replacement" violet), cost / GST-held read-outs. The built grid is flat text. Qt only.
10. **Reason wording**: "Billed, not came / Short on challan / Replacement due" vs the mockup's "Short on bill / Short vs challan / Replace". Qt only.

## Table

| # | Mockup element / callout | Built? | What is there now | What is missing | Layer | Evidence |
|---|---|---|---|---|---|---|
| **Frame** | | | | | | |
| 1 | Page title "Supplier dues - Sri Murugan Agencies" | DIFFERENT | "Shortage Receipt - new / - SHR/..." small label | big title; supplier in title | Q | shortage_01_new_empty.png; SE:64 |
| 2 | Sub-title company / branch | NO | not shown | line | Q | shortage_01 |
| 3 | Status chip ("3 OPEN") | DIFFERENT | plain text DRAFT / POSTED / CANCELLED | pill chip; open-count | Q | shortage_05_posted.png |
| 4 | Heading "B. SHORTAGES & REPLACEMENTS - goods expected" + caption "tick rows, receive what came -> Shortage Receipt (stock in at the ORIGINAL bill cost)" | NO | one hint line below the grid | heading + caption | Q | shortage_02_open_rows.png |
| 5 | One page with the claims section A | NO | separate menu 195 screen; claims is a dialog elsewhere | shared page (decision, see audit-claims) | Q | frontend plan F6 |
| **Header** | | | | | | |
| 6 | Receipt No (disabled grey) | PARTLY | read-only, "given on save", looks white once loaded | greyed look | Q | shortage_07_reopened_posted.png |
| 7 | Received * (date + time, "28/10/2026 11:20 AM") | DIFFERENT | "Date *" date only, `29-10-2026` | time (AM/PM), slash format, editable | Q + S (DTO has no receivedAt; server stamps now) | SE:112-115; `ShortageReceiptHeaderDto` |
| 8 | Godown (combo) | YES | Godown * dropdown 26 (branch godowns), required | shown blank after reopen (row 42) | - | shortage_03_received_typed.png |
| 9 | Challan no | YES | Challan No text | - | - | shortage_03 |
| 10 | Received by (supervisor combo "Bala (supervisor)") | NO | no field | second person field | D + S + Q | `pur_shortage_receipt.psr_received_by` = uuid[] |
| 11 | Loadmen (combo "Ravi") | PARTLY | "Unloaded by" multi-dropdown 72 (LOADMAN), maps to `receivedBy` | label "Loadmen"; it is the only person field | Q | SE:117; `psrReceivedBy: h.receivedBy` |
| 12 | Remarks (span 2) | YES | Remarks text | - | - | shortage_03 |
| 13 | Supplier is the page's title, not a header field | DIFFERENT | "Supplier *" dropdown at top of the header (needed to load the open rows) | none needed | - | shortage_01 |
| **Grid** | | | | | | |
| 14 | Tick box column | NO | row gutter "#" | tick column | Q | shortage_02 |
| 15 | From ("PUR/.../00012 - line 2") | PARTLY | "Short From" = source refno only (no line no) | " - line n" | Q (S if line no missing) | shortage_02; SC:13 |
| 16 | Date (source doc date) | NO | not shown | `/shortages/open` has no source date | S | `open()` field list, purchase-shortage.ts:213-229 |
| 17 | Age | NO | not shown | days from the source date | S + Q | same |
| 18 | Item ("Tata Salt 1kg (CASE-25)") | PARTLY | item name; Unit is a separate column | name + unit in one cell (or keep) | Q | shortage_02 |
| 19 | Why (Short on bill / Replace / Short vs challan) | DIFFERENT | "Billed, not came / Short on challan / Replacement due" | wording | Q | SE:36-41 |
| 20 | Short (original short qty) | DIFFERENT | one "Short" column showing the OPEN qty | original short qty (`qtyBase`) | Q | shortage_02 |
| 21 | Recd (received so far) | NO | not shown; `received` is in the answer | column | Q | `open()` has `received` |
| 22 | Closed (closed so far) | NO | not shown; `closed` is in the answer | column | Q | `open()` has `closed` |
| 23 | Open | PARTLY | the "Short" column is this value | label | Q | shortage_02 |
| 24 | Receive now (boxed input, red border on focus) | PARTLY | "Received" column, editable | title, box look | Q | shortage_03 |
| 25 | Batch / Serial (boxed) | PARTLY | Batch text, defaults to the source batch; no serials | serial entry; F12 | Q + S (serials) | shortage_03 |
| 26 | Expiry (dd/MM/yyyy box) | YES | editable, validated dd/MM/yyyy | - | - | shortage_03 |
| 27 | Cost (original) | YES | cost per unit | - | - | shortage_02 |
| 28 | GST held | NO | not shown; `openGst` is in the answer | column | Q | `open()` has `openGst` |
| 29 | Outcome chip (Supplier sends blue / Replacement violet) | NO | not shown | chip derived from the reason | Q | shortage_02 |
| 30 | Extra columns not in the mockup: Unit, MRP, Value | DIFFERENT | present | decide keep / hide | Q | shortage_03 |
| 31 | Rows pre-filled with the supplier's open shortages of all kinds (bill short, challan short, Replace) | YES | all three reasons loaded; Replace row from the return appeared | - | - | shortage_02_open_rows.png |
| **Strip** | | | | | | |
| 32 | Ticked rows / Receive now (base) / Stock in at original cost | PARTLY | "2 row(s) received - Qty 5 - Value 350.00 (of 2 short row(s))" in plain text | 3 strip cells | Q | shortage_03 |
| 33 | ITC released (green) | NO | not shown | `openGst` share per row | Q | shortage_03 |
| 34 | Still open after this ("1 row") | NO | only "of N short row(s)" | count of rows left open | Q | shortage_03 |
| 35 | Strip drawn as a framed panel | NO | plain label | frame | Q | shortage_03 |
| **Buttons** | | | | | | |
| 36 | "Batches - F12" | NO | none | button + dialog | Q (+S serials) | shortage_03 button row |
| 37 | "Close row... (CN / claim / write-off)" | DIFFERENT | Supplier CN..., Write off..., Undo close... (3 buttons, left) | one button / dialog with the CLAIM choice (transporter / insurer) | Q + S (kind CLAIM) | `CloseShortageDto.kind` enum = SUPPLIER_CN, WRITE_OFF; shortage_modal_03 |
| 38 | Right row: New, Save Draft, Post receipt, History, Show List, Close | DIFFERENT | New, Save Draft, **Post** (label "Post - Ctrl+Enter", not "Post receipt"), Edit F2, Cancel, History, Print, Close; **Show List - Ctrl+O missing** | label; Show List | Q | shortage_01 button row; SE:131-170 |
| **Callouts** | | | | | | |
| 39 | (3) Shortage Receipt receives against open rows; stock in at the ORIGINAL bill cost; Dr Stock / Cr SHORTAGE_RECEIVABLE; Dr Input GST / Cr INPUT_TAX_PENDING | YES | Cost 100.00 on the row = bill cost; server posts balanced voucher, releases the pending GST (checked by test_purchase_shortage: voucher balances, GST 48 released) | - | - | shortage_05_posted.png; test_purchase_shortage S2 |
| 40 | (4) Other ways a shortage closes: Supplier CN; transporter / insurer claim -> CLAIM_RECEIVABLE; write-off with a reason | PARTLY | CN (prompt for the CN no) and write-off (prompt for the reason) work; claim kind missing; undo works for a part close | CLAIM; cn date; part qty | S + Q | shortage_modal_03_supplier_cn_prompt.png; shortage_06 |
| 41 | "Reports: shortage ageing, claims pending" | NOT RENDERED | reports are on hold (B10) | - | - | - |
| **Loaded / state checks** | | | | | | |
| 42 | Reopened posted receipt keeps its header and lines | PARTLY | lines + refno + challan + remarks + supplier come back; **Godown and Loadmen blank**, hint stale, "Short" 0 | restore godown name, loadmen | Q (+S if `/receipt/get` lacks the godown name) | shortage_07_reopened_posted.png; SE:`applyLoaded` setGodown(id, QString()) |
| 43 | Posted / cancelled lock | YES | grid + header disabled; Post / Save Draft off; Cancel on | - | - | shortage_07 |
| 44 | Refusals by code (qty exceeds, shortage closed, etc.) | YES | `showRefusal` plain text + warning box | - | - | SE:`showRefusal`; test_purchase_shortage |
