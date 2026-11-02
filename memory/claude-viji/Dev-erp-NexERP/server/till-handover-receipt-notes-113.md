---
name: till-handover-receipt-notes-113
description: "Notes 113/114 — till.handover_receipt REQUIRED parks CLOSE_HANDOVER / DROP in role TILL_CASH_AWAITING; since notes 114 (2026-10-14) a supervisor approves a CLOSED session's cash with POST sessions/approve-cash (273 OVERRIDE, no count); Cash Office 286 retired; the user's company is REQUIRED"
metadata:
  node_type: memory
  type: project
  originSessionId: c78c5194-a12d-48dd-a125-6bfccf139357
  modified: 2026-10-14T12:24:52.668Z
---

Migration `20261014100000_till_handover_receipt_notes_113`: role `TILL_CASH_AWAITING` (catalogued,
Cash-in-Hand, CASH), voucher type `TRcv` "Till Cash Received" (Contra), setting
`till.handover_receipt` NONE|REQUIRED (BRANCH max, default NONE), receipt columns on
`till_cash_movement` (`tcm_receipt_status` NULL|AWAITING|RECEIVED, received_by/on/amount,
receive_count_id, receive_voucher_id + acc year), `till_variance.tvr_stage` RECEIVE +
`tvr_movement_id` (in `ux_tvr_live`), event `CASH_RECEIVED`.

**Notes 114 (built 2026-10-14, migration `20261014160000_till_approve_cash_notes_114`):** the user
found Cash Office (blind bag count + receiver PIN + third-person decider) too much. Now
`POST /till/sessions/approve-cash` (273 OVERRIDE) approves every AWAITING movement of a CLOSED
session for one total; `ck_tcm_received` no longer requires a count id. Menu 286 is
`menu_is_active` false AND its user_menus grants soft-deleted — `/user-administration` save 400s
on an inactive menu id, so a live grant would break every rights save of its holders. The old
`movements/awaiting-receipt` / `movements/receive` routes stay, judged on 273 VIEW / OVERRIDE.

Design choices (mine):
- approveCash lives in TillMovementService (it reuses verifyPin / postReceive; the session service
  cannot inject the movement service — circular), so `varianceReason`, `staffLedgerOf`,
  `hasAnyPending` on TillSessionService are public for it
- one RECEIVE variance per approval, anchored on the CLOSE_HANDOVER (else the latest drop); a
  shortage spills hand-over first then drops newest first, none below 0 (a 0 share = no TRcv)
- notes 116 (user's rule): approve-cash / receive refuse ONLY the session's cashier
  (`tssOperatorId`), never `tcm_done_by` — a supervisor who closed for the cashier owns the
  hand-over and may approve it; message "Session X is yours (you were its cashier)…"
- in-step decision: assertApprover with requester = the cashier; a gap sent on is decided via
  decide-variance + tcmId where requester = tcm_received_by, so the approving supervisor can't
- `receiptState` renamed: null → NONE, RECEIVED → APPROVED; payload adds `cashAwaiting`,
  `receivedBy/receivedByName/receivedOn`

**Why:** live VIJI PC-261014-02 — a cashier closed and "handed over" ₹500 into the safe alone.

**How to apply:**
- The user's company 019c8ea6 (the e2e company too) is REQUIRED at COMPANY scope — set by VKPOS
  2026-10-14 12:51, not by me. Till suites still pass (they don't assert hand-over legs).
- `test/till-handover-receipt-http.e2e-spec.ts` (13 tests) needs `--runInBand`: it sets prathap's
  PIN, two grants, BRANCH setting rows (handover_receipt, single_operator briefly).
- Designer grids 144 / 150 read `tvr_stage <> 'OPEN'`; should be `= 'CLOSE'` (a RECEIVE row is
  newer). Not changed — the user's grids; I handed them a tested LATERAL snippet for a
  `cash_receipt_state` column instead.
- Day close (phase 4) must refuse while `days/get awaitingReceipt.thisDay > 0`.

Related: [[till-variance-decision-notes-105]], [[till-counter-signed-movements-notes-110]], [[till-management-schema]], [[new-ledger-role-needs-catalogue]], [[user-menu-transaction-rights]].
