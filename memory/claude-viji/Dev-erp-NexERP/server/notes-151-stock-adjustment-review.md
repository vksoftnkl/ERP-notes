---
name: notes-151-stock-adjustment-review
description: "Notes 151 (2026-10-24) stock adjustment (menu 264) server review — all built: ADJ_RIGHT_* menu-264 rights, cost rebuild takes the last LIVE row (all-cancelled item reads 0), StockPostHooks beforePost/afterPost run the adjustment rules inside the post tx (save+post too), re-lot same-bucket rule, lot company check (adjustment + shared preflight), pick-stock batchNo + size/colour/model, /get detail fallback join, by-batch avgCostRate; ZT-ITEM-A repaired on .106"
metadata:
  type: project
---

Built 2026-10-24 from Viji's notes 151 (the adjustment counterpart of notes 143 / 150). Shared code
changed, so several items touched EVERY stock voucher type:
- **#1** StockAdjustmentController.requireRight → assertMenuRight menu 264, codePrefix ADJ
  (ADJ_RIGHT_VIEW/CREATE/EDIT/POST/CANCEL/DELETE; save+post needs post too). Body company/branch
  NOT compared with the token — same stance as notes 150 #7 ([[token-scope-is-home-company-and-device-branch]]).
  Reasons master routes untouched. test/helpers/menu-rights has no constant for 264 yet (no HTTP e2e).
- **#2** rebuildItemCosts + rebuildLotCosts: the qty≤0 fallback rate is the latest LIVE row
  (NOT sml_is_reversal AND no reversal row), so a cancel leaves the average alone. An item whose
  every movement is cancelled reads avg 0 = "not stated" (next inward needs MANUAL cost or is a
  fresh start). ZT-ITEM-A on .106 had ZERO live rows, so the repair (rebuildStockDerivedFigures
  over itemId) set it to 0/0, not the 50 the review expected — the 50 was itself a cancelled trace.
- **#3/#4** already covered by notes 150 #5/#3 (server totals, login author, docDatetime bound).
- **#5** `StockPostHooks { beforePost(tx, svhId); afterPost(tx, rowsPosted, svhId) }` in
  stock-voucher.types; StockVoucherService.post() 7th arg takes the object OR the old bare
  afterPost fn; save() takes hooks as 3rd arg (save+post path). Adjustment `postHooks()`:
  beforePost = `assertRulesHold` (kindOf re-read under the lock → 409 STOCK_ADJ_KIND_CHANGED if
  the pinned rules differ, then check() over stored rows on `tx`), afterPost = re-lot guard decided
  from THAT re-read. post() no longer checks before the tx. All adjustment reads take a `db` param.
- **#6** STOCK_ADJ_RELOT_BUCKET_MISMATCH: all RELOT_IN lines of an item must sit in a bucket some
  RELOT_OUT leaves (a bucket change is a Move stock).
- **#7** LotRow carries slt_company_id → LOT_OTHER_COMPANY in check(); the SHARED preflight CASE
  got three named-lot branches (missing / other company / other item) via `LEFT JOIN stock_lot nl`.
- **#8** pick-stock joins stock_batch_detail (company+item+slt_key_batch) → sizeId/Name,
  colourId/Name, model. /get's sbd join is now a LATERAL: the line's own row (retired too) first,
  else the batch's live row by company+item+folded batch — an adjustment/count line that only names
  a batch reads it.
- **#9** PickStockQueryDto.batchNo = exact `slt_key_batch = upper(btrim(?))`; two more binds before
  the search binds (spec slices updated).
- **#10** by-batch: godowns[].avgCostRate (sbl_avg_cost_rate) + lot-level avgCostRate (Σvalue/Σqty
  else slt_cost_rate). The `held.on_hand, 0) > 0` text is grepped by its spec — the CASE avoids it.

**Why:** Viji's live review on .106; the cancelled-cost average and the pre-lock rule check were
the real bugs, the rest hardening + textile screen needs.
**How to apply:** e2e stock-adjustment is 23 cases (19–23 = notes 151; case 16 now asserts
server-summed totals); physical/opening/stock-engine-ts/textile e2e green after the shared changes.
A future type-specific rule that must see the locked rows goes in `beforePost`, never before the
shared post. See [[notes-150-physical-stock-review]], [[notes-143-opening-backend-review]].
