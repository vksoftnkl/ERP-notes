---
name: erp-qt-74b755c-port
description: "Port of Qt NexERP a204dc8..74b755c (2026-10-26) — textile batch detail (size/colour/model) on Opening Stock, Physical Stock + Stock Adjustment moved to TxnKeys/top band/batch-label scan; Enter-commit double-fire trap"
metadata:
  node_type: memory
  type: project
  originSessionId: 4b0e677e-7ec5-4b05-9b34-4efab700d934
  modified: 2026-10-26T05:21:31.958Z
---

Ported 2026-10-26 by three parallel agents (one per screen) on a shared layer written first:
`features/stocks/shared/batch-detail.ts` (`batchDetailItemText` = "Shirt · XL · Maroon · Slim fit",
`lotsByBatchParams`, `AUTO_BATCH_NO_SETTING_KEY`). Follows [[erp-qt-a204dc8-port]]. Not committed.
All server routes were already committed (server af922b84, notes 149–152).

- **`stock.auto_batch_no`** gates everything textile; each screen parses it in its own `*.settings.ts`.
  Dev DB: ON for company Amazon Retail (019c8ea6…); batch **CBE26-00008** (Texttiles Items, XL/BLACK/DIWALI)
  sits in godown Coimbatore as SALEABLE 70 + EXPIRED 10 — a ready-made scan target.
- **Opening Stock**: Size/Colour/Model = ui_table 27 cols 39/41/43 (40/42/44 are line fields `sizeId`,
  `colourId`, `batchTyped`), shown only while the setting is on. `<datalist>` cells (`batch-detail-cell.tsx`),
  quick-add via `/item-sizes|/item-colours/create`. Deviation: a batch counts as "typed" only when its text
  CHANGED (Qt wipes size/colour on any commit). Live may lack ui_table rows 39/41/43 — the join inserts
  them after Batch No anyway.
- **Physical Stock / Stock Adjustment**: shared TxnKeys, Opening-Stock top band, ↓ in Barcode = item list,
  Prev/Next via `/txn/adjacent`. Adjustment: Pick from stock moved **F2 → F12** (F2 = Edit). Physical shows
  the book-moved warning on Post too (Qt: draft save only). Adjustment's pick dialog: first Enter in the
  search box drops to the list, the second picks (pre-existing design).
- **Trap fixed in `batch-detail-cell.tsx`**: Enter commits, then `moveCellFocus` focuses the next cell
  synchronously, so the old cell's blur runs with the STALE `buffer` closure and commits again. `GridCell`
  has the same pattern (harmless for idempotent writes); a commit with side effects (a confirm, a lookup)
  must read the buffer through a ref.
- Headless typing (`Input.insertText`, no Shift) gets title-cased by [[erp-global-uppercase-inputs]]
  ("CBE26-00008" → "Cbe26-00008"); real scanners hold Shift, and the server upper-cases batch numbers.
