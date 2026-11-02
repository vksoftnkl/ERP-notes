---
name: erp-qt-4ae6575-port
description: "Port of Qt NexERP 3b5069d..4ae6575 (2026-10-23) — item card/list, Item Qty Price redesign, Change Selling Price scheduled prices (notes 140), small masters (notes 139), bank-account popup, date ↑/↓; server notes 139/140 were only STAGED at port time"
metadata:
  node_type: memory
  type: project
  originSessionId: 32403bc8-74f1-4aa5-a593-5ee4c0d78037
  modified: 2026-10-23T06:04:52.378Z
---

Ported 2026-10-23 by five parallel agents on a shared layer written first. Follows [[erp-qt-3b5069d-port]].
Not committed. Mid-task the WHOLE port (78 files) got staged at once, with both peer sessions idle and
neither on the client — possibly the user's own VS Code "stage all", so I left the index alone
(contrast [[erp-workspace-layout]], where a peer did it).

- **Shared layer**: `features/stocks/selling-price` gained `listPriceSchedule`/`cancelPriceSchedule`,
  `effectiveFrom` on save + `rows[]` in its answer, `priceLvlIsActive` on level names, and
  `components/scheduled-prices-dialog.tsx` (`ScheduledPricesDialog` for CSP F9, `ScheduledPricesLauncher`
  for the Item list's Scheduled button). `toast.apiError(err, { refusedTitle })` titles a non-save
  refusal ("Cannot delete"/"Cannot restore"; Qt `handleError` 3rd arg).
- **Server state at port time**: notes 139/140 sat STAGED, uncommitted, in the server repo. Until they
  ship, F9 404s, a scheduled save 400s (`effectiveFrom` is an undeclared key → forbidNonWhitelisted), the
  item card never sees `scheduled_prices`, qty-price overlap/value/unit rules are client-only.
- **Open decision (asked user)**: on CSP a NEW price row scheduled for a later day stays "changed" after
  the save (no row in force today to re-read it from; the date resets to today), so a second Save
  inserts it for TODAY. Qt behaves the same.
- **Deliberate gaps**: item card expiry checks absent on purpose (cb8fe0a added, 54b0eed removed);
  bank account type omits "Fixed Deposit" (server enum SAVINGS/CURRENT/CASH_CREDIT/OVERDRAFT); date ↑/↓
  only on TEXT date boxes (`lib/date-step.ts`: quotation `DateField`, expense `DateInput`) — native
  `type="date"` and the dynamic modal's date fields untouched (browser steps segments; arrow-nav modals
  use ↑/↓ for fields); Qt `MenuWidgetSetting` = the right-click Visible Settings tree
  (`features/masters/shared/widget-visibility-tree.tsx`), not `settings/widget-master`.
- **Item card**: base-unit lock + insert-above-base, Default Barcode field IS the default EAN row (grid
  Default column removed), Copy from Item (new only), saved rows refuse removal on all four grids, price
  grid profit % on cost w/o tax, bucket lock (notes 140 D), level bands; rules in `item-card-rules.ts`,
  `item-price-grid.utils.ts`. Qty price button → `itemQtyPriceHref` ([[erp-item-qty-price-page]]).
- `use-soft-delete-actions.ts` now has opt-in `plainRefusals` + optional `restoreEndpoint`; `useApi`
  throws the raw axios error, so `toApiErrorShape` (server-error-text.ts) adapts it for `apiRefusalOf`.
