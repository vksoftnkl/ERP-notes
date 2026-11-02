---
name: shipping-dialog
description: "TxnShippingDialog (shared .ui popup, 3.0 ShippingDetail layout) replaced the inline TxnTransportBand on the sale bill 2026-09-24; DC still on the band and is next"
metadata:
  node_type: memory
  type: project
  originSessionId: 2a57b2fc-d57a-4c6a-8daf-436af25c2f7f
  modified: 2026-09-24T04:47:09.598Z
---

2026-09-24: `src/shared/ui/txn/txn_shipping_dialog.{ui,h,cpp}` — the Shipping
popup as its own .ui, laid out like 3.0 `NextLvl/3.0/pos/Sales/shippingdetail.ui`
(Transport on top, Dispatch From | Ship To, `vboxExtra` slot, Clear F7 /
Save transport F5 / Close). Same API as TxnTransportBand (setData/readFlat/
writeFlat/toNested/locks), so a screen swaps the type and keeps its calls.

User decisions: convert the BILL first, then the DC (DC still uses the inline
band — convert it next, then delete txn_transport_band.*). The Site "+" opens
ShipAddrList for the customer and re-reads only `shipTo[]` on close. Vehicle
stays a HEADER field, mirrored in the popup.

Choices made: Dispatch From is read-only, from GET /branch-masters/get?brId
(godown has no address; branch does). Empty Ship To = "as bill-to", shown as
grey placeholders and never written (prefilling would stamp a ship-to on every
bill that opened the popup). 3.0's Vehicle Type / Transaction Type / 3 address
lines were left out: they aren't in txn_transport_detail (33), and they belong
to generating the e-way bill.

Known gap: NexDropdownSingle::setSelection(id, "") shows no text and there is
no id→text lookup. State is resolved from a static GST-code table in the
dialog; a saved GODOWN still shows blank.

Related: [[sale-bill-upgrade]], [[transport-band-shared-table]]
