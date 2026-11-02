---
name: erp-sale-order-qt-parity
description: "Sale Order React screen brought up to the Qt one on 2026-10-02 (settings hook, F5 route, tender→save, print, price-level keys, quick edits, role gate, list policy) — uncommitted on dev; plan + status in ERP client plan/sale-order-qt-parity.md"
metadata:
  node_type: memory
  type: project
  originSessionId: e0290841-c622-4b94-bead-23b4bafdf3d4
  modified: 2026-10-02T07:41:18.821Z
---

2026-10-02: compared Qt `src/modules/sales/saleorder` (NexERP 09f8896) with React `features/sales/sale-order`
and implemented the gaps (plan + status: `~/Dev/erp/ERP client/plan/sale-order-qt-parity.md`). Uncommitted.

Key facts:
- The React screen now reads ~20 app settings through `sale-order.settings.ts` / `use-sale-order-settings.ts`
  (shape of `features/accounts/receipt/state/use-receipt-settings.ts`). Catalog table is
  `public.app_setting_def` (asd_key / asd_default_value). `sales.default_validity_days` default is 7 — the
  screen now uses it (was hard-coded 30); flagged as D6.
- `sales.tender_type` decides F5: Tender (dialog's Save/Save&Print saves at once via a `saveAfterTender`
  effect) vs plain Save. `PrintOptionsDialog` (purpose SALE_ORDER, `/print-render/preview`) is used after
  Save & Print and by F11 (reprint last saved) — the old "no print endpoint" toast was stale.
- `ItemGrid` (shared with quotation) gained optional `priceLevelCount` and `onStepPriceLevel`; when passed,
  Ctrl+± steps the row price level and Alt+± inserts/removes rows. Quotation unchanged.
- Shared `CustomerSnapshot` got optional `pin` (for `soCustPin`); `customerApplied` no longer flips Term to
  CREDIT — only forces CASH when `debit_allowed` is false (Qt rule).
- Validation additions: `chargeRoleGate` (FREIGHT/LOADING/UNLOADING computed but no charge row),
  `creditGate` skips when `customer.overdueBilling`, `freeItemTax` context.
- Tests: `sale-order.parity.test.ts` (14), `sale-order.settings.test.ts` (6); all client tests + tsc green.

**How to apply:** not manually verified against the dev server yet (tender-route save, print dialog,
settings changing defaults). If the Qt side changes its key map, swap the one constant in the view/grid.
Related: [[nexerp-qt-client-source]], [[erp-sale-bill-react-port]].
