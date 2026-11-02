---
name: till-counter-signed-movements-notes-110
description: "Notes 110 built 2026-10-13 — /till/movements/create PICKUP/TOP_UP are the CASHIER's call + supervisorId/supervisorPin; DROP needs the counter's own drop safe (tsf_is_drop_safe); /change keeps the notes 108 roles"
metadata:
  node_type: memory
  type: project
  originSessionId: 58c50534-10fb-4a74-a175-a05ddec92459
  modified: 2026-10-13T05:13:48.250Z
---

Notes 110 (2026-10-13) flipped who calls a pickup / top-up on `/till/movements/create`. The caller is now the
session's cashier on its device (272 EDIT); anyone else gets 422 `TILL_MOVEMENT_AT_COUNTER`. The supervisor signs with
`supervisorId` + `supervisorPin`, checked before the transaction by `proveSupervisor`. The row records
`tcm_done_by` = cashier, `tcm_witness_by` = supervisor. `GET /till/movements/supervisors` lists who can sign.
Migration `20261013100000` adds `till_safe.tsf_is_drop_safe` and puts it into grid `MAIN LIST - TILL SAFES` (matched by
name). A DROP goes only to `tcn_safe_id` when that safe is flagged, else 422 `TILL_NO_DROP_SAFE`. Every session
payload carries `dropSafeName`.

**Why:** the user wants every cash hand-over signed in front of both people, so nothing can be recorded behind
the cashier's back.

Decisions that were mine, not the user's:
- Supervisors are NOT narrowed to the company or branch, although notes 110 §2 says "for this company / branch".
  `user_menus` is not scoped, and a home `usr_company_id` does not bind where a user works: VKPOS, whose home is
  company 019cc885, supervised VIJI PC in 019c8ea6. A home-company filter, built first and then reverted the same day,
  hid the user's real supervisor. `signerScope` (active, not deleted) is shared by the list and the check. Ask the
  user before adding any scope.
- `witnessPin` stays in the DTO, accepted and ignored. Dropping it would 400 any client still built for notes 108.

**How to apply:**
- `/change` and `/void` were deliberately left alone. `/change` still runs `checkWitnessPin` (TILL_WITNESS_* codes,
  supervisor calls).
- A `PIN_FAIL` row's `tev_user_id` is the caller. On /create that is now the cashier, and the payload names
  `supervisorId`.
- Rows posted before notes 110 carry the reversed roles. On the dev box that was 4 rows: counters ZT1 and VIJI PC.
- Existing safes default to false, so drops are refused until a safe is ticked. Any suite that drops must create its
  safe with `tsfIsDropSafe: true`, as till-money-paths does.

See [[till-witness-pin-notes-108]], [[till-management-schema]].
