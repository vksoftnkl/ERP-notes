---
name: notes-150-physical-stock-review
description: "Notes 150 (2026-10-24) physical stock server review — all built: dup-holding 422, one freeze per godown 409, docDatetime bound (syncDate declares offline), bookChanges on re-save, server totals + login author (ALL stock vouchers), menu-45 rights, superseded-cancel 409, count-sheet size/colour/model, by-batch bucket/godownId, 12-hour clock text"
metadata:
  type: project
---

Built 2026-10-24 from Viji's notes 150 (the physical-count counterpart of notes 143). Code is in
StockVoucherService (shared), so several items changed EVERY stock voucher type:
- **#1** assertPayloadRules refuses two COUNT lines of one holding (lot|godown|bucket) → 422
  STOCK_COUNT_HOLDING_DUPLICATE. Same lot in two buckets is allowed.
- **#2** save() under the row lock: a freeze on a godown another DRAFT PHYSICAL already holds over an
  OVERLAPPING window → 409 STOCK_FREEZE_GODOWN_HELD (`assertFreezeNotHeld`). Non-overlapping ok.
- **#3** (user asked "offline session or a cap"; the server has NO offline-device record — token has no
  tenant, till only logs late arrivals) → docDatetime without syncDate must be within 15 min of now();
  with syncDate: docDatetime ≤ syncDate ≤ now (+15 min skew) and syncDate − docDatetime ≤ 7 days →
  422 STOCK_DOC_DATETIME_OUT_OF_RANGE. Both fields are the client's: the claim is explicit + stored.
  test/stock-engine-ts case 6 (b′) now sends syncDate.
- **#4** save response: `bookChanges[]` + per-line `bookChanged` / `previousBookQty` (COUNT re-save
  re-reads svi_book_qty). Types: StockVoucherSaveResult.lines are StockVoucherSavedLine.
- **#5** header totals are SERVER-computed on every save for ALL types (`documentTotals`: COUNT = net
  variance at sbl_avg_cost_rate(_wot); adjustment family signed by svi_direction; else magnitude,
  value = round((base+free)×rate,2) like the generated svi_value); payload's four fields accepted +
  ignored (DTO docs say so). `actorFor`: a LOGIN overrides body createdBy/modifiedBy on header AND
  lines; body text stands only when no user is in context (resolveActor → DEFAULT_ACTOR).
- **#6** PhysicalStockVoucherController.requireRight → assertMenuRight menu 45, codePrefix PHY
  (PHY_RIGHT_VIEW/CREATE/EDIT/POST/CANCEL); save+post needs post too. No SUPER ADMIN bypass. The four
  physical HTTP e2e specs grant/restore via test/helpers/menu-rights PHYSICAL_STOCK_MENU.
- **#7** NOT changed, by design: company/branch ≠ token is deliberate ([[token-scope-is-home-company-and-device-branch]]);
  tenant has no server-side source (not on company_master, not in the token) so svh_tenant_id still
  comes from the body — one-line change once a source exists.
- **#8** cancel() of a POSTED COUNT: any holding of it with a later POSTED PHYSICAL (by the POSTED
  txn_status_log step, ledger-row or not) → 409 STOCK_COUNT_SUPERSEDED (`assertCountNotSuperseded`).
  test/physical-stock.e2e afterEach now cancels newest-first because of this.
- **#9** count-sheet LEFT JOINs stock_batch_detail (company+item+upper(btrim(batch))) → sizeId/Name,
  colourId/Name, model. accYear on count-sheet is optional and unused.
- **#10** /stock/lots/by-batch: `bucket` (SALEABLE default | any | ALL) + `godownId`; each godowns[]
  row carries `bucket`. Param order changed ($7 is the batch) — spec updated.
- **Minors:** `src/common/utils/ist-clock.util.ts` (istClockText 'DD-MM-YYYY hh:mm AM' IST +
  IST_CLOCK_SQL_FORMAT for to_char); validate's frozen message and assertNotFrozen (now a coded 409
  STOCK_GODOWN_FROZEN via throwStockConflict) use it.

**Why:** Viji's live review on .106; HIGH items were duplicate variances and mutually blocking freezes.
**How to apply:** the totals/author rules are shared — an opening or adjustment spec that asserts
body totals or body createdBy is now wrong, not the code. Unit: stocks+common 94 suites green;
e2e physical-stock / opening-stock / stock-engine-ts / textile green. See
[[pre-existing-test-failures]] for the two physical HTTP suites that fail on dev DATA.
