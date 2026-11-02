---
name: opening-stock-txn-list
description: Opening stock (menu 44) on the shared TxnMainView descriptor, plus the two backend gaps the screen currently works around (grid 99 SQL, no per-line trackSignature on GET)
metadata:
  type: project
---

As of 2026-09-09 menu 44 opens `TxnMainView::openForMenu(TxnDocTypes::openingStock())`
(`src/modules/inventory/common/txn_doc_types_inventory.cpp`) instead of the retired
`OpeningStockList`, and OpeningStockEntry's "Show List - F8" opens the same descriptor
as a Pick-mode picker — the quotation pattern, see [[txn-main-view-architecture]].

**Pending backend step:** grid 99's stored `grid_sql` is still the 2026-09-08 version.
The updated statement is in `/home/vk/Shared/Prathap/grids/opening_stock_list_grid.sql`
(see [[db-schema-share-location]]) and adds `svh_company_id` / `svh_branch_id` to the
SELECT plus `isvh_from_date` / `isvh_to_date` predicates. Until it is re-POSTed to
`/grid-details/create` with grid_id 99, the list's From/To range is drawn but inert and
TxnDocKey arrives with no company/branch — `OpeningStockEntry::loadVoucher()` falls back
to the session for exactly that window.

**Why:** the `claude` DB role is read-only (see [[db-live-connection]]) and grid SQL is
only writable through the API, which needs a login this session had no credentials for.

**How to apply:** re-register the grid, then delete the session fallback in
`loadVoucher()` and the "not yet" notes in the descriptor and the .sql header.
Grid params on 99 are svh-prefixed and BARE (`isvh_company_id`, not `':icompany_id'`),
which is why the descriptor overrides `companyParam`/`branchParam`/`accYearParam`/
`fromDateParam`/`toDateParam` — and why the status filter sets the new
`TxnFilter::sendWhenEmpty`: a param the runner never substitutes leaves a bare
identifier standing in the statement and the query fails.


**Second backend gap — the line tracking signature.** `/stock/opening/get`'s
`OpeningStockLineDto` returns no `trackSignature`, so `applyLines()` could only guess
one from the values a line carried — and a draft saved before its batch numbers were
keyed guessed `"N"`, whereupon the delegate greyed the batch/expiry/MRP cells and the
draft could never be completed. Reported 2026-09-09 against "Sakthi Manjal Powder"
(Acme Foods, ITEM-scope policy `BMEP`, preset PHARMA — the DB was right all along).

Worked around by `OpeningStockEntry::refreshLineSignatures()`: one `/item-lookup` per
DISTINCT item, writing only TrackSignature + TrackedBy. It is called from the END OF
`applyLines()` itself, not from its callers — putting it in one caller missed the
repaint from the SAVE response, where the same guess greyed the batch cell that
`/validate` had just told the operator to fill in, so the document could neither be
fixed nor posted. Add `trackSignature` to `OpeningStockLineDto` and the whole path
collapses back into `applyLines()`.

Also note `stock.fn_stp_effective` does NOT exist on the live DB — the schema file
names it but the resolution is implemented in the NestJS layer, so policy questions
cannot be answered with psql alone.


**Third backend gap — supplier.** Supplier is a tracked facet (the P of "BMEP") and
identifies the LOT, but the screen had no way to enter one: `svi_supplier_id` was read
and sent, never filled. Built 2026-09-09 as a per-line popup picker on a new
`SupplierName` column (37), grid **100 "POPUP - SUPPLIERS"** — the family of 71 (items)
and 79 (customers).

Three things this turned up, all still owed:
- It cannot be a HEADER field: `stock_voucher.svh_supplier_id` exists and the GET
  returns it, but `SaveOpeningStockVoucherHeaderDto` has no supplier field at all.
- ~~No `supplierName`~~ — FIXED 2026-09-09: it is on `OpeningStockLineDto` and
  `OpeningStockHeaderDto` now (and the physical-stock / transfer DTOs), so a reopened
  document shows the name. `applyLines()` reads it.
- `/validate` (Q2 of 19q_opening_stock_queries.sql) checks batch/expiry/MRP/serial but
  NOT supplier or sale price, though both are tracked facets. So a BMEP line with no
  supplier posts and `fn_slt_resolve` normalises it to the zero-uuid sentinel — an
  unidentifiable lot, and unlike the missing-batch case nothing catches it.

