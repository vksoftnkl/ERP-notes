---
name: erp-qt-737e671-purchase-port
description: "Port of Qt NexERP 75c682b..737e671 (2026-10-28) — the Purchase module: Purchase Entry 24, PO 23, GRN 196, Purchase Return 25 + claims, Shortage Receipt 195 on a shared list/route layer; Godown Transfer short refno; WhatsApp needed nothing"
metadata:
  node_type: memory
  type: project
  originSessionId: 28a9a568-4cc0-4027-8e2b-bb58759c735a
  modified: 2026-10-28T11:33:03.795Z
---

Ported 2026-10-28 by five parallel forks (one per screen) on a shared layer written first. Follows
[[erp-qt-75c682b-port]]. Not committed.

- **Layout**: `features/purchase/shared/` = `PURCHASE_DOC_TYPES` (the Qt `TxnDocTypes::purchase*`: grid key, SQL
  field names, cancel route + idKey, row-policy words, memoryKey), ONE `PurchaseListView` for grids 189–193 (all bind
  grid 183's six quoted tokens), `PurchaseDocumentPage` (list ↔ entry, `system.txn_entry_first`, screen-memory
  resume), `usePurchaseScope`, format helpers. Each document = `features/purchase/<invoice|order|grn|return|shortage>/`
  with its own reducer + `<doc>.api.ts` + entry view taking `PurchaseEntryViewProps`. Routes `/purchase/purchase-entry`,
  `/purchase-order`, `/grn`, `/purchase-return`, `/shortage-receipt` (MENU_ROUTES 24/23/196/25/195).
- **Lists never delete** — Qt sets no `onDelete`, the server has no purchase delete route; F3 Cancel posts
  `{<idKey>, accYear, reason}`. Consequence: a GRN DRAFT can never be removed (cancel takes POSTED only) and it
  holds the PO's pending qty — dev DB had 21 stray GRN drafts before the port.
- **ui_tables 56 / 58 grew past Qt's cols.h** (Sell A–D, Size/Colour, Markup…, later server work): the screens map
  columns BY NUMBER and draw only the Qt-known ones (PI 0–18, GRN 0–14). Adding those features = extend the column maps.
- **Server get gaps**: GRN and PO `get` return no godown name (screens look it up via the GODOWNS dropdown);
  PI `get` has `godownName` only from server 638f45cf.
- **WhatsApp (Qt 737e671)** = desktop click-to-chat + PDF on the OS clipboard; the client's
  [[erp-sale-bill-whatsapp-send]] already resolves the number the same way and adds a share link — nothing to port.
- **Godown Transfer (Qt 3ce8487)**: `shortRefno()` in stock-transfer.state (old uuid-numbered refnos → first block;
  device-code numbers pass through), audit on one line, shape-note strip gone, Pick from stock in the footer.
- A fork claimed `/purchase/shortages/receipt/post` 500s (service returns nothing) — FALSE: `service.post` returns
  `this.get()`, header included. Don't re-report.
- Live test scripts the forks used sit in that session's scratchpad (`*-live/`); recipe as in [[erp-qt-75c682b-port]].
