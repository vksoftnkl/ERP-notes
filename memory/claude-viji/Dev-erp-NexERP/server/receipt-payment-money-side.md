---
name: receipt-payment-money-side
description: "notes (56) 2026-09-28: Receipt/Payment cash-bank side is enforced by sideVerdict() keyed on vchr_nature (RECEIPT DR = money only, CR never; PAYMENT mirror); ledger-pick applies the same filter; RcpV/PmtV group lists now hold the three money groups"
metadata:
  type: project
---

Built 2026-09-28. `sideVerdict(type, side, ledger)` in voucher-derive.ts returns
OK | GROUPS | MONEY_ONLY | NO_MONEY; `legalOnSide` is `=== 'OK'`. The nature rule speaks
BEFORE the group list, so a refusal on RcpV DR reads "the DR side takes cash or bank only"
and on RcpV CR "cash to bank is a Contra" (PmtV: "bank to cash is a Contra"). Cash/bank =
`isMoneyLedger`: groupPath holds one of the three RESERVED ids (`MONEY_GROUP_IDS` in
voucher-facts.ts) or a group name matches. `ledgerPick` repeats the rule in SQL with a
recursive `money` CTE. Migration 20260928170000 set PmtV cr-groups and RcpV dr-groups to
Cash-in-Hand / Bank Accounts / Bank OD A/c (applied on localhost).

**Why:** Dr Cash / Cr Bank filed as a receipt broke the receipt register and the
day-close cash figure; the old group lists only guarded one side of one type.

**How to apply:** Con and Jrl have no money side — never route a deposit through RcpV.
A fixture type with `nature: 'RECEIPT'` in voucher-derive.spec.ts must put a money
ledger on DR whatever its drGroups say. Tests: derive spec "notes (56)" block, http
e2e ledger-pick + validate cases in test/vouchers-register-http.e2e-spec.ts.
