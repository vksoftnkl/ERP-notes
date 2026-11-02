---
name: erp-qt-d5c3637-till-expense-settlement-port
description: "Port of Qt NexERP d5c3637 (2026-10-10) — till masters split, Open Till/Sessions/cockpit, Business Day, Approval Setup, Expense Voucher, Settlement Reconciliation; routes, the till-session store, and two open house-level bugs"
metadata:
  node_type: memory
  type: project
  originSessionId: 0b73ab1f-c7a9-4e96-ab8e-ef67bead65d9
  modified: 2026-10-15T11:20:32.963Z
---

Ported 2026-10-10 from NexERP 09274cd..d5c3637, built by six parallel subagents (one folder each), menus
wired in `components/layout/constants.ts` (Sales › Till › … and Accounts › …):
- 272 Open Till `/sales/till/open`, 273 Till Sessions `/sales/till/sessions` + cockpit
  `/sales/till/sessions/:company/:branch/:year/:tssId` (Business Day links there) — `features/sales/till/{session,sessions,shared}`
- 274 Business Day `/sales/till/business-day`; 276 Till Approval Setup `/sales/till/approval-setup`
- 283 "Till Masters" is now a GROUP: 275 Till Counters / 280 Till Safes / 281 Till Reasons / 282 Denominations at
  `/sales/till/{counters,safes,reasons,denominations}` (`features/sales/till-masters`, `/sales/till-masters` redirects)
- 277 Expense Voucher `/accounts/expense-voucher` (+ keyed route); 278 Settlement Reconciliation
  `/accounts/settlement-reconciliation?asiId=&accYear=`
Grids 137–152, 154 and dropdowns 68/69 registered BY NAME in the two registries (pre-registered centrally so
parallel agents never edit them at once).

**Till session state** = Qt AppSession::TillSession: `features/sales/till/shared/session-store.ts` (sessionStorage
`erp.till.session`, stamped with user + login sid, cleared on auth event); `useTillSession()`, `getTillSessionId()`.
Sale bill / testbill / sale order send `sessionId: getTillSessionId() ?? getAuthSessionId()`; Expense Voucher's
`till-session.ts` uses it only when company+branch match the header.

`canOverride` is now parsed from usermenu (types.ts, constants.ts `toMenuPermissionFlags`, menu-permissions FULL/NO/merge).

**Open house-level bugs (auto mode refused edits to shared store files — needs the user's go-ahead):**
1. FIXED 2026-10-15 exactly as prescribed below (see [[erp-data-freshness-bus]]). Was: `store/store.ts` `announceMutations` announces EVERY fulfilled mutation, incl. GET-as-mutation reads (Party
   Outstanding, Ledger Statement, UserAdmin, GST, Ledger Map, the new till/expense/settlement APIs) → same-tab
   `useDataRefresh` subscribers re-read and every OTHER tab invalidates all RTK tags, on every read. Fix: return
   `meta: result.meta` from `baseQueryWithAuthHandling` (baseApi.ts drops it) and skip when
   `meta.baseQueryMeta.request.method === "GET"`. Business Day avoided it with `initiate(…,{subscribe:false, forceRefetch:true})` queries.
2. FIXED 2026-10-15 (`store/api/run-all-pages.ts` pages all three directories). Was: `metadataApi.getGridDirectory` read ONE page of 100; the server caps a page at 100 and dev has 105 Desktop grids
   → grids past 100 (149–154 today) never resolve by name and silently use fallback ids (wrong on live).

Smoke-checked all 13 routes headless with faked API ([[erp-client-ui-browser-verification]]): a 404 from a fake for
`…/get` makes EVERY CrudMasterPage list show a "Cannot continue / Not found" popup — the shell's own call, not the screen.
Related: [[erp-till-masters-port]], [[erp-menu-permission-gating]], [[erp-rtk-pending-query-dedupe-abort]].
