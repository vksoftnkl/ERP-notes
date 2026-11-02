---
name: erp-testbill-module
description: "features/sales/testbill = the FULL REV 2 sale-bill plan (docs/plan-react-sale-bill.md) in the plan's §2 layout with a local useReducer; mounted at /sales/testbill beside the older /sales/sale-bill; built 2026-09-25"
metadata:
  node_type: memory
  type: project
  originSessionId: 7bdf3a2d-1316-4195-a45f-8470e14f3dde
  modified: 2026-09-25T14:37:05.143Z
---

`features/sales/testbill/` is the complete implementation of `docs/plan-react-sale-bill.md`
(REV 2, phases 1–6) laid out as the plan's §2 asks: `state/` (draft reducer + `use-bill-draft`
hook + verbs + hold), `domain/` (validate, party, notes, import, load-map, rollups, carry,
shipping), `payload/` (one builder per DTO + dto-keys), `engines/` (promotions, adjust,
settle, weight-barcode), `api/` (facade over `store/api/saleBillApi` + 4 own endpoints),
`components/`, `retender/`, `lists/`. Route `app/sales/testbill/page.tsx`; the two registers
are `/sales/bill-delivery` (menu 226, grid 113) and `/sales/temp-credits` (menu 257, grid 114),
labels added to `DEFAULT_PRIMARY_MENU`.

**Why:** the user asked for the whole plan "in this folder" (2026-09-25) after `features/sales/salebill`
had reached phase 4; `testbill` was scaffolded FROM salebill (phases 1–4 kept verbatim, imports
rewritten) and phases 5–6 were added on top. The older `salebill` module and `/sales/sale-bill`
route still exist untouched.

**How to apply:**
- The draft is `useReducer(billReducer)` inside the hook (plan §5.1) — NOT the global
  `saleBillSlice`. `state/draft.ts` still uses `createSlice` for its action creators but is never
  registered in the store; `storeDispatch` is used only for RTK `initiate()` reads.
- New in phases 5–6: Open Sources dialog (Ctrl+F4 on a bill with lines / Ctrl+F5 → `appendOpenSourceLines`,
  source rate kept via `itemPriceApplied({keepRate})`), charge carry (`domain/carry.ts`, carry keys
  emitted only on carried rows by `payload/build-charges.ts`, `/validate` proposals applied),
  shipping popup (`domain/shipping.ts`; PUT `/bills/transport` once posted; §16.5 note opens it via
  `transportRequiredTick`), GST e-Inv / e-Way / Part-B, quick-add customer (`/customers/create`),
  re-tender (Ctrl+F6; grid 115 picker; rows numbered after ALL existing rows), hold/recovery
  reconciliation of set-offs + party-context re-read, F11 reprint (`isReprint`), crew names resolved
  by id from the employee dropdown after load.
- Shared edits made for it: `DropdownCombo` and `quotationApi.runDropdown` accept optional `params`
  (`dropdown_param`) — needed by the param-bound transporter (55) / vehicle (56) dropdowns.
- Not built (server/other-screen gaps, all surfaced honestly in the UI): e-invoice / e-way registers
  (menus 152/153, Shift+F2/F3 toast), a customer ship-to list screen, printing beyond
  `/print-render` (needs the bill's template).
- Verified: `tsc --noEmit` clean, 422 vitest cases in the folder pass, the three routes compile on
  the dev server. NOT yet exercised live against a registered device (the §29 acceptance modes).
