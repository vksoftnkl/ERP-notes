---
name: till-safe-notes-118
description: "Notes 118 built 2026-10-15 — safes/balance · count · decide-variance, movements/remit · remit-confirm (Cash Office menu 287); safe gaps are till_variance stage SAFE/REMIT with NO session; counter decides + witness is the two-person second (user's call); SAFE_NOT_COUNTED RED by default"
metadata:
  node_type: memory
  type: project
  originSessionId: 1e7ecb9a-55b9-4694-8a97-3ac095580f2b
  modified: 2026-10-15T04:45:16.196Z
---

Migration `20261015100000_till_safe_notes_118`, `TillSafeService` + `TillSafeController` (src/modules/till, README "The safe").

Decisions (the notes left them to me):
- A safe gap lives in `till_variance` (stage SAFE = a count, `tvr_count_id`; REMIT = a CIT bank credit short, `tvr_movement_id`), `tvr_session_id` NULL, `tvr_safe_id`; `ck_tvr_owner` replaced `ck_tvr_receive`; `ux_tvr_live` gained safe + count. Any reader that assumes a variance has a session must filter by stage.
- REMIT stays on `till_cash_movement` (`tcm_remit_mode` BANK_DEPOSIT|CIT, `tcm_remit_status` DEPOSITED|IN_TRANSIT|CONFIRMED, `tcm_bank_*`, `tcm_confirmed_*`, `tcm_confirm_voucher_*`); `tcm_ledger_id` = the bank either way. Confirm voucher is CRem too, src `TILL_REMIT_CONFIRM`.
- Expected balance: before the first count = the safe ledger's book (so pre-till cash is no gap), but 0 when another active safe of the branch shares the ledger (safes default to the SAFE_CASH role's) — `openingFrom` BOOK / ZERO / COUNT. Back-office cash on the safe ledger (48 §2.3 tender rows) counts as flows. Flows are timed (create / void, receipt, post / cancel); `tbd_safe_opening` is walked back from now.
- Blind: under `till.blind_close` the balance's money is null without 287 or 273 OVERRIDE.

**Why:** the user chose (2026-10-15) "counter + witness": the caller decides a gap on their own SAFE_VARIANCE grant, the rule's no-self flag does not bind a safe count, and the witness's PIN is the two-person second. `assertApprover` grew `selfExempt` and `second` — `second` undefined keeps the old "Part B" refusal for every other caller.

**How to apply:** `till.day_close_safe_count` ships TRUE, so SAFE_NOT_COUNTED is RED on any branch whose safe holds cash — day-close e2e suites must turn it off on their temporary branch (till-day-close-http does) or count the safe. `test/till-safe-http.e2e-spec.ts` (9 tests) runs on a temporary branch with a 10,000 opening balance row; `--runInBand`. Not built: SAFE_TRANSFER route, the remittances grid (user's; SQL handed over), a safe count print. See [[till-day-close-notes-117]], [[till-variance-decision-notes-105]], [[till-handover-receipt-notes-113]].
