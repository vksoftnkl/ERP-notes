---
name: erp-notes-95-staff-advance-till-pin
description: "Notes 95 (till_operator dropped) server half built 2026-10-07, staged uncommitted; dev DB backfilled; open items are till phase 3 and the Open Till error code"
metadata:
  node_type: memory
  type: project
  originSessionId: b4432845-bb98-494c-be58-deb4669eb306
  modified: 2026-10-08T06:51:53.528Z
---

Notes 95 is the server half of `Dev/till/plan-till-operator-to-user.md`. The note itself is on the SMB share at `Prathap/admin/notes-95-user-employee-till-fields.md`. It was built 2026-10-07 and is **staged, not committed**. The pieces:
- `StaffAdvanceLedgerService`: the employee save creates the "<name> - Staff Advance" ledger.
- `secret-hash.utils`: the PIN uses the password's scrypt format.
- `usrPin` / `usrPinSet` on the user.
- `usrEmployeeId` checks.
- The ledger delete guard.
- `POST /employee-masters/backfill-staff-advance-ledgers`.
- Dropdown 63, seed only.

On 2026-10-08 I added the plan §4 "employee is active" rule to the user save. It applies only to a new link or a re-activation. It is not staged.

**State on dev:** every employee has a ledger, because the backfill ran. Dropdown 63 is in the DB. 17 tests pass in `test/employee-*-notes-95.e2e-spec.ts`.

**Still open:**
- **Open Till error code.** A refused Open Till answers 403 `TILL_RIGHT_CREATE`. The plan says `TILL_NOT_ALLOWED`.
- **RECOVER and the PIN gate.** RECOVER → `emp_loan_ledger_id` and the PIN lockout belong to till phase 3, which is not built. On `TILL_LEDGER_UNMAPPED`, the plan says 409 and the till module uses 422.
- **`usrPinChangedOn`.** Skipped, because it would need a column.
- **Live.** Dropdown 63 needs a migration or a seed run; see [[erp-live-vps-deploy]]. The backfill route must also be called once.

**Why:** the user re-asked to "run this plan" a day after it was built, so the work was not visible to them. Check `git diff --cached` before rebuilding.

**How to apply:** for a till or employee-ledger request, start from this state. Another session owns `src/modules/till`; see [[erp-notes-80-txn-status-log]] for the event conventions.
