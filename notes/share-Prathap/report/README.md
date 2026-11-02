# report/ — reports

## Ledger Statement (ledger vouchers)

| file | what |
|---|---|
| `ledger_report_ui_mockup.png` / `.py` | the screen mockup (Vouchers / Daily / Monthly) |
| `plan-backend-ledger-statement.md` | NestJS plan: 7 NEW routes under `/api/v1/reports/ledger-statement/…`, no old URL reused |
| `plan-react-ledger-statement.md` | React plan: read-only, zero client money arithmetic, filters in the URL, only the new routes |

**Read §3 of the backend plan first:** two server defects found while writing it
(the Dr opening read as Cr in `fn_ledger_book_balance`; opening-balance closing
and carry-forward drop cancelled originals).

## Party-wise Outstanding (receivable / payable, ageing, as on a date)

| file | what |
|---|---|
| `party_outstanding_ui_mockup.png` / `.py` | the screen mockup (Party-wise tab, Sri Krishna Traders selected; ties to the ledger mockup's 1,76,300.00 Dr) |
| `plan-backend-party-outstanding.md` | NestJS plan: 9 NEW routes under `/api/v1/reports/party-outstanding/…`, one "pending as on D" definition, decisions O1–O7 |
| `plan-react-party-outstanding.md` | React plan: no money or date arithmetic on the client, filters in the URL, server sort/paging |

**Read §3 of the backend plan first.** Four things were found while writing it:
- counter tenders settle sale bills with no dated adjustment row (35 bills on the box);
- recompute may drop those tenders (suspected, test given);
- carry-forward leaves the parent bill pending, so other readers will double-count after 1 April;
- reversal rows carry the original date.

## Account Day Summary (day book — summary)

| file | what |
|---|---|
| `account_day_summary_ui_mockup.png` / `.py` | the screen mockup (Summary tab; Vouchers = day book detail, Day-wise = a range one row per day) — Main Store, 09-10-2026 |
| `plan-backend-account-day-summary.md` | NestJS plan: 7 NEW routes under `/api/v1/reports/day-summary/…`, `/summary` = one snapshot, menu 284 (+181 → Vouchers tab), decisions D1–D9 |
| `plan-react-account-day-summary.md` | React plan: no client money arithmetic, filters in the URL, ◀ ▶ day step, drills into vouchers / group ledgers / Ledger Statement |

**Read §3 of the backend plan first:** `avh_doc_amount` ≠ the posted total on 1,106 vouchers;
same-type cancel mirrors (Bil/Rct/Pmt/…) double-count unless detected; no money-ledger role
(cash/bank = group profile); card/wallet/RRN/CREDIT tenders post to the wrong ledgers; number gaps
in 17 of 30 series; 12 legless receipts.

2026-10-12: plans written, nothing built. Worked example reconciles:
Σ voucher-type amount = Σ group Dr = Σ group Cr = 4,95,270.00, and the cash / bank in/out
columns tie to the money-ledger grid. Drafts are counted, never summed; a cancelled pair nets 0.
