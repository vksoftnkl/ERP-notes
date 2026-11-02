# Notes 95 — Till PIN on the user, staff advance ledger created with the employee

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-07 · **For:** Prathap
**Design:** `../till/plan-till-operator-to-user.md` · **Mockup:** `../till/till_user_employee_ui_mockup.png`

## Why

The till design had its own table of cashier profiles (`accounts.till_operator`). It has been dropped.
The till reads what already exists instead:

| What the till needs | Where it comes from |
|---|---|
| the approver's PIN | `user_master.usr_pin_hash` |
| who the employee is | `user_master.usr_employee_id` |
| where a recovered shortage goes | `employee_master.emp_loan_ledger_id` |

Two of these have no way in through the API today, and one has no data behind it. Nothing here changes
a table; the columns already exist.

## A. Employee save creates the staff advance ledger (same transaction)

**Today:** `employee-master.service.ts` accepts `empLoanLedgerId` as a bare uuid and checks nothing. The
column has no FK either. None of the 3 employees on the box has one.

**Wanted:** the same pattern as `/customers/create`, which writes the ledger row in the same call.

1. **Create or update with `empLoanLedgerId` absent or null** creates `acc_ledger_master` in the same
   Prisma `$transaction` and stores its id on the employee. The new ledger is:
   - **name:** `<empName> - Staff Advance`. If that name is taken in the company (`uq_led_name_company`), use `<empName> - Staff Advance (<empCode>)`.
   - **group:** `Loans & Advances (Asset)`, the shared reserved group under Current Assets. Look it up by name with `acc_group_company_id IS NULL`.
   - **`led_ledger_type` = 'PARTY'**, `led_is_bill_by_bill` = false, `led_category` = 'GENERAL'.
   - **`led_company_id` = `empCompanyId`, `led_branch_id` = NULL**, so a branch transfer does not strand the balance.
   - **mobile, email, address, PAN** copied from the employee, as the customer save copies them.
2. **`empLoanLedgerId` given** (an existing ledger, for example one imported from Tally): the ledger must
   - exist,
   - not be deleted,
   - sit in Loans & Advances (Asset) or a sub-group of it,
   - belong to the same company or be shared.

   Otherwise answer 400 on `empLoanLedgerId`.
3. **Name edit:** if the ledger still carries the generated name, rename it in the same transaction.
   A ledger someone renamed by hand is left alone.
4. **Delete:** refuse with 409 `EMP_LEDGER_HAS_BALANCE` while the ledger's balance is not zero. On a
   zero balance, soft-delete both rows.
5. **GET** also returns `empLoanLedgerName`, so the Qt dropdown can show the name on edit.
6. **The ledger delete guard** (notes 82) should also refuse a ledger that an employee points at:
   "This ledger belongs to employee ‹name›".
7. **One-off backfill:** create the ledger for every live employee with `emp_loan_ledger_id IS NULL`.
   A loop over the same service method is enough, with no SQL function.

## B. Till PIN on the user

**Today:** `usr_pin_hash` exists, but nothing in `userAdministration` or `auth` reads or writes it.

1. **The user-admin save accepts `usrPin`:**
   - 4 to 6 digits;
   - hashed the same way as the password (argon2/bcrypt, whatever `usrPasswordHash` uses) into `usr_pin_hash`;
   - `usrPin: ""` clears it;
   - an absent key leaves it untouched.
2. **GET never returns the hash.** It returns `usrPinSet: boolean` and `usrPinChangedOn` (if you want the
   date; the mockup shows "SET · 12-09-2026"). If that needs a column, skip the date.
3. **No public "verify PIN" route.** The PIN is checked only inside the till approval service, so it
   cannot be brute-forced from outside. That service should:
   - lock the PIN after 5 wrong tries in 10 minutes, reusing the `usr_failed_login_count` pattern;
   - log a `PIN_FAIL` event.

   This is listed now so the hash format is chosen once.
4. **GET also returns `usrEmployeeName`** (join `employee_master.emp_name`). The API already accepts
   `usrEmployeeId`, but the Qt dropdown shows a blank name on edit without it.
5. **Check `usrEmployeeId` on save:**
   - the employee exists, is not deleted, and is in the user's company (or `usrCompanyId` is null);
   - it is **not linked to another active user** (409 on `usrEmployeeId`).

   That second check is a service rule, not an index: two offline sites could each link one.

## C. Nice to have: a staff advance dropdown

The Qt employee screen uses dropdown 43 (LEDGERS - ALL) for now, because the `claude` role cannot
write `fixed.dropdown_details`. Please register this one when convenient:

```sql
-- STAFF ADVANCE LEDGERS (param icompany_id; always sent)
SELECT l.led_id, l.led_name, l.led_short
  FROM accounts.acc_ledger_master l
  JOIN accounts.acc_group_master g ON g.acc_group_id = l.led_group_id
 WHERE l.led_is_active AND NOT l.led_is_deleted
   AND (l.led_company_id IS NULL OR l.led_company_id::text = 'icompany_id')
   AND (g.acc_group_name = 'Loans & Advances (Asset)'
        OR g.acc_group_parent_id IN (SELECT acc_group_id FROM accounts.acc_group_master
                                     WHERE acc_group_name = 'Loans & Advances (Asset)'))
 ORDER BY l.led_name
```

## Acceptance

1. Create employee "ZT Ravi" with no ledger. The response carries `empLoanLedgerId`, and a ledger named
   "ZT Ravi - Staff Advance" exists in Loans & Advances (Asset) with type PARTY.
2. Payment (menu 100): the payee dropdown offers "ZT Ravi - Staff Advance". Pay 500 cash and post it;
   the ledger is Dr 500.
3. Rename the employee to "ZT Ravi K"; the ledger name follows. Delete the employee; the answer is 409
   `EMP_LEDGER_HAS_BALANCE`.
4. Save a user with `usrPin: "4321"`. GET shows `usrPinSet: true` and no hash. `usrPin: ""` gives `usrPinSet: false`.
5. Save a user with `usrEmployeeId` of an employee already linked to another active user; the answer is 409.
6. GET a user and an employee; `usrEmployeeName` and `empLoanLedgerName` are filled in.
