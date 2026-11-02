---
name: stock-small-masters-notes-159
description: Notes 159 (2026-10-26) — menus 293/294, /stock/ageing-slabs (menu 294, AGE_RIGHT_*), /stock-track-presets/save + /deactivate (menu 249, SPT_RIGHT_*), usedByItems/usedByGroups on the GET
metadata:
  type: project
---

Built and live on :3011 and :3000, 2026-10-26 (migration 20261026130000_stock_small_masters_notes_159; uncommitted when written).

- **Menus**: 293 Stock Reasons (verbs VIEW,CREATE,EDIT,DELETE,PRINT,EXPORT) and 294 Ageing Slabs (VIEW,EDIT), parent 34, positions 110/120; 249 Tracking Presets widened to VIEW,CREATE,EDIT,PRINT. Seeded in Menu_Master.sql too; VKPOS granted. The /stock/reasons routes are still NOT judged on 293 (notes 159 did not ask for that).
- **Ageing slabs** (`src/modules/stocks/ageing-slabs`): GET judged on view, save and revert on edit. Save = advisory lock, `SET CONSTRAINTS stock.ex_sag_overlap DEFERRED`, SOFT-delete every company row, then createMany. The ladder rules are the pure `ladderErrors()` and answer 422 `slabs.<i>.<field>`. The DTO checks types only, so range rules come back as 422s and never as a 400 from the pipe. GET = own active rows, else the shared rows, never a mix. No report reads the slabs yet, and there is no fn_sag_effective in the DB despite the fragment's comment.
- **Presets**: save = create (no sptId; code + name required, 422 STOCK_PRESET_FIELD_REQUIRED) or update; an omitted field keeps its stored value. valuationMethod is derived when omitted. The lock (409 STOCK_PRESET_IN_USE, one detail per field) covers the six flags, valuation AND code once any NOT-DELETED item/group uses the preset. Deactivate is refused only for ACTIVE users. The GET stays open to any login (the item screens feed off it) and lost its `@CacheTTL(1)`, so a save shows at once. Its payload is snake_case `spt_*` plus `spt_is_active`, `usedByItems`, `usedByGroups`.
- **Tests**: unit specs in both modules; `test/stock-small-masters-notes-159.e2e-spec.ts` (12 cases, cleans up by its own ids, --runInBand). The in-use lock is unit-only: no live item is repointed.

Related: [[menu-verbs-capability-column]], [[db-table-master-registry]], [[dev-box-runs-two-apis]]
