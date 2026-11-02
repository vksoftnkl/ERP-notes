---
name: account-day-summary-mockup
description: Account Day Summary (day book summary) 2026-10-12 in share/report/ — mockup + backend + React plans; nothing built, unreviewed
metadata:
  type: project
---
`/home/vk/Shared/Prathap/report/account_day_summary_ui_mockup.py/.png` + README section (2026-10-12). Not yet reviewed by the user (once approved → [[mockup-fidelity]]).

Layout: tabs Summary | Vouchers (day book detail) | Day-wise; filter Date with ◀ ▶ (3.0 daybook.* had the one-day step), To, Company, Branch, Type; 6 KPI tiles; VOUCHER-TYPE SUMMARY (count/draft/cancelled/amount/cash in-out/bank in-out); CASH & BANK POSITION per money ledger incl. card/UPI clearing; GROUP SUMMARY Dr/Cr/net; CHECKS (Dr=Cr, numbering gaps, drafts, cancelled pair, clearing pending, business day open → menu 274).

Rules shown: POSTED + CANCELLED headers summed (cancel nets with its mirror, as in [[ledger-report-mockup]]); drafts counted, never summed. Plans written 10-12: `plan-backend-account-day-summary.md` (7 routes under /reports/day-summary: voucher-types, summary = ONE snapshot, vouchers, daily, group-ledgers, checks/detail, export; menu 284 new + 181 → Vouchers tab; D1–D9) and `plan-react-account-day-summary.md`.
Key rules: in-range cancelled pairs LEFT OUT of figures (D2); amount = Σ DR legs, never avh_doc_amount; cash/bank = group chain acc_ledger_profile 'Cash'/'Bank'; clearing = tender ledger with tnd_settlement_ledger_id; group level REPORT = root's child (cash/bank/debtors are children of Current Assets).
Findings for Prathap (backend §3): doc_amount≠total_debit on 1,106; same-type cancel mirrors; CARD/WALLET/BANK/TEMP_CR tenders post straight to bank, RRN/CREDIT to SALES FREIGHT COLLECTION; gaps in 17/30 series; 12 legless Rct 09-29; 24 future-dated. 3.0 menu sibling: 181 Day Book Detail under 137.