Registration is owed too and CANNOT be done from psql: run
`/home/vk/Shared/Prathap/grids/apply_supplier_picker.sh <token>` (grid 100 +
ui_table 27's new column). Until it runs the Supplier column stays invisible —
`NexTable::loadLayout()` hides every column and only `applyLayout()` reveals the ones
the registration marks visible.


**Still open as of 2026-09-09 (re-checked against the live spec):**
- `trackSignature` is on `OpeningStockItemLookupDto` ONLY, not on the line DTO — so
  `refreshLineSignatures()`'s round trip per distinct item stays necessary.
- `SaveOpeningStockVoucherHeaderDto` still has no supplier field, so the supplier
  remains per-line. (The GET's header now returns supplierId + supplierName, but the
  screen deliberately does not surface them: a header supplier could be read and never
  written.)
- The cancel route's spec still reads "Cancel a POSTED opening" and `fn_svh_cancel`
  still raises on non-POSTED, though the client now offers cancel on a DRAFT too on the
  user's word that the route handles both. Verify with one draft cancel.


**Closing test (2026-09-09).** Plan + automated verifier live on the share:
`schema/stock/19t_opening_stock_test_plan.md` (driven off all 8 `stock.stock_track_preset`
rows) and `schema/stock/19t_verify_opening.sql` (run per refno after every step; prints
PASS/SKIP/FAIL per invariant — ledger pairing, lot identity normalisation, balance vs
ledger, reversal netting, which are invisible in the UI). NOT YET RUN — see the plan's
blocker table: the `Cost / Unit` registration is unapplied (so no cost can be keyed at
all), 4 of 8 presets have no sample item, and draft-cancel depends on the route not
delegating to `fn_svh_cancel`.

**LIVE-VERIFIED 2026-09-30 — the open items above are CLOSED:** grid 99 now selects company/branch and binds the date range; draft cancel works (it marks the draft CANCELLED); Cost / Unit is registered (ui_table 27 = 39 columns, no mismatch). The post writes a STOCK/OPENING accounts voucher and the cancel reverses it. Suite: `tests/openingstock/run-live.sh`, 26/26. Posts go to **Trichy**, because ZT-ITEM-A is already opened in Coimbatore. Still open: notes 66 (a) "already opened" ignores the bucket — a second opening of the same lot in DAMAGED is refused; (b) no trackSignature on /get lines.

**Notes 66 FIXED 2026-09-30:** "already opened" is per bucket now, and /get lines carry trackSignature. refreshLineSignatures() and applySignatureToItem() are DELETED; applyLines reads the signature straight off the line.

**Save & Post is ONE request (2026-10-01, the user's rule).** If Save & Post fails validation, nothing is saved: show the warning and return. `saveAndPost()` sends `/stock/opening/create` with `header.status = "POSTED"`. The server saves, runs the per-line preflight and posts in one transaction, so a 422 rolls the whole thing back with no stray draft. The client first refuses missing batch/expiry/MRP/serial with no round trip. The 422's `errors[].message` lines appear in one "Nothing was saved" dialog. The old save → /validate → /post chain (postDocument/showLineProblems) is gone. Save Draft is unchanged. run-live.sh now runs 30/30, including L5 (an already-opened holding is refused and grid 99 shows no draft). The same `status: POSTED` switch exists on the shared stock-voucher save (adjustment, physical), if they ever need the same rule.

**2026-10-02 relayout (user ask, "elegant like change selling"):** .ui is now a QVBoxLayout root (no gridLayoutWidget/setLayout): title + lblScope subtitle, status PILL (tones draft/posted/cancelled), header in a `card` frame, totals strip `frameTotals` (QLabels, not NexDoubleEdits), lblHint, `frameButtons`. btnQuickAdd/RemoveLine and frameQuick/grpNotes/grpTotals are GONE (+/- on the grid); Split batch = `btnSplitLine` in the button bar. **Ctrl+Enter = Save & Post** (asks first), F5 = Save Draft, F6 dropped. Focus: `focusEntryStart()` (godown when blank, else first item-less line) on first show / New / Edit / next document; load puts the grid on (0, Item). Tab stops: date, godown, rate, ref, remarks, grid. Tracked by is PAINTED as chips (delegate paintTrackedBy, theme tokens, tooltip = words); ui_table 27 has it at 5% so chips fall back to letters — widening was offered, not done. Styles = template **v11** (v8 + new OpeningStockEntry block), published 2026-10-02, md5 06eb30b6…; share/theme/seed-template-v11.qss. run-live.sh NOT re-run (blocked by the permission classifier) — the user must run it.
