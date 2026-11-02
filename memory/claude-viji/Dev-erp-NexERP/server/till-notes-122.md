---
name: till-notes-122
description: Notes 122 (2026-10-15 retest) — PENDING_APPROVAL device still a till, named real sessions judged anywhere, unknown session ids stored NULL, till.cash_out_over_expected (DEVICE scope, REFUSE), APPROVER_NO_PIN, SLIP_GAPS reads SLIP_CHECK, safe opening = ledger as of the date; 269ST stays off
metadata:
  type: project
---

Built 2026-10-15 (migration 20261015150000_till_notes_122, catalogue 2026-10-15.2):
- `resolveForMoney`: `held` uses HOLDING_SESSION_STATUSES (PENDING_APPROVAL too); a non-optional
  document naming a REAL till session is judged on any device (late arrival still allowed); an
  optional doc on a non-OPEN held drawer returns null. New `cashOut` scope → 409
  TILL_CASH_SHORT_IN_DRAWER (message never names the drawer's figure — blind close).
- `till.cash_out_over_expected` REFUSE|ALLOW, max scope DEVICE (so a test device can ALLOW):
  expense-voucher-http sets ALLOW on its own till PC (created_by 'e2e-expense-drawer'; the §2.3
  test deletes every 'e2e-expense' row mid-suite, so never reuse that tag).
- SLIP_GAPS also counts the latest SLIP_CHECK event's live noSlip/amountDiffers rows + slips
  without a row (the event had no reader before).
- refreshDay: safe opening/expected = safe ledgers' balance as of (date−1)/(date), each ledger
  once; `expectedAt` removed. Negative books print as they are (ZTS ledger has no opening balance).
- Dropdown 68 and dropdowns 64–69, grids 133+ are DB-only (made on screens): Dropdown_Details.sql
  stops at 63, Grid_Details.sql at 132 — a fix is a guarded UPDATE migration, no seed row to edit.

**User's calls (10-15):** §3 an sbSessionId/soSessionId naming no till session is stored NULL on
save (`till/till-session-id.ts` in create/updateBillInTx, create/updateOrderInTx), NOT refused —
the React web client sends `getTillSessionId() ?? getAuthSessionId()` (the LOGIN session) when no
till is open. §4 269ST/PAN stay OFF (the user's own commit 273a8aa4, 10-06): a tester expecting a
269ST refusal is told it is off by design. Notes 123 (10-15) puts 269ST / PAN / HSN ON HOLD — "we
can't make the customer wait at the counter"; when picked up it starts as WARN only, never a
counter refusal. Walk-in today = the bill's effective `sales.default_customer_id` (max scope
DEVICE, so one per counter is possible) — there is no walk-in flag on sales.customers. Notes 123
§5 LOADED 10-15 (catalogue 2026-10-15.3, 202 rows): the user pasted sales-error-wording.md REV 1
as approval; rows built from its tables (the seed JSON was never on this box). SALES_CASH_LIMIT has
an `isWalkIn=true` variant — the refusal must carry `isWalkIn` when 269ST returns. No `SALES_*`
prefix row (an APP-worded catch-all would misword future user-facing codes).
See [[till-management-schema]], [[e2e-suites-steal-default-safe]].
