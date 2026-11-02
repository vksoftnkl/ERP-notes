# Notes 128: the counter rule for the tender check (answer to your notes-126 question, 2026-10-16)

**STATUS:** decided by the user. Read with notes 126. This answers your question "Counter rule": where a money line with no tender is refused.

## 1 · The decision: the counter is never refused

| Where | No tender on a CASH / BANK line |
|---|---|
| **Counter documents:** sale bill, sale return, re-tender, order advance | **Never refused.** For a cash line the server writes the CASH tender row itself. Any other gap **posts anyway** and shows on the "money lines with no tender" report (notes 126 §4.2). |
| **Accounts vouchers:** Voucher Register, Expense, Payment, Receipt, till transfers (TFlt / TDrp / TRcv / TVar / CRem), cheque routes, settlement, reversal mirrors | **Refused:** `VCH_MONEY_LINE_NO_TENDER` (field `lines.N`) / `VCH_NO_TENDER_FOR_LEDGER` (naming the ledger), as in notes 126 §1. |

**Why:** the user's rule is that the counter is never blocked while a customer waits (same as notes 123 / 124 §5). A bill always carries its tender anyway: the cashier picks Cash / UPI / Card / Credit first. So a gap on a bill can only come from a wrong tender master, and the report shows it the same day. Nobody waits at an accounts voucher, so refusing there is safe. That is where the real gaps came from (`rcv00140`, `rcv00146`).

## 2 · Stop a wrong setup at the tender master instead (please add)

Under §1 a wrong tender master (e.g. CARD → Kvb) keeps posting the right tender to the wrong ledger. Catch it once, when the master is saved, so it never reaches the counter:

- **CARD / UPI / WALLET:** refuse a ledger whose `led_ledger_type` is BANK or CASH. It must be a clearing ledger.
- **CASH:** refuse a ledger that is not CASH type.
- **BANK / CHEQUE:** the ledger must be BANK type (CHEQUE received → Cheques In Hand, as today).
- **CREDIT / TEMP_CR / LOYALTY:** no ledger (notes 126 §5).
- Give each refusal a `code` and `field: 'tndLedgerId'`. Send us the codes so we can write the plain wording.

**Do the notes-126 §5 clean-up first** (Card Clearing, Wallet Clearing, no ledger on TEMP.CR / CREDIT / LOYALTY). Otherwise the new save check blocks editing the existing CARD / WALLET rows.

## 3 · Till transfers: don't count cash twice

Notes 126 §3 adds tender rows to till transfers. Every reader that today adds up CASH tender rows will then also see the transfers: drawer expected, safe book, 40A(3), day Z, Business Day, Account Day Summary, the reports. **Please make each reader filter by source/voucher type** (sales-side rows only for drawer expected; transfer rows only for the safe path), and tell us which readers you changed. We will recheck the Cockpit, Business Day and day Z figures against before.

## 4 · "Cashin Hand"

You found it is a duplicate (no role, no live safe, 9 legs on deleted test safes). Mark it inactive, as notes 126 §5 asked. The 9 legs are test data, so no repost is needed.

Tell Viji when it is deployed. Claude will recheck live: a bill with a gap posts and lists on the report, a register voucher with a gap is refused, a CARD tender saved on Kvb is refused, and the till figures stay the same.
