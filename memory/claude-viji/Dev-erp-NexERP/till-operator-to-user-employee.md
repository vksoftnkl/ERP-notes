---
name: till-operator-to-user-employee
description: "2026-10-07 DONE — accounts.till_operator dropped from 47; the till reads user_master (PIN, employee) + employee_master (staff advance ledger, auto-created on employee save via notes 95); Qt fields added"
metadata:
  node_type: memory
  type: project
  originSessionId: 437b9134-899f-4f46-94f2-a058376f179c
  modified: 2026-10-07T13:26:05.444Z
---

**Decided 2026-10-07 (user: "drop till_operator … create a loan ledger at the time of employee creation").** Written up in `share/till/plan-till-operator-to-user.md`; this file overrides TILL_DESIGN §5.1/§5.7/S10/§14-2.

**What changed:**
- **47 edited in place** (it is unapplied). §1.3 is now a note plus `DROP TABLE IF EXISTS`.
  - Two re-run seed guards were fixed: running 47 after 48 used to re-seed PAID_OUT reasons and rules.
  - Validated on a fresh throwaway PG with 47→48→47→48: 24 till relations, 20 rules, 45 reasons, the 10 probe refusals intact.
- **Updated:**
  - `47_TABLES_GUIDE.md` (now 16 objects);
  - `47_tables_map.png` and `47_session_flow.png`, re-rendered from copies of the other session's er.py/flow.py (now in this scratchpad's `er/`);
  - the masters mockup: picture 5 is `till_user_employee_ui_mockup.png`, and the old till_tab PNG was deleted;
  - the till guide artifact (v2).
- **Server asks:** `share/admin/notes-95-user-employee-till-fields.md`.
  - The employee save creates "<name> - Staff Advance" in Loans & Advances (Asset), type PARTY (so Payment dropdown 60 offers it), branch NULL, in the same transaction.
  - New `usrPin` → `usr_pin_hash`; the GET returns `usrPinSet`, `usrEmployeeName` and `empLoanLedgerName`.
  - The PIN is checked only inside the till approval service.

**Notes 95 LIVE + TESTED 2026-10-07.** Prathap deployed it and ran the backfill; all 3 employees have ledgers.
- **Tests:**
  - `tests/staffledger/run-api.py`: 35/35 checks. Covers create, rename, null keeps the ledger, the 400 and 409 picks, the name-clash suffix, the ledger-delete guard, the balance 409 via an opening Dr 100, the PIN keep/clear/400, and the employee link 409 or inactive-OK.
  - `tests/staffledger/run.sh` (offscreen Qt): 24/24. Covers the Employee ledger name, the User employee name, and the PIN placeholder, validation, keep and clear.
  - Both clean up their ZT- rows.
- **Qt (uncommitted):**
  - `user_entry` has Employee (dd 38, branch token from usrBranchId, nil uuid when there is none) and usrPin with a screen-only `_clearPin`. Both are applied in a `fillFromJson` override; that virtual is the house post-load hook.
  - `employee_entry` has empLoanLedgerId (dd 43, every ledger; the server refuses ones outside Loans & Advances).
- **Trap:** a successful NexCrudForm save closes and deletes the form. A test must hold it in a QPointer and reopen it.
- **Open:** dropdown C (staff advance ledgers only) is not registered. The backfill route has no admin guard.

**Build:** use `build/Desktop_Qt_6_10_2-Debug`, the folder the test runners use. `Desktop_Qt_6_10_3-Debug` is a stale folder, because Qt 6.10.3 is not installed.

**Facts:**
- Before notes 95, employee create wrote no ledger. It now does, via StaffAdvanceLedgerService; unlike a customer ([[customer-creates-ledger]]) the employee does not share the ledger's id.
- The `claude` role cannot write `fixed.dropdown_details`.

Links: [[till-management-design]]
