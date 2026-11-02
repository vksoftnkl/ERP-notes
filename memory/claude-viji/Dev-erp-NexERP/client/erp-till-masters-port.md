---
name: erp-till-masters-port
description: "Till Masters (menu 271 › 275) React port of Qt 09274cd at /sales/till-masters — grids 133–136 + dropdowns 64–67 by name, live-verified 2026-10-09; plus the rest of that Qt pull (PIN/employee, staff ledger, document rights, tdSettleStatus)"
metadata:
  node_type: memory
  type: project
  originSessionId: ae95de82-c4ee-47e7-b59c-dea93586f83b
  modified: 2026-10-09T05:32:31.136Z
---

Ported 2026-10-09 from NexERP 09274cd/f1dd5fa. `features/sales/till-masters/` (domain/ pure + vitest,
`use-till-tab.ts` one hook per tab, all four tab bodies stay mounted with `hidden`), `store/api/tillApi.ts`,
route `app/sales/till-masters`, catalogue entry Sales › Till › Till Masters (menu row exists; only VKPOS granted).

Wire facts:
- Lists = grids 133 counters / 134 reasons / 135 denominations / 136 safes (registry keys `till*List`), QUOTED
  `'icompany_id'` (+ `'ibranch_id'` for counters/safes). Numerics arrive as strings ("500.00"), counts as strings.
  No /get — a row opens from its grid row. Save = `/till/<master>/create` (id in body = update).
- Dropdowns 64 TILL DEVICES (`ibranch_id`,`icounter_id`), 65 CASH LEDGERS, 66 TILL REASON LEDGERS, 67 TILL SAFES —
  pass `clearOnParamsChange={false}` or switching rows wipes the pick.
- The screen posts the HEADER's business context, not a seeded one — a CDP test must read
  `businessContext` back from sessionStorage before checking the server. Names come back title-cased
  ([[erp-global-uppercase-inputs]]), so compare case-insensitively.

Rest of that pull, also done: User Admin (Till PIN + Clear, Employee via dd 38 with nil-uuid branch token,
SUPERVISOR role, tree now `visibleOnly=false` = 245 menus with hidden ones greyed — see
[[erp-user-administration-port]]); Employee "Staff Advance Ledger" uses dd 63 STAFF ADVANCE LEDGERS, not
Qt's 43 (43 offers ledgers the server refuses); tender payloads no longer send `tdSettleStatus` (server
sets PENDING only when absent); stock screens follow `lib/permissions/document-rights.ts`
([[erp-menu-permission-gating]]).

Live: grids 133–136 and dropdowns 63–67 exist only in dev until synced ([[erp-live-config-drift]]).

**2026-10-10:** superseded by the Qt d5c3637 split — four screens at `/sales/till/{counters,safes,reasons,denominations}`, `/sales/till-masters` redirects; see [[erp-qt-d5c3637-till-expense-settlement-port]].
