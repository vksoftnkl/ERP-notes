---
name: erp-qt-e0c56ef-purchase-port
description: "Port of Qt NexERP 737e671..e0c56ef (2026-10-29) — purchase phase 2: PI line + side features, F12/attach/labels shared dialogs, GRN textile/excess/Sell A–D, PO amend/merge/approval, Reorder → PO 296, Agents 28, Supplier–Item Map 295, MST-01/03/04"
metadata:
  node_type: memory
  type: project
  originSessionId: cb3140f7-8553-40bc-9c24-d78121d74377
  modified: 2026-10-29T08:19:17.453Z
---

Ported 2026-10-29 by seven parallel forks on a contract-first shared layer. Follows [[erp-qt-737e671-purchase-port]]. Committed 2026-10-30 as client 182af33 (party rules D1-7 separately as 8e63c05).

- **Shared contracts** (features/purchase/shared): `purchase-dialogs.types.ts` (PurchaseSettings + defaults, F12 `PurchaseAllocEntry`/`Need`/`allocDtoOf`, attach + labels props), `purchase.api.ts` `useGetPurchaseSettingsQuery` (GET /purchase/invoices/settings — needs View on menu 24, so a user without it gets no Sell A–D / labels-after-post), `purchase-handoff.ts` (Ctrl+F11 `?againstBill=` → Return page; Ctrl+F8 reuses lib/navigation/party-handoff, supplier id = ledger id). Dialogs: `components/purchase-{alloc,attach,labels}-dialog.tsx` + `usePurchaseAfterPost`. `PurchaseDocumentPage` got `initialView="list"` for hand-off arrivals.
- **Purchase Entry is split across files**: line work in `invoice-lines.ts` / `use-invoice-lines.ts`, header/side (charges, chips, hold, sister, nav, attach) in `use-invoice-side.tsx` + `invoice-charges.ts` + `invoice-hold.ts`; shared files carry hunks marked PI-HEADER / PI-LINES. `addSourceLines` is intercepted in the main reducer (drops off-bill charges for GRN lines) before `lineAction`.
- **Keys**: attachments = Ctrl+A on every purchase screen, EXCEPT inside a typing field (stays select-all, as in Qt) — `isTypingTarget` in grn/grn.keys.ts. Prev/Next = Alt+PgUp/PgDn via /txn/adjacent (five purchase docTypes added to `TxnAdjacentDocType`).
- **Deliberate differences from Qt**: an F12 split folds back onto the line only if every row is in the doc's own godown (Qt loses the godown); changing a line's item clears its PO/sister link, split and textile cells; PO has an extra "Stop amending" button (user may want it dropped).
- **Untested live**: PO approval (dev `po_approval_above` = 0), Sell A–D (price update off in dev), unknown-barcode→EAN save, Payment ON HOLD chip (no open supplier bills in dev), F10 resume of a held bill end to end (creates an undeletable draft).
- **Server**: all routes committed (through f2b509ed). Dev DB was MISSING migrations 20261029100000 (grid 195 scheme columns), …110000, …120000 at port time — reported, not applied. Config rows (ui_tables 62–64, grids 194/195, menus 28/295/296) are dev-DB rows → [[erp-live-config-drift]].
- Live test harnesses: that session's scratchpad `*-live/` dirs; test docs left CANCELLED (PUR 00365/66/68/69, PO 00180–00191, GRN 00255+, PRN 00128).
