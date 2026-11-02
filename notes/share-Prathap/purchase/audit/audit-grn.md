# Audit — GRN (menu 196) vs purchase_grn_ui_mockup.png (+ GRN parts of purchase_textile_lines_ui_mockup.png)

Auditor: qt-tester, 2026-10-29. Spec: `purchase_grn_ui_mockup.png` (all 10 callouts read), textile mockup (callouts 1-6), plan-purchase-phase1.md §3.2 (DOC-03), addenda 1/2.
Screen: `src/modules/purchase/grn/grn_entry.cpp` (built in C++, no .ui), grid ui_table 58 (`grn_cols.h`, 32 columns incl. hidden). F12 = `invoice/purchase_alloc_dialog.*`, labels = `common/purchase_labels.*`.
Snapshots (real screen, offscreen 1560 x 1000, live :3011 as VKPOS, theme v41) in `/home/vk/Shared/Prathap/purchase/audit/`: `grn_01_new_empty`, `grn_02_supplier_picked_po_offered`, `grn_03_filled_from_po` (3 PO lines; 8 of 10 / 3 of 5 received, challan 5, 2 damaged), `grn_04_F12_batch_split_popup`, `grn_05_posted`, `grn_06_labels_dialog`, `grn_07_existing_posted_loaded` (GRN 00248), `grn_08_existing_draft_loaded` (GRN 00244), `grn_09_textile_mode_empty` (autoBatchNo on). Harness: `tests/purchase/audit_snap_po_grn.cpp`. Posted GRN 00256 and its PO were cancelled afterwards (stock back).

## Summary
Rows (57 = elements + callouts): **YES 17 · PARTLY 11 · DIFFERENT 8 · NO 21**. The posting behaviour (stock, GRNI, coolie per loadman, shortage, shelf-life warning, excess, labels, per-row godown, ratio pack inside F12, Sell A–D) is built server-side and mostly reachable from the screen. The **look** is far from the mockup: most header fields of rows 2-5 are missing, no charges panel, no inline F12 panel, no warning strip, no Check / Shelf life / Labels / Dmg reason columns, no totals strip, and the key row misses the 5 action buttons.

**The 10 biggest gaps**
1. **Check column (in-cell chips OK / SHORT 2 / 2 DAMAGED / SHELF LIFE / 5/5 SN)** absent. Also **Shelf life %, Labels, Dmg reason** columns absent (Qt can compute Check and Labels; Shelf life % needs the server to return the figure, Dmg reason needs the reasons dropdown; DTO `damageReasonId` exists).
2. **No WARN strip** (yellow band with SHELF_LIFE / SHORT_VS_CHALLAN codes and "override (logged)" tick). Built: the server's warnings arrive in a message box after Post and one hint line (grn_05). The override tick has no DTO field (server).
3. **No off-bill CHARGES panel** (Charge / Basis / Rate / Amount / Payee loadman-wise; own lorry freight). Built: only the "Unloaded by" multi-select and a "· Coolie N" suffix on the totals. The GRN DTO has `payCoolie` but no `charges[]`: own lorry freight etc. need a server change.
4. **No inline F12 batch-split panel.** Built as a modal popup (grn_04): columns Godown / Bucket / Batch / Mfg / Expiry / MRP / Serial no / Warranty to / Qty / Free; mockup panel has a Lot ("new" / existing) column and sits under the grid. Functionally equivalent, visually different.
5. **Header is missing 9 fields:** From-PO is a single-PO combo (mockup: multi-PO "PO/…/00004, 00005"), no Received date-**time**, no Bill status, no Supervisor, no Driver, no Transport (LR · transporter, shared band), no E-way bill, no Packages (sent / recd), no Received by. Server DTO already takes supervisorId, driverName, ewbNo, packagesSent / packagesReceived, receivedAt; the transport band (LR / transporter) has no DTO field.
6. **Action buttons:** missing **From PO · Ctrl+L**, **Reject (return to supplier)**, **Make bill · Ctrl+B**; "Print labels · Ctrl+P" is "Labels…" with no key (Ctrl+P is the doc Print); **Show List · Ctrl+O** absent.
7. **Totals strip** (Lines / Received (base) / Damaged / Short (vs challan) "2 cases" / Off-bill charges / Provisional value) is one text line "n line(s) · Qty · Value (provisional) · Short on challan" (grn_03).
8. **Godown and Received-by blank on a reopened GRN** (grn_05, grn_07): `applyLoaded` does `setSelection(godownId, QString())`, no name. Godown is a required (*) field and the stock-bearing one. (Memory purchase-module-build lists this as open on menu 24 too.)
9. **Grid overflows**: ui_table 58 visible widths sum to 97 % even without textile / Sell columns and up to ~140 % with them, so the grid always scrolls sideways at 1560 px; Expiry / MRP are off-screen in grn_03 and grn_05. Mockup fits on one screen.
10. **Textile columns:** Model / Size / Colour are placed after Batch (grn_09); mockup puts them before Batch and shows "given on post" in the Batch cell. No Markup %, Check chips (SLAB 18 % / FABRIC 5 % / EXISTING BATCH), ratio-pack panel on the lines card (it lives inside the F12 popup), or label-preview card (the labels dialog is a table, grn_06).

