---
name: erp-stocks-rebuild-from-qt
description: "2026-10-02 React features/stocks rebuilt from Qt inventory/stock — 4 screens (opening, physical, stock adjustment, change selling price), old code + store modules deleted; tsc/tests green, never run logged-in; uncommitted"
metadata:
  node_type: memory
  type: project
  originSessionId: e0290841-c622-4b94-bead-23b4bafdf3d4
  modified: 2026-10-02T10:34:31.224Z
---

User's request 2026-10-02: "don't compare — remove the old files in features/stocks and create the new files
from the Qt application." Done the same day, uncommitted (deletions are `git rm`-staged).

- New: `features/stocks/{opening-stock,physical-stock,stock-adjustment,selling-price}` (~29k lines, 363 tests),
  routes `/stock/opening-stock`, `/stock/physical-stock`, `/stock/stock-adjustment`, `/stock/change-selling-price`.
  Each screen keeps local state (no Redux slice) and its own `*.api.ts` (`baseApi.injectEndpoints`).
- Deleted: old `features/stocks/**` and `store/{slices,sagas,api}/{openingStock,physicalStock}*`.
  `StockLookupOption` moved to `store/api/lookupsApi.ts`; `formatAccountingYear` to quotation.utils.
- Registries added: grids 101/122/100/102/78, ui-tables 41/44, dropdowns stockReason 50, stockTrackPreset 49.
- Device id on stock vouchers comes from the login (`getUserInfo().deviceId`, FK to fixed.device_master), NOT the
  browser uuid that sale order uses.
- Change Selling Price follows ui-table 44's visibility; Barcode / Cost B.Tax / Base Rate A were hidden by hand
  2026-10-01 (Qt forces them visible) — barcode scan needs Barcode re-enabled.

**How to apply:** per-screen ported / not-ported / deliberate-deviation lists are in
`~/Dev/erp/ERP client/plan/stocks-rebuild-from-qt.md`. None of the screens has been run logged in against the
server yet — do that before claiming they work. Related: [[nexerp-qt-client-source]], [[erp-sale-order-qt-parity]].
