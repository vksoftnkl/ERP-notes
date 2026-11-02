---
name: erp-contra-voucher-register-port
description: "2026-10-03 Contra (menu 104, type Con) ported to React as the Voucher Register with the type fixed — features/accounts/vouchers; uncommitted; since grown to every register type"
metadata:
  node_type: memory
  type: project
  originSessionId: e05d84f3-470d-4bf0-bac8-1b311496d7c1
  modified: 2026-10-03T06:52:36.195Z
---

Built 2026-10-03 in `/home/vk/Dev/erp/ERP client`, UNCOMMITTED: `features/accounts/vouchers/` (voucher-page = list
→ entry, voucher-screen, state/use-voucher-entry, domain/lines, components: legs-grid, ledger-picker, review-dialog,
reason-dialog, voucher-list-view), `store/api/vouchersApi.ts`, route `/accounts/contra` (contra-page: typeCode "Con",
menu 104), registries grid 117 `voucherRegisterList`, ui table 39 `voucherLegs`.

Qt has no Contra screen: menus 101-104/163/259-262 all open `VoucherRegisterEntry` with a type code. The React screen
is generic; [[erp-receipt-payment-voucher-port]] added MANY-party/instruments/TDS and [[erp-voucher-register-all-types]]
added the ONE-party header, GST band, raised bills and the menu-262 register — no type is refused any more.

Contract facts: `/vouchers` keys are BARE (companyId/branchId/accYear/voucherId); line `amount` is a decimal STRING;
`/create` rejects `overrides` (400); rights come per type from `/vouchers/types?menuId=` (incl. post/cancel/override);
`/validate` always 200 (debounced 400ms); a draft re-posts the full payload; draft lines come back in `draft` with ids
only. Grid 117 needs `iuser_id` sent by the client (SQL joins user_menus) and has no status token. Grid 119 (Qt's ledger
popup) is dev-DB only → React uses `/vouchers/ledger-pick`. Server allows same ledger both sides and cash↔cash.

Verified without login: bodies/queries vs compiled DTOs, grid 117 read-only on dev (con00001-3), SSR render. Never
clicked through logged-in.
