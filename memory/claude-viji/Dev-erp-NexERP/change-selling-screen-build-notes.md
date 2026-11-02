---
name: change-selling-screen-build-notes
description: "Pre-gathered build notes for the Change Selling Price bulk screen (menu 30): file set, registry, backend prerequisites, and 3 open decisions"
metadata: 
  node_type: memory
  type: project
  originSessionId: 28c2bb21-93af-4cd5-a729-6aea4637de01
  modified: 2026-09-05T05:30:55.197Z
---

Context gathered 2026-09-05 for coding form 5 (spec:
schema/stock/change_selling_screen_plan.md — see [[stock-schema-design]]).
Build paused by user; resume from here.

**File set (mirror `src/modules/inventory/master/item_qty_price`):**
`src/modules/inventory/bulk_price/change_selling_price_{entity.h,delegate.h/.cpp,entry.h/.cpp/.ui}`
— NexDialog subclass + .ui (Designer, promoted NexTable/NexDropdownSingle),
entity header with `namespace ...Cols { enum + formats() }`, delegate emitting
`cellEdited(row,col)` from `setModelData()` (createEditor returns nullptr for
non-editable cells — never item flags). Register `fm.reg<...>(30)` in
`src/core/navigation/form_inventory_menu.h` (**menu id 30 is free**, verified);
add sources + include dir to CMakeLists.txt.

**Backend prerequisites (Prathap), all missing as of 2026-09-05:**
1. the 3 endpoints (GET/POST /stock/price-bulk, GET /stock/price-buckets/:item);
2. a `ui_table_master` row — NexTable::loadLayout(table_id) pulls columns from
   the backend, ids are backend-issued (17/18/19/21/26-29 taken);
3. the `menu_master` row for id 30.

**House specifics that matter here:**
- API: `ApiClient::instance().get/post(endpoint, cb, errCb)` with
  `QPointer<Screen> self` guard in every lambda; parse with `NexJson::*`
  (API quotes numerics — strict toDouble() is a known past bug); errors via
  `ApiClient::handleError(err, self)`; messages via `AppMessage` — its
  `showChoice()` fits the below-cost block/ask/allow prompt.
- Filter combos: `NexDropdownSingle::setDropdownId()` — ids **17 group,
  18 brand, 19 section, 35 supplier** (exactly the four filters).
- NexTable: F12 already emits `lineStockRequested` (use for the bucket
  picker); `PopupConfig`/`NexPopupWindow` is the cell-picker mechanism;
  `setCellValue()` never reaches the delegate, so the 4-number sibling
  recompute needs no QSignalBlocker. Per-row dirty = `QSet<int>` on the
  screen, set only in the cellEdited slot (no house helper exists).
- Session: `AppSession::instance()` — companyId/branchId (pass as
  `toString(QUuid::WithoutBraces)`), `belowCostPriceSetting()`, `userType()`.
- Ctrl+Enter save must bind BOTH Key_Return and Key_Enter (item_qty_price
  pattern); F-key buttons get shortcuts in Designer, not code.

**Three open decisions for the planning session:**
1. **No HQ-vs-branch role exists in the client** (grepped: nothing). The
   spec's "All branches is an HQ-role action" scope switch needs a mechanism —
   likely `userType() == "SUPER ADMIN"` (the only existing role gate) or a new
   app setting via the /app-setting-values/effective layering.
2. **F5 conflict:** spec/legacy says F5 = Load Items, but house standard is
   F5 = Save (F8 = Show List). Decide with user.
3. Screen must fan the save payload: rows with a bucket dimension → new smp
   endpoint; untracked headline rows → existing item-prices endpoint.
