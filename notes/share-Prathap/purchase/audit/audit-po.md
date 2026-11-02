# Audit — Purchase Order (menu 23) vs purchase_order_ui_mockup.png

Auditor: qt-tester, 2026-10-29. Spec: `/home/vk/Shared/Prathap/purchase/purchase_order_ui_mockup.png` (read incl. all 5 callouts) + plan-purchase-phase1.md §3.1 (DOC-01/02).
Screen: `src/modules/purchase/order/purchase_order_entry.cpp` (built in C++, no .ui), grid ui_table 57 (`purchase_order_cols.h`).
Snapshots (real screen, offscreen, 1560 x 1000, live on :3011 as VKPOS, theme v41) in `/home/vk/Shared/Prathap/purchase/audit/`: `po_01_new_empty`, `po_02_new_keyed_5_lines` (3 lines took: 2 of my 5 test items are service / inactive), `po_03_confirmed`, `po_04_part_received`, `po_05_amending`, `po_06_merge_dialog`, `po_07_revisions_dialog`, `po_08_loaded_rev2_cancelled`, `po_09_revisions_rev2`. Snapshot code: `tests/purchase/audit_snap_po_grn.cpp` (ZT data, cancelled afterwards: PO/2026-2027/D14/00192 and GRN 00256 are CANCELLED).

## Summary
Rows below (53 = elements + callouts): **YES 18 · PARTLY 8 · DIFFERENT 7 · NO 20**. Behaviour of the PO engine (derived received / pending, amend = revision, clone, merge, approval, short close, validate) is built and works; what is missing is the **look of the mockup**: it has no section headings, no totals strip, no status chip, no in-cell status chips, and half of the lines columns and header fields are absent.

**The 10 biggest gaps**
1. **No totals strip.** Mockup: Lines / Ordered value / Received value / Pending value / Short-closed / PO total (incl. est. charges). Built: one text line "Taxable · GST · PO value" (po_04). Qt only for 5 of the 6 cells.
2. **Lines grid is missing 6 columns:** Sch %, Short-closed, Last rate, Stock, Reorder, **Status (DONE / PART / SHORT CLOSED chip)**. Last rate and Stock have a server route (`/purchase/lookup/item-context`); Short-closed is in `/orders/get` lines (poiShortClosedQty); Status is derivable from Recd / Pending.
3. **Column order and fit.** Mockup: Ordered Free Rate Disc Sch GST MRP Tax Amount Received Short-closed Pending. Built: Barcode, Qty, Free, Rate, Disc, **Taxable**, GST, Tax, Total, Recd, Pending, **MRP last** (MRP is pushed off the right edge: ui_table 57 visible widths sum to 105 %, the grid scrolls sideways at 1560 px, po_01). Heading "Qty" where the mockup says "Ordered".
4. **Header is missing 5 fields:** Agent, Credit days, Approval, Revision, Terms. Credit days and terms text are already in the save DTO (`creditDays`, `termsText`) and `po_credit_days` defaults from the supplier server-side; Agent has a column (`po_agent_id`) but no DTO field.
5. **Status chip.** Mockup shows a coloured chip PART RECEIVED / (rev in title "· rev 2"). Built: plain text CONFIRMED / DRAFT / SHORT-CLOSED / AMENDING; a part-received PO still reads CONFIRMED (po_04); revision is only shown in the totals line when > 1.
6. **No section headings / sub-title.** Mockup: "THE HEADER", "THE LINES — estimated charges only, nothing posts", sub-title "Acme Foods · Coimbatore". Built: none (po_01).
7. **Estimated charges** (CHG-07, the "incl. est. charges" in the PO total): no charges grid on the PO, no `charges` in the PO DTO; `po_charges_amt` exists in the table.
8. **Action row differs.** Missing: **Reorder suggestions · Ctrl+R** (the Reorder screen exists as its own menu 296, not reachable from the PO), **Make GRN · Ctrl+G**, **Send (PDF / WhatsApp)**, per-line Short close (Qt only closes the whole PO; server accepts `lineIds`). "Clone / merge pending" is two buttons; "Post · Ctrl+Enter" is labelled "Confirm" with no key shown; **Show List · Ctrl+O is absent** (Print · Ctrl+P is there instead).
9. **Blank-row prompt and hint** "type item / scan or Reorder suggestions (Ctrl+R) · + / − lines" absent (po_01 blank row is empty).
10. **Deliver-to is blank on a reopened PO** (po_04, po_05: the godown was set on the new PO; `applyLoaded` calls `setSelection(id, QString())`, so the name is never shown). Same fault on the GRN.

