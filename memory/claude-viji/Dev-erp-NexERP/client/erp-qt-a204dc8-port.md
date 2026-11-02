---
name: erp-qt-a204dc8-port
description: "Port of Qt NexERP 4ae6575..a204dc8 (2026-10-24) — shared TxnKeys, Opening Stock rework, Price Change Log 290, Ctrl+R quick rate, Size/Colour masters 291/292; web key deviations and the ModalShell Escape fix"
metadata:
  node_type: memory
  type: project
  originSessionId: 87df7b8b-02ed-4890-b721-38d98db068b1
  modified: 2026-10-24T10:52:58.242Z
---

Ported 2026-10-24 by three agents on a shared layer written first; Size/Colour by the main
session. Not committed. Follows [[erp-qt-4ae6575-port]]. All server routes were already live
(notes 142–147), so nothing waits on the server.

- **Shared keys** `lib/txn-keys.ts` + `components/txn/use-txn-keys.ts` + `txn-keys-sheet.tsx` (F1).
  Web deviates from Qt where Chrome RESERVES the chord (no preventDefault reaches the page):
  New = Alt+N (Qt Ctrl+N), Previous/Next = Alt+PgUp/Alt+PgDn (Qt Ctrl+PgUp/PgDn); the Qt
  chords are still matched. Only Opening Stock adopted the whole set (as in Qt); Ctrl+R
  (change rate) is on Quotation, Sale Order (menu 11), Sale Bill, Testbill and the Item list.
- **`/txn/adjacent`** is `components/txn/txn-adjacent.ts` (GET mutation, every docType); only
  Opening Stock calls it so far.
- **ModalShell Escape is now per event**: only the topmost `aria-modal` surface closes, and an
  Escape already `preventDefault`-ed is left alone — stacked ModalShells used to all close on
  one Escape. See [[erp-nested-dialog-escape-race]]. Price Change Log also carries its own
  `components/dialog-keys.ts` from before the fix.
- **GridCell `dropBufferOnKeys`** (opt-in): Opening Stock's Barcode ↓ hands the typed text to the
  item list without committing it as a barcode.
- **Size/Colour** = one component, `features/masters/inventory/item-attribute/`. Grids 181/182
  bind a bare company token as well as the deleted flag — both always sent. Dropdowns 70/71
  are registered but unused (the purchase line is not ported). The app-wide capitalization
  rewrites names ("extra large" → "Extra Large").
- **Quick rate** saves only with a GRANTED Edit on menu 30 (`status === "granted"`, not the
  fail-open default). No Delivery Challan screen on the web, so no Ctrl+R there.
- **Opening Stock**: Ctrl+H = `TxnHistoryDialog` status trail only (Qt shows a per-revision
  panel). Dev ui_table 27 has columns 39/41/43 (Size/Colour/Model) that neither client
  defines — dropped by the column join; they arrive with the textile auto-batch work.
