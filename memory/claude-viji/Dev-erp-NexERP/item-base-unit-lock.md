---
name: item-base-unit-lock
description: "Item card base unit — follows the last row on a new item, LOCKED once saved (no stock check: offline tills); factors stay editable"
metadata:
  node_type: memory
  type: project
  originSessionId: dec6eebb-d475-4e27-9545-17a8221ccd6b
  modified: 2026-10-22T04:32:43.456Z
---

Decided 2026-10-22 (user asked "block change base unit … any stock present … but offline sync how to achieve"). Built in ItemEntry (menu 34 item card), tests/itemcard (SRC=test_item_base_unit 28 checks, SRC=test_item_enter_nav 11).

- **New item:** the base tick follows the LAST unit row automatically (units run biggest first, base last). Picking a unit on the placeholder moves the base to it; removing the base row moves it up. The default tick is left alone.
- **Saved item:** base LOCKED — tick not clickable (ItemIsUserCheckable off on every row), base row's unit can't be re-picked, base row can't be removed, validateForm refuses a base unit != m_loadedBaseUnitId. A new unit is inserted just ABOVE the base row, factor blank to type (at row 0 → factor 1 and the base row takes the factor).
- **Factors stay editable**: every txn line stores its own `*_to_base_factor` (sbi/sdi/sri/soi/sqi) and stock_ledger stores `sml_base_qty`, so a factor change only affects new bills. The base unit is what past base_qty is counted in.

**Why not "block only when stock exists":** the server can't see an offline till's unsynced sales, so the check passes and the sync later lands base_qty in the wrong unit. Same reasoning as [[offline-sync-invariants]]. "Saved once = locked" depends only on the item record.

**How to apply:** wrong base unit → make the item inactive and create a new item. A HO-only "change base unit" tool that converts history was mentioned as possible later, not built. A saved item with NO unit rows (old data) is not locked (baseUnitLocked needs a stored base). Legacy items whose stored base isn't last keep it where stored (m_loadedBaseWasLast).

**Server ask:** Shared/Prathap/inventory/notes-138-item-base-unit-lock.md (2026-10-22). A1 refuse base change on update (409 ITEM_BASE_UNIT_LOCKED; today items-master.service.ts:980-982 + syncUnitConversions + /item-unit-conversions update write it through). A2 derive item_base_unit_id from the base row: live 322 of 364 items with a base row have header NULL (all 6 stock_ledger items among them). A3 confirm synced lines post with their own to_base_factor.

**2026-10-22 LIVE-VERIFIED (user: "ran 138"):** A1 all 5 routes 409 ITEM_BASE_UNIT_LOCKED (other row marked base, base row left out, item_base_unit_id differs, base row unit changed, /item-unit-conversions unmark), DB unchanged each time; factor change + new unit still 201. A2 backfill done: 0 NULL headers of 364. A3 no written reply yet. Check script: scratchpad n138/check138.py pattern — throwaway ZT-N138 item, send only collectJson keys (echoing GET iuc_* fields = 400 VALIDATION_WHITELIST), delete after.
