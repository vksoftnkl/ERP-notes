---
name: erp-receipt-entry-port
description: Receipt Entry (menu 99) React port — where it lives, the wire traps verified live, and the UI reference it matches
metadata:
  type: project
---

Receipt Entry (menu 99, "Receipt" under Accounts) was ported to React on
2026-09-24 under `features/accounts/receipt/` + `store/api/receiptApi.ts`,
routes `/accounts/receipt` and `/accounts/receipt/:company/:branch/:year/:id`
(the drill-through target — a receipt is addressed by FOUR keys everywhere).

Verified against the live server, not just the plan:

- **`areaId` must NOT be sent.** The plan's draft body lists it; the server runs
  `forbidNonWhitelisted: true` and `SaveDraftReceiptDto` has no such property,
  so it would 400 the whole request. The beat filters dropdown 54 and nothing else.
- **`writeoffApprovedBy` is a UUID** (`OptionalUuid`), not a typed name — so the
  screen needs a user picker (grid 62), not the Qt workaround of "put the name
  in the narration".
- **The envelope trap is real:** `/create` returns the id at
  `data.header.avhVoucherId` and `data.avhVoucherId` is undefined.
- **A draft really does touch nothing (R10):** probed create → get → delete; the
  bill it named stayed at its full pending amount and no number was allotted.
- Grids/layouts: grid 108 (register, `iavh_status` MUST be sent — omitting it
  400s the whole query), 111 banks, 112 bill payment history, 62 users;
  `ui_tables` 32 "RECEIPT - BILLS" (24 cols) and 33 "RECEIPT - TENDERS" (18),
  both storing Qt fractions. Dropdown 54 "CUSTOMERS BY AREA" needs `iarea_id`
  bound even when empty.

The UI follows a reference screenshot the user supplied: title + subtitle +
status pill, one 8-field doc strip (no customer), then PARTY DETAIL |
LAST RECEIPTS | CHEQUES STILL OUT, the bills grid under
"WHAT THE PARTY OWES — AND WHAT IT HOLDS" (with an Amount box that writes the
first instrument), the instruments grid under "HOW IT CAME — AND WHAT WAS
DEDUCTED", the identity band, and one row of labelled buttons (no separate
shortcut legend — the keys are on the buttons). Keys: F1 New, F5 Save draft,
F6 re-read, F7 paid detail, F8 list, F9 ledger statement (not built yet),
Ctrl+Enter post, Ctrl+PgUp/PgDn walk, Esc close.

See [[erp-rtk-lazy-trigger-in-mount-effect]] and
[[erp-client-ui-browser-verification]].

**Grid affordances + keying order (2026-09-24).** Both grids import
`useColumnResize` and `useGridSettings` from `features/sales/quotation/components/`
— drag-to-widen (saved as `ui_tbl_clm_px`, local until "Save column width")
and the right-click Admin-settings dialog — exactly as Opening Balance does.
Each grid's viewport owns `onContextMenu`; each header cell carries the resize
handle; `{settings.overlays}` renders once per grid. Both pass a `focusNote`
saying Focus is NOT used on these grids (the bills grid decides editability by
row kind, the instruments grid by row KIND), or the dialog invites ticking
boxes that do nothing.

`focus-walk.ts` declares the keying order — receiptNo → date → beat →
collectedBy → theirRef → refDate → ourRef → narration → party → amount — and
Enter follows THAT, not the DOM. The party picker and the Amount box sit in a
panel and a section bar below the header strip, so the app's own
`focusNextInteractiveControl` (DOM order) would visit three buttons and a
table on the way. The three `NexDropdownSingle`s therefore set
`advanceFocusOnSelect={false}` and let the walk move on. The caret starts on
the read-only RECEIPT NO and is re-placed on every new/loaded document, keyed
on the header — the screen does not unmount between receipts.

**Print (2026-10-14).** Purpose `RECEIPT_VOUCHER` (ACCOUNTS / ACC_VOUCHER,
already seeded) through `PrintOptionsDialog`: list toolbar Print + multi-select
(drafts refused — no number), entry screen Post & Print opens the dialog on the
just-posted receipt (`post()` now returns a `ReceiptPrintTarget | null`, since
the screen is cleared before the dialog shows), and F11 / Print reprints the
opened posted or cancelled receipt. The same day `RECEIPT_A5` was built, published
and assigned GLOBAL in the dev DB — see [[erp-receipt-print-template]]. See
[[erp-document-print-button]].
