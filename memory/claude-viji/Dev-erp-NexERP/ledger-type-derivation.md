---
name: ledger-type-derivation
description: "how the ledger form derives led_ledger_type from the account group (profile is NOT enough — nature decides), and when the combo stays editable"
metadata: 
  node_type: memory
  type: project
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T12:09:21.041Z
---

Settled 2026-09-17 while reworking `LedgerEntry` (`src/modules/accounts/master/ledger/`).

**The group's `accLedgerProfile` alone CANNOT give the ledger type.** Verified
against the live group list:

- `SalesPurchase` is BOTH "Sales Accounts" (nature Income) and "Purchase
  Accounts" (nature Expenses).
- `General` is Direct/Indirect Incomes, Direct/Indirect Expenses, Fixed Assets,
  Current Liabilities — everything.

So the rule is **profile first, then nature**:
Bank→BANK, Cash→CASH, Party→PARTY, Tax→TAX; otherwise nature Income→INCOME,
Expenses→EXPENSE, else GENERAL.

The old profile-only mapping stamped `SalesPurchase → EXPENSE`, which is how
one ledger under Sales Accounts on the box carries EXPENSE.

**The combo is locked only for Bank/Cash/Party/Tax.** Under General and
SalesPurchase it stays editable, because ROUNDOFF and DISCOUNT ledgers live
under Indirect Expenses beside a plain EXPENSE one and nothing on the group can
tell them apart (both exist live). A fully-locked combo could not reproduce the
data that is already there.

**An edit must not re-derive.** `NexDropdownSingle::setSelection()` is silent,
so filling the group on load does not fire the derivation — keep it that way or
every edit of the ROUNDOFF ledger saves it as EXPENSE. The layout instead comes
from `ledGroupLedgerProfile`, which the ledger GET already carries (no second
call). `NexCrudForm::fillFromJson` was made **virtual** for this; it is the only
post-load hook.

Sections follow the group profile, widened once: an EXPENSE/INCOME ledger under
a *General* group still gets Supply Details + Tax Rate (Tally allows GST on
expense ledgers). ROUNDOFF/DISCOUNT do not.

See [[ledger-tax-id-change]] and [[acc-group-master-contract]].