## Table
Layer: Q = Qt only · S = needs server route/field · D = needs DB (ui_table / migration).

| # | mockup element / callout | built? | what is there now | what's missing | layer | evidence |
|---|---|---|---|---|---|---|
| 1 | Title "GRN — GRN/2026-2027/D1/00007" | YES | "GRN — GRN/…/00256" | — | — | grn_05 |
| 2 | Sub-title "Acme Foods · Coimbatore" | NO | none | company · branch line | Q | grn_01 |
| 3 | Status chip DRAFT (pill) | DIFFERENT | plain text DRAFT / POSTED / CANCELLED | pill, and "Part / Billed" state | Q | grn_05 |
| 4 | Heading "THE HEADER" | NO | none | heading | Q | grn_01 |
| 5 | Supplier * (callout 1) | YES | dd 35 | — | — | grn_02 |
| 6 | **From PO** (several POs, "PO/…/00004, 00005") | PARTLY | "Against PO" single combo; "— without PO —" shown even after a PO filled the lines (grn_03) | multi-PO pick and the chosen POs shown; Ctrl+L | Q | grn_03 |
| 7 | Received * date **and time** ("24/10/2026 04:35 PM") | DIFFERENT | "GRN Date *" date only | time (DTO `receivedAt`) | Q | grn_01; swagger GrnHeaderDto |
| 8 | GRN No (read-only) | YES | | | | grn_05 |
| 9 | Godown * | PARTLY | dd 26, saved | blank after reopen (name not set) | Q | grn_05, grn_07 |
| 10 | Challan / Inv (SMA/1452 "if known", callout 2) | YES | "Challan No" | label only | — | grn_01 |
| 11 | Challan date | YES | | | | grn_01 |
| 12 | **Bill status** ("Not billed yet" / Part / Billed) | NO | "· Billed" appended to totals text only when billed | read-only field; /get returns `billStatus` | Q | applyLoaded; grn_05 |
| 13 | Loadmen (callout 9) | DIFFERENT | "Unloaded by" multi-select dd 72, spans 4 columns, row 3 | label "Loadmen", position; names show only if loaded (works: test) | Q | grn_01 |
| 14 | **Supervisor** | NO | none | dropdown; DTO `supervisorId` exists | Q | swagger |
| 15 | Vehicle | YES | | | | grn_01 |
| 16 | **Driver** | NO | none | DTO `driverName` exists | Q | swagger |
| 17 | **Transport** ("LR 88412 · KPN Parcel", shared transport band) | NO | none | LR no / transporter; the shared txn_transport_detail band / shipping dialog; no field in the GRN DTO | S + Q | DB public.txn_transport_detail has ttd_lr_no, ttd_transporter_*; GrnHeaderDto |
| 18 | **E-way bill** ("3512 8840 1123 · 24/10") | NO | none | DTO `ewbNo` (date col `grn_ewb_date` exists, DTO has no date) | Q (+ S for date) | swagger |
| 19 | **Packages** ("34 recd of 34 sent") | NO | none | DTO `packagesSent` / `packagesReceived` | Q | swagger |
| 20 | **Received by** ("Bala (supervisor)") | NO | the loadmen list is stored in `grn_received_by`; no separate field | clarify with plan: supervisor vs received-by | Q / question | server `grn_received_by uuid[]` |
| 21 | Remarks | YES | | | | grn_01 |
| 22 | Heading "WHAT ARRIVED — rate is PROVISIONAL here…" | NO | none | heading + note | Q | grn_03 |
| 23 | Col Barcode (callout 10) | YES | first column | supplier's own code also finds the item: untested | Q | grn_03 |
| 24 | Col Item, Unit | YES | | | | grn_03 |
| 25 | Col **PO pend.** | DIFFERENT | two columns "PO" (ref, last segment "00192") and "Pending" | single "PO pend." qty (the ref column is extra but useful) | D | grn_03 |
| 26 | Col Challan, Received (Recd), Dmg, Free, Short | YES | all present; Short red; typed Recd frames follow the theme | heading "Received" vs "Recd" | — | grn_03 |
| 27 | Col **Dmg reason** (callout 3: "Wet carton") | NO | none | reason per line; DTO `damageReasonId` + stock reasons master (menu 293) | D (ui_table col) + Q | swagger GrnLineDto |
| 28 | Col Prov. rate | DIFFERENT | "Rate" (+ extra **Value** column) | heading; Value is extra | D | grn_03 |
| 29 | Col Batch / Serial (in-cell "2 batches (F12)", "5 serials (F12)", typed batch) | YES | typed batch, "n batches (F12)" | shows "n batches" also for serials (not "n serials") | Q | grn_07 row 2 |
| 30 | Col Expiry | PARTLY | present | off-screen at 1560 px (widths) | D | grn_03 |
| 31 | Col MRP | PARTLY | present | off-screen at 1560 px | D | grn_03, grn_07 |
| 32 | Col **Shelf life %** (green 82 % / red 41 %, callout 4) | NO | none | per-line % left; the server only returns warnings text | S (return % per line) or Q (needs item shelf-life days) | server shelfLifeIssues, purchase-grn.ts l.170 |
| 33 | Col **Sell A** (callout 4) | PARTLY | Sell A–D columns appear only when setting purchase.price_set_at = GRN; this company: hidden | mockup assumes visible; behaviour is setting-driven | — (by design) | grn_03 (hidden), cpp m_priceAtGrn |
| 34 | Col **Labels** (qty = received + free) | NO | none (a Labels… dialog after post) | per-line label count column | Q | grn_03 |
| 35 | Col **Check** chips OK / SHORT 2 / 2 DAMAGED / SHELF LIFE / 5/5 SN | NO | none; Short is plain red number | chips (DelegatePaint::background) | Q (SHELF LIFE chip needs #32) | grn_03 |
| 36 | Extra built columns: Excess, Excess action (HOLD_FOR_BILL / KEEP_FREE / RETURN) | DIFFERENT | shown, raw codes "HOLD_FOR_B…" truncated (grn_05) | not in the mockup; labels are raw enum codes | Q | grn_05 |
| 37 | Blank-row prompt "scan barcode · or pick PO lines (Ctrl+L) · F12 batch / godown / MRP / expiry split · + / − lines" | NO | empty blank row | prompt | Q | grn_01 |
| 38 | Panel "F12 — BATCH SPLIT (line 3)" inline, Godown / Bucket / Batch / Expiry / MRP / Qty / Free / **Lot** ("new"), Σ line | DIFFERENT | modal popup "Batches / serials — line n" (grn_04) with Σ check line | inline panel; Lot column; popup has extra Mfg, Serial no, Warranty to | Q (Lot needs S: existing-lot match) | grn_04 |
| 39 | Panel "CHARGES (off the bill — our own labour)": Unloading coolie (Qty-wise, rate 2.00) split per loadman, Own lorry freight | NO | none; server auto-creates the UNLOADING charge, screen shows "· Coolie 464.00" suffix | whole panel; own-charge rows need `charges[]` in DTO | S + Q | grn_05, swagger GrnHeaderDto |
| 40 | Totals strip (6 cells) | DIFFERENT | one line "n line(s) · Qty · Value (provisional — the bill decides) · Short on challan" | Lines / Received (base) / Damaged / Short in cases / Off-bill / Provisional value as cells | Q | grn_03 |
| 41 | WARN strip: `SHELF_LIFE`, `SHORT_VS_CHALLAN` + "override (logged)" tick | NO | warnings shown in a message box after Post and the hint label | strip with codes, live while typing, override tick | Q (strip) + S (override flag; live check needs the server to evaluate on `validate`: there is no GRN validate route) | grn_05; swagger has grns/{save,post,get,cancel} only |
| 42 | Button **From PO · Ctrl+L** | NO | combo only | button + key | Q | grn_01 |
| 43 | Button **Batches · F12** | YES | "Batches / serials · F12" | label | — | grn_01 |
| 44 | Button **Print labels · Ctrl+P** | PARTLY | "Labels…" (enabled after Post), no key; Ctrl+P is document Print | key conflict: mockup assigns Ctrl+P to labels | Q (decide) | grn_05 |
| 45 | Button **Reject (return to supplier)** (callout 8) | NO | none; return-against-GRN exists on the Return screen | button that starts the return with this GRN as source | Q | cpp buttons; Return screen /purchase/returns/sources |
| 46 | Button **Make bill · Ctrl+B** | NO | none; the invoice has "From GRN" | button that opens the bill on this GRN | Q | grn_05 |
| 47 | TxnKeys New / Save Draft / Post / Edit F2 / Cancel Alt+X / History / Close | YES | all present ("Post · Ctrl+Enter" label correct here) | | | grn_03 |
| 48 | **Show List · Ctrl+O** | NO | absent; Print · Ctrl+P instead | button | Q | grn_03 |
| 49 | Callout 1 "From PO(s)" picking supplier offers pending lines; GRN without a PO allowed | YES | combo lists pending POs; direct allowed | multi-PO | Q | grn_02 |
| 50 | Callout 2 "Bill may not exist yet; Bill status Not billed / Part / Billed" | PARTLY | challan optional YES; Bill status field NO (#12) | field | Q | grn_01 |
| 51 | Callout 3 "Received + Free to SALEABLE, Damaged to DAMAGED, Challan − received = SHORT → shortage register" | YES | server does it; Short column; test GRN 00256 opened shortage | — | — | grn_05; server l.394 |
| 52 | Callout 4 "Shelf life % + labels" | PARTLY | server WARN / REFUSE (setting); labels after post dialog (36 labels, grn_06) | per-line Shelf life %, Labels columns (#32, #34) | S/Q | grn_06 |
| 53 | Callout 5 "Batch split (F12), textile auto batch, Σ must equal received + free + damaged" | YES | popup with Σ check; textile choose Model / Size / Colour | inline panel (#38) | — | grn_04, grn_09 |
| 54 | Callout 6 "Our own labour, coolie split per loadman, paid later in Payment" | PARTLY | server does it; screen picks loadmen | charges panel (#39) | S/Q | test_purchase_po_grn |
| 55 | Callout 7 "What Post writes" (stock at provisional rate, Dr Stock / Cr GRNI) | YES | server | — | — | server purchase-grn.ts |
| 56 | Callout 9 / 10 "Crew, vehicle, papers; line extras: Mfg, Weight, Remarks" | PARTLY | crew + vehicle only; Mfg is in the F12 popup | weight / remarks columns (DTO has `weight`, `remarks`), supervisor, driver, LR, e-way, packages | D + Q | swagger GrnLineDto |
| 57 | **Textile (textile mockup):** Model / design, Size, Colour **before** Batch; Batch "given on post"; Markup %; Sell; MRP; Labels; Check chips; Ratio pack (Ctrl+P) panel; label preview card | PARTLY | Model / Size / Colour columns shown when stock.auto_batch_no (grn_09), but after Batch; ratio pack lives inside the F12 popup (header comment in purchase_alloc_dialog.h; not exercised here); Labels dialog is a table with "Preview & print" | order, "given on post" text, Markup %, chips, the ratio-pack and label cards on the screen | D (order) + Q | grn_09, grn_06 |

## Server routes checked on :3011 (swagger)
Present: `/purchase/grns/{save,post,get,cancel}`, `/purchase/labels/from-doc`, `/purchase/orders/pending-lines`, `/purchase/invoices/settings`, `/purchase/lookup/{item-context,supplier-context}`.
No `/purchase/grns/validate` (nothing to drive live WARN strip / Check chips except local rules).
`GrnHeaderDto`: grnId … supplierId, godownId, docDate, **receivedAt**, challanNo, challanDate, supplierInvNo, receivedBy[], payCoolie, **supervisorId**, vehicleNo, **driverName**, **ewbNo**, **packagesSent**, **packagesReceived**, remarks, post. Missing vs mockup: transporter / LR, e-way date, charges[], warning-override flag.
`GrnLineDto`: lineNo, itemId, uomId, poItemId, poItemAccYear, challanQty, receivedQty, damagedQty, **damageReasonId**, freeQty, excessQty, excessAction, rate, mrp, sellA–D, **weight**, **remarks**, allocs[].
`/get` header returns `billStatus` and `receivedBy` as {id, name}; no godown name, no supervisor name.

## Agent-file feedback (qt-tester)
- Same as audit-po.md: modal closer is mandatory in a snapshot program; start the first build in the background; test items must be checked for service / inactive.
- For an audit task the agent file should say where the Swagger JSON and the source of the DTOs are, and that screens are built in C++ (`buildUi()`) rather than `.ui`, so "compare to the .ui" does not apply.
