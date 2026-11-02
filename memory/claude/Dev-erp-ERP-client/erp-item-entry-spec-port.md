---
name: erp-item-entry-spec-port
description: "Item Master form implements the legacy Qt ItemEntry spec (July 2026) — deliberate adaptations, config-constant stubs, and the price-grid change-detection gotcha"
metadata: 
  node_type: memory
  type: project
  originSessionId: 92dc5e6a-c365-4b94-b1b2-7817f9611930
  modified: 2026-07-20T04:58:19.177Z
---

The Item Master form (features/masters/inventory/item/) was aligned to the legacy Qt ItemEntry functional spec on 2026-07-20. Key facts for future work:

**Deliberate adaptations (do not "fix" back to spec):**
- Price rows carry **item-level** company/branch (`ipm_company_id`/`ipm_branch_id` from the item), not per-row pickers; uniqueness is unit-only (one row per unit + godown column). The server accepts per-row values but the port's ladder model keys rows by unit.
- Reorder-type options keep the extended lowercase set (purchase/production/repack/transfer) — server stores free strings; existing rows use them.
- EAN grid keeps its editable Default/Active columns (richer than spec); save falls back to first-row-default when none is ticked.
- Legacy AppSession toggles (`skipMrp`, `belowCostPriceSetting`, `hsnMinLength`) have **no server source** — they're constants in item-master-page.constants.ts with legacy defaults (false/"warning"/0), waiting for a real config endpoint. Tier count comes from `/price-level-masters/get` (levels 1–4 relabel A–D columns via `priceLvlShort`).

**Gotchas found the hard way:**
- `detectChangedItemPriceField` must diff the **raw incoming rows** (before `syncItemPriceRowsWithUnitConversions` rewrites factors/slno) and tolerate the auto-appended placeholder row (compare common prefix) — otherwise unit-pick/cost edits mis-dispatch.
- `normalizeItemPriceRowForEditor` must NOT force row-0 factors to 1 (old row-0-is-base model); it fights the UC-factor mirror and creates phantom factor diffs every pass.
- Below-cost "Save anyway?" confirm lives in `buildRequestPayload` (throwing there aborts the save silently, modal stays open); hard blocks (restrict/MRP/min) live in the sync validator because validators also run on change. See [[erp-modal-onvaluechange-double-run]] for the dialog double-run gate.
- Local dev DB's ui-table config (uiTableId 3) hides some price columns (e.g. "Sale A") — missing columns in the browser are config, not code.