## Table
Layer: Q = Qt only · S = needs server route/field · D = needs DB (migration / ui_table registration). "ui_table" columns are DB rows registered by Prathap's registration script, but each also needs Qt code.

| # | mockup element / callout | built? | what is there now | what's missing | layer | evidence |
|---|---|---|---|---|---|---|
| 1 | Title "Purchase Order — PO/… · rev 2" | PARTLY | "Purchase Order — PO/2026-2027/D14/00192" | "· rev N" in the title (only in totals when >1) | Q | po_04; cpp applyStatus l.~640 |
| 2 | Sub-title "Acme Foods · Coimbatore" (company · branch) | NO | none | company / branch line under the title | Q | po_01 |
| 3 | Status chip PART RECEIVED (coloured pill, top right) | DIFFERENT | plain text label DRAFT / CONFIRMED / SHORT-CLOSED / AWAITING APPROVAL / AMENDING / CANCELLED | pill style; states PART RECEIVED, RECEIVED, CLOSED; part-received PO shows CONFIRMED | Q | po_03, po_04; applyStatus |
| 4 | Section heading "THE HEADER" | NO | none | heading | Q | po_01 |
| 5 | Supplier * | YES | NexDropdownSingle dd 35 | — | — | po_01 |
| 6 | PO Date * | YES | NexDateEdit, shows dd-MM-yyyy (mockup dd/MM/yyyy) | separator only | Q | po_01 |
| 7 | Expected | YES | NexDateEdit | — | — | po_01 |
| 8 | PO No (read-only) | YES | read-only, "given on save" placeholder | — | — | po_01 |
| 9 | Deliver to (callout 1 marker) | PARTLY | dropdown dd 26, saved as `godownId` | **blank when the PO is reopened** (name never set) | Q | po_04 vs po_02; applyLoaded `setSelection(id, QString())` |
| 10 | Agent | NO | none | Agent dropdown (agents menu 28 exists) | S (DTO has no agentId; column `po_agent_id` exists) + Q | swagger PurchaseOrderHeaderDto |
| 11 | Contact | YES | line edit (placeholder "who placed / confirmed it") | — | — | po_01 |
| 12 | Revision (read-only, "2 — qty raised on line 4") | NO | revision only in totals text when >1 | read-only field with the last amend reason | Q (revision in /get header; reason in /revisions) | po_08 |
| 13 | Credit days ("30 (supplier default)") | NO | none shown or sent | field, prefilled from supplier | Q (DTO `creditDays` exists; server defaults it) | swagger; cpp l.279 server |
| 14 | Approval ("Off (setting 0)") | NO | state only inside the status text; Approve / Not approve buttons when awaiting | read-only field | Q (`poApprovalStatus` in /get) | applyStatus |
| 15 | Terms (dropdown of a small text master, "Std terms #2 …") | NO | none | terms picker. No terms master table exists (only `po_terms_text` free text) | D (terms master + dd) + S (route/dd) + Q; or Q-only with a free-text box | DB: no *terms* table |
| 16 | Remarks (full-width row) | YES | line edit, shares row with Contact | position only | Q | po_01 |
| 17 | Section heading "THE LINES — estimated charges only, nothing posts" | NO | none | heading + note | Q | po_01 |
| 18 | Line col # | YES | row gutter | — | — | po_04 |
| 19 | Col Item | YES | popup item picker, + extra **Barcode** column in front (mockup scans into Item) | extra column | Q | po_04 |
| 20 | Col Unit | YES | | | | po_04 |
| 21 | Col **Ordered** | DIFFERENT | "Qty" | heading | D (ui_table name) | po_04 |
| 22 | Col Free | YES | | | | po_04 |
| 23 | Col Rate | YES | | | | po_04 |
| 24 | Col Disc % | YES | | | | po_04 |
| 25 | Col **Sch %** | NO | none | scheme % column (DTO `schPct` exists) | D (ui_table col) + Q | swagger PurchaseOrderLineDto |
| 26 | Col GST % | YES | server figure | | | po_04 |
| 27 | Col MRP (before Tax) | DIFFERENT | MRP is the **last** column and off-screen at 1560 px | position + fit | D (ui_table order / widths 105 %) | ui_table 57 rows; po_04 |
| 28 | Col Tax | YES | | | | po_04 |
| 29 | Col Amount | DIFFERENT | built has **Taxable** and **Total** (Amount = taxable? mockup 5,042.10 = 20x245 less disc, excl. nothing) | clarify; Taxable + Total both shown | Q/D | po_04 |
| 30 | Col Received (callout 2, derived) | YES | Recd, from /get `receivedQty` | heading "Recd" vs "Received" | D | po_04 |
| 31 | Col **Short-closed** (per line) | NO | none | column; /get lines carry `poiShortClosedQty` | D (ui_table col) + Q | server purchase-order.ts l.~376 |
| 32 | Col Pending (derived, orange when >0) | PARTLY | value shown, no colour | colour rule (0 green, >0 orange) | Q | po_04 |
| 33 | Col **Last rate** | NO | none | last purchase rate | D (col) + Q; route exists | `/purchase/lookup/item-context` (swagger) |
| 34 | Col **Stock** | NO | none | on-hand | D + Q; route exists | same |
| 35 | Col **Reorder** (callout 3) | NO | none | reorder qty per item | D + Q; value from `/purchase/reorder/suggestions` or item policy | swagger |
| 36 | Col **Status** chips DONE / PART / SHORT CLOSED | NO | none | in-cell chips (use DelegatePaint::background, see memory delegate-chip-background) | D (col) + Q | mockup rows 1-5 |
| 37 | In-cell: ordered qty "360" red frame = current cell; greyed engine values | DIFFERENT | current-cell border comes from the theme; no greyed derived cells | derived cells are not visually greyed | Q (theme/delegate) | po_04 |
| 38 | Blank-row prompt "type item / scan · or Reorder suggestions (Ctrl+R) · + / − lines" | NO | empty blank row, empty hint | prompt text | Q | po_01 |
| 39 | Totals strip: Lines / Ordered value (incl. tax) / Received value / Pending value / Short-closed / PO total (incl. est. charges) | DIFFERENT | one rich-text line "Taxable · GST · **PO value**" (+ "Short-closed: reason", "Revision n", "Approved by") | six-cell strip; Received value, Pending value, Lines, Short-closed lines are computable from the grid | Q | po_04; paintCalc / applyLoaded |
| 40 | Estimated charges (in PO total, callout "THE LINES") | NO | none | charges grid, `charges[]` in the DTO, total incl. charges | S + Q (column `po_charges_amt` exists) | swagger SavePurchaseOrderDto |
| 41 | Button **Reorder suggestions · Ctrl+R** (callout 3) | NO | Reorder is a separate menu 296 (`purchase/reorder/`) | button + Ctrl+R opening it / filling this PO | Q | grep: no Ctrl+R in order/*.cpp |
| 42 | Button **Clone / merge pending** | PARTLY | "Clone" and "Merge…" separate; merge popup lists pending POs (po_06); clone works | one button / label; keys | Q | po_04, po_06 |
| 43 | Button **Short close…** (per line or whole PO, with reason) | PARTLY | "Short-close" whole PO, asks reason | per-line (server takes `lineIds`), "…" | Q | cpp shortClose; swagger |
| 44 | Button **Send (PDF / WhatsApp)** | NO | Print · Ctrl+P (PDF via print dialog) from PurchaseDocNav | Send / WhatsApp | Q (+ S if WhatsApp needs a route) | po_04 |
| 45 | Button **Make GRN · Ctrl+G** (callout 4) | NO | none | button that opens a GRN against this PO | Q | no Ctrl+G |
| 46 | TxnKeys: New Ctrl+N, Save Draft Ctrl+S, Edit F2, Cancel Alt+X, History Ctrl+H, Close Esc | YES | all present, "Cancel PO · Alt+X" | | | po_04 |
| 47 | **Post · Ctrl+Enter** | DIFFERENT | button reads "Confirm" with no key shown (Ctrl+Enter is bound) | label "Post · Ctrl+Enter" | Q | po_04 |
| 48 | **Show List · Ctrl+O** | NO | absent; "Print · Ctrl+P" sits in its place | Show List button / key | Q | po_04 |
| 49 | Callout 1 "Amend = revision" (free until received, then qty up / pending-rate only; each save a revision; History shows rev 1 → 2) | YES | Amend button, F2 on a confirmed PO, reason asked, qty floor, Versions… dialog | none (extra buttons Amend / Versions… are beyond the mockup) | — | po_05, po_07, po_09; tests/purchase/test_purchase_po_doc01.cpp |
| 50 | Callout 2 "Received / pending derived, short close per line or whole PO" | PARTLY | derived (no counter); whole-PO short close | per-line short close, 3.0 config 16 "close PO on first bill" is server (WS3) | Q | po_04 |
| 51 | Callout 3 "Reorder → PO; clone / merge" | PARTLY | clone, merge built; reorder screen exists but is not linked | link (row 41) | Q | merge dialog po_06 |
| 52 | Callout 4 "Where it is used": GRN / bill offer this PO's pending lines | YES | GRN "Against PO" combo; Purchase Entry "From PO" | — | — | grn_02 |
| 53 | Callout 5 "Terms, credit, approval" | PARTLY | approval flow built (Approve / Not approve, AWAITING state) | terms + credit days not visible (rows 13-15) | Q / D | po_01 |

## Server routes checked on :3011 (swagger /api/docs-json)
Present: `/purchase/orders/{save,validate,get,pending-lines,cancel,short-close,clone,merge,approve,revisions}`, `/purchase/reorder/{suggestions,create-pos}`, `/purchase/lookup/{item-context,supplier-context}`.
`PurchaseOrderHeaderDto`: poId, companyId, branchId, accYear, deviceId, supplierId, docDate, expectedDate, godownId, **creditDays**, contact, **termsText**, remarks, post, amendReason. No agentId, no charges.
`PurchaseOrderLineDto`: lineNo, itemId, uomId, qty, freeQty, rate, discPct, **schPct**, taxId, mrp, weight, remarks, lineId.
DB `purchase.pur_order` has po_agent_id, po_credit_days, po_terms_text, po_approval_status, po_charges_amt. No terms master table.

## Agent-file feedback (qt-tester)
- Add: "when the plan has a mockup, read it and snapshot before you write checks" and keep a ready-made snap harness (`tests/purchase/audit_snap_po_grn.cpp` now exists; copy it). The runner's `ninja` build takes ~3 min on first run: tell the tester to start it in the background.
- Remind: **always install the modal closer in a snapshot program** (the first run hung 5 min on the "Confirmed" message box).
- Test items: item_master has service / inactive items; use the ZT items or check `item_is_service`/active before picking test items.
