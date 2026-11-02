---
name: account-day-summary-module
description: "reports/day-summary (menu 284, also 181) built 2026-10-12; shares balance SQL with the Ledger Statement via reports/common; type filter nulls balances; day-wise uses a per-day pair rule"
metadata:
  node_type: memory
  type: project
  originSessionId: a9c46fef-191b-41d7-9a42-b241bd58e91a
  modified: 2026-10-12T05:37:12.952Z
---

Account Day Summary / day book, built 2026-10-12 from Prathap's plan
(plan-backend-account-day-summary.md). Seven GET routes under
/api/v1/reports/day-summary; menu 284 under 137 at 0.60 (migration
20261012120000_account_day_summary_menu + Menu_Master.sql); 181 "Day Book
Detail" accepted too (D8), left unchanged.

Non-obvious choices (README "Differences from the plan" has the full list):
- D1 done: countedLegsSql / openingSetSql / money / sided / isoDate live in
  src/modules/reports/common/voucher-books.sql.ts; the Ledger Statement was
  refactored onto them.
- A voucherTypeId filter makes money opening/closing null (a balance has no type).
- /daily pairs halves only on the same date, so each day's closing is the true
  balance; /summary pairs over the whole range.
- CLEARING = ledger of a company tender with tnd_settlement_ledger_id; on the
  dev box only UPI Clearing qualifies (tender master defect, plan §3.5 / D9).
- Every route runs in a READ ONLY REPEATABLE READ tx with force_custom_plan —
  see [[prisma-generic-plan-slow-reports]].

Tests: test/day-summary-http.e2e-spec.ts (25, incl. synthetic scenarios — see
[[synthetic-vouchers-rolled-back]]) and day-summary.groups.spec.ts.
Lone cancel halves never occur in real data: every mirror shares its
original's date and branch.
