# Till: no `till_operator`; the till reads the user and the employee (2026-10-07)

> **User decision, 2026-10-07:** *"drop till_operator, but add missing fields in user master if it needs;
> shall we create a loan ledger at the time of employee creation."* Yes to both.
>
> **A separate plan, per house rule.** `TILL_DESIGN.md` and `plan-till-review-rev2.md` are not edited.
> §0 lists what this file overrides. **Where they disagree, this file wins.**
>
> **What was edited in place, with the originals kept in the session scratchpad:**
> - The SQL file 47, which has not been applied anywhere.
> - `47_TABLES_GUIDE.md`, `47_tables_map.png` and `47_session_flow.png`, because they describe the SQL.
> - The masters mockup (picture 5).
>
> **Server work** is in `../admin/notes-95-user-employee-till-fields.md`.

## 0 · What this overrides

| In | Section | Now |
|---|---|---|
| TILL_DESIGN.md | §5.1 "Roles (`till_operator.top_role`)" | Roles live only in `till_approval_authority.taa_role`. A cashier is anyone holding the Open Till menu right. AUDITOR is a read-only menu right, not a role. |
| TILL_DESIGN.md | §5.7 Variance RECOVER row | The debit goes to the staff advance ledger: `tss_operator_id` → `usr_employee_id` → `emp_loan_ledger_id`. |
| TILL_DESIGN.md | §6.x table list, `till_operator` row | Dropped. 47 now has 15 `till_` tables plus 2 columns on `acc_tender_types`. |
| TILL_DESIGN.md | §8 masters "authored on the store" list | Operators drop out; the user and employee masters are authored where they always were. |
| TILL_DESIGN.md | S10 masters list | "Till Operator" is removed. Six till masters remain. The user and employee screens gain fields (§3). |
| TILL_DESIGN.md | §14 decision 2 | "loan ledger only": `top_recovery_ledger_id` is gone. The one source is the employee's staff advance ledger. |
| plan-till-review-rev2.md | anything naming `top_*` | Read as §1 below. |

## 1 · Where each operator fact went

| Was (`till_operator`) | Now | Why |
|---|---|---|
| `top_user_id` | gone. `tss_operator_id` already holds `usr_id`. | Nothing in 47 referenced the table. |
| `top_employee_id` | `user_master.usr_employee_id` (exists; the API already accepts it) | Same fact twice would drift. |
| `top_recovery_ledger_id` | `employee_master.emp_loan_ledger_id`: the **staff advance ledger** | User decision: recovery goes to the loan ledger only. |
| `top_role` | `till_approval_authority.taa_role` | This was a label only; the authority grid was always the truth. |
| `top_can_open_session` | menu right on the Open Till menu (`user_menus`) | Reuses the rights screen admins already know. |
| `top_max_float` | gone | The counter default decides. A different float at open is a FLOAT_MISMATCH approval. |
| `top_badge_hash` | deferred | PIN only for now. A badge is one later column on `user_master`. |
| `top_company_id` | gone | The user's own company/branch scope plus the authority rows. |

## 2 · Staff advance ledger, created with the employee

**Rule:** saving an employee that has no `emp_loan_ledger_id` creates one, in the **same server
transaction**. This is the same pattern as a customer, whose save writes `acc_ledger_master` in the
same call.

| Ledger field | Value |
|---|---|
| name | `<emp_name> - Staff Advance`. On a name clash, append ` (<emp_code>)`. |
| group | **Loans & Advances (Asset)**: shared, reserved, under Current Assets. |
| `led_ledger_type` | **PARTY**. Payment's payee list (dropdown 60) and Receipt then offer it with no new code. |
| `led_is_bill_by_bill` | false: advances and recoveries run on account. |
| company / branch | `emp_company_id` / **NULL**, so a transfer between branches does not strand the balance. |

**Behaviour:**
- **Pick, not create:** an employee whose ledger came from Tally can be given an existing ledger. The server accepts it only from the Loans & Advances group.
- **Rename:** editing the employee's name renames the ledger, but only while it still carries the generated name.
- **Delete or resign:** deleting an employee is refused while the ledger has a balance. Resigning leaves the ledger active until it is settled.
- **Existing employees:** they get their ledger the first time they are saved. The note also asks for a one-off backfill.

### Where the ledger is used

| Event | Voucher | Dr | Cr |
|---|---|---|---|
| Salary or staff advance paid | Payment (PmtV) | Staff advance | Cash / Bank |
| Till shortage, decided RECOVER | TVar | Staff advance | Till cash |
| Staff repays (cash or salary cut) | Receipt (RcpV) or Journal | Cash / Salary payable | Staff advance |

The **balance** on the ledger is what HR recovers. There is no payroll link, per decision 2.

## 3 · Screens

Mockup: `till_user_employee_ui_mockup.png` (picture 5 of `till_masters_ui_mockups.py`). It replaces
`till_user_till_tab_ui_mockup.png`, which has been removed.

**User Administration, Identity tab:**
- **Employee** (new on screen): dropdown 38, filtered by the user's branch. The API already takes `usrEmployeeId`.
- **Till PIN** (new): a set/reset/clear button with a "SET · date" pill. This needs a server route (notes 95).
- **On the till** (read-only card):
  - the Open Till right,
  - the staff advance ledger and its balance,
  - approval grants, with a link to Approval Authority,
  - the live session,
  - the last 30 days.

  It is shown once the till screens exist; nothing in it is stored on the user.

**Employee Entry, Salary & Allowances:**
- **Staff advance ledger** (new on screen): blank on a new employee means it is created on save. Otherwise pick an existing ledger.
- A read-only group/balance strip.

**Removed:** the "Till Operator" master (S10) and the User master's "Till" tab.

## 4 · Server checks (NestJS; no DB function)

| When | Check | Error |
|---|---|---|
| opening a session | caller holds the Open Till right | 403 `TILL_NOT_ALLOWED` |
| RECOVER decided | operator user → employee → `emp_loan_ledger_id` all present and active | 409 `TILL_LEDGER_UNMAPPED`, naming the cashier |
| approving at the counter | approver has a PIN set and the PIN matches; 5 failures in 10 min locks the PIN | 401 `TILL_PIN_INVALID` / 423 `TILL_PIN_LOCKED` |
| user save | `usrEmployeeId` exists, is active and belongs to the user's company; not linked to another active user | 400 / 409 on `usrEmployeeId` |
| employee save | `empLoanLedgerId`, when given, is a live ledger in Loans & Advances (Asset) | 400 on `empLoanLedgerId` |
| employee delete | staff advance ledger balance is zero | 409 `EMP_LEDGER_HAS_BALANCE` |

## 5 · Verified

- 47 → 48 on a fresh throwaway PG 18 (the live schema stubs): **24 accounts `till%` relations**, including partitions, which is one master fewer than before. `accounts.till_operator` does not exist.
- 47 → 48 → 47 → 48 re-run is clean. Two seed guards were fixed while here: re-running 47 after 48 used to re-seed the PAID_OUT reason and rule that 48 had renamed to EXPENSE. The check failed on the reason. Now 20 rules and 45 reasons, with no duplicates.
- The behaviour probes still give the 10 expected refusals.

## 6 · Setup before go-live (data, not code)

Every cashier and approver needs:
- a user linked to an employee,
- a staff advance ledger on that employee (automatic once the server change is live),
- a PIN for every approver.

On the test box today, 14 users have no employee and no PIN, and none of the 3 employees has a ledger.
