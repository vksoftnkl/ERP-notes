# notes (47) — trial-mode check: bill-wise balance = ledger balance, on every post

2026-09-25. Decided with the user: while we are in trial, every post that
touches a **bill-by-bill party** (`acc_ledger_master.led_is_bill_by_bill`)
checks that the party's open bills add up to its ledger balance, opening
included. A mismatch REFUSES the post, so the error surfaces at the counter
instead of at year end.

## The rule

For each bill-by-bill ledger the post wrote to (usually one), company-wide:

    LEDGER = opening (acc_opening_balance, this acc_year, DR +, CR −)
           + Σ av_signed_amount of live legs (av_is_deleted = false)
             whose header is NOT DRAFT      ← POSTED **and CANCELLED**
    BILLS  = Σ abl_pending_amount (DR +, CR −), abl_is_deleted = false, ALL years

    |LEDGER − BILLS| > 0.005  →  refuse

* **CANCELLED headers count.** A cancel keeps the original voucher's legs live
  and posts a mirror reversal (POSTED); excluding the CANCELLED original
  leaves every cancelled bill as a false mismatch. Verified: with them in,
  ZT-CUST-CREDIT (102 bills), ZT-CUST-CASH and MADHAVAN reconcile to 0.00.
* **All years on the bill side**, one year (+ its opening) on the ledger side.
* **Company-wide**, not per branch — the ledger balance is not branch-scoped.
* Only `led_is_bill_by_bill = true`. Running-balance ledgers have no bills.

## Where

* One DB function, e.g. `accounts.fn_party_bill_reconcile(company, ledger,
  acc_year) → (ledger_bal, bills_bal, diff)`, so the post and the nightly
  report use one definition.
* Called at the END of the post transaction (after every write) by: sale bill
  post / cancel / amend / re-tender, receipt post / cancel / amend, cheque
  deposit / clear / bounce / re-present, opening-balance save — anything that
  writes `acc_vouchers` or `acc_bill_balance` for a party.
* Behind a company setting, e.g. `accounts.reconcile_on_post` (default ON
  during trial, OFF after go-live — offline sync can make the two sides lag
  briefly in production, and a check must never lock a contended resource;
  see the offline-sync rule).
* Refusal: 422 `ACC_PARTY_OUT_OF_BALANCE`, details
  `{ partyId, partyName, ledger, bills, diff }`. The Qt strip shows it like
  any other refusal — no client work beyond that.
* Also a whole-company pass (same function over every bill-by-bill ledger),
  run nightly during the trial: catches drift from paths the per-post call
  misses.

## Dry run on the box today — 3 defects it would have caught

Query: `recon.sql` below. 15 bill-by-bill ledgers; 12 reconcile; 3 do not
(one of them only because of the CANCELLED rule above — now fixed in the rule).

**D1 — a bill's own PARTIAL counter payment is not in its bill balance.**
SATHEESH bil00685: bill 4,000, voucher credits the party 3,768 (CASH tender),
but `abl_alloc_amount` = 5 (only rct00055) → pending 3,995; ledger says 227.
Freshora bil00695: 800, CASH 100 + TEMP_CR 700 → pending 650 (800 − 150
receipt), the 100 cash missing. Fully-paid bills close correctly (bil00600,
bil00696). So the partial path skips writing the tender as settled on the
bill's balance row.

**D2 — opening sign disagrees.** Freshora: `acc_opening_balance` CR 98, but
the opening bill "89" is DR 98 → diff 196. The opening save must keep the
ledger opening = NET of the party's opening bills (the opening-balance rule).

**D3 — cancelling a bill burns an invoice number.** The cancel's mirror
voucher takes its refno from the SALE BILL series: bil00687's reversal is
`bil00694`, bil00595's is `bil00596`; no sale bill 694 / 596 exists. Every
cancel leaves a gap in the GST invoice series. The reversal needs its own
series (or the original's refno + a suffix, like re-tender's `/RT<n>`).

## recon.sql (read-only)

```sql
with led as (select led_id, led_name from accounts.acc_ledger_master
              where led_is_bill_by_bill and not led_is_deleted),
op as (select op_ledger_id lid, sum(case when op_dr_cr='DR' then op_amount else -op_amount end) amt
         from accounts.acc_opening_balance
        where not op_is_deleted and op_acc_year='2026-2027' group by 1),
mv as (select v.av_ledger_id lid, sum(v.av_signed_amount) amt
         from accounts.acc_vouchers v
         join accounts.acc_voucher_header h on h.avh_voucher_id=v.av_voucher_id and h.avh_acc_year=v.av_acc_year
        where not v.av_is_deleted and not h.avh_is_deleted and v.av_acc_year='2026-2027'
          and h.avh_voucher_status in ('POSTED','CANCELLED') group by 1),
bb as (select abl_party_id lid, sum(case when abl_dr_cr='DR' then abl_pending_amount else -abl_pending_amount end) amt
         from accounts.acc_bill_balance where not abl_is_deleted group by 1)
select l.led_name, coalesce(op.amt,0)+coalesce(mv.amt,0) ledger, coalesce(bb.amt,0) bills,
       coalesce(op.amt,0)+coalesce(mv.amt,0)-coalesce(bb.amt,0) diff
  from led l left join op on op.lid=l.led_id left join mv on mv.lid=l.led_id left join bb on bb.lid=l.led_id
 order by abs(coalesce(op.amt,0)+coalesce(mv.amt,0)-coalesce(bb.amt,0)) desc;
```
(The real function takes company_id too — this box has one company in play.)

## Addendum (same day) — one more check, same setting, same 422 shape

### C2 — Cheques In Hand = the register  (cheque / receipt / sale bill posts)

For each Cheques-In-Hand ledger the post touched (the tender row's
`td_tender_ledger_id`):

    LEDGER   = Σ av_signed_amount, live legs, headers POSTED or CANCELLED
    REGISTER = Σ apd_amount where apd_status IN ('HELD','DEPOSITED')
               and apd_posting_mode = 'ON_RECEIPT' and not apd_is_deleted
               (DEPOSITED counts: a deposit posts no voucher; ON_CLEARING
               cheques have no voucher until they clear, so they are out)

Dry run today: ledger 9,000 vs register 7,000 → **diff 2,000 = bil00696's
cheque that never reached the register (notes 46)**. Caught to the rupee.
Called by: receipt post / cancel / amend, every /cheques verb (deposit, clear,
bounce, return, re-present, replace), sale bill post / cancel / amend /
re-tender, sale order advance.
Refusal code e.g. `ACC_CHEQUES_OUT_OF_BALANCE`, details `{ ledgerId, ledger, register, diff }`.

### C3 — dropped: Dr = Cr is already enforced

`tr_av_refresh_totals` → `fn_avh_recompute_totals` re-derives
avh_total_debit/credit from the live legs on every leg insert / update /
delete, and `ck_avh_balanced` refuses a POSTED header whose totals differ.
Nothing to add. (Only corner left: a POSTED voucher with NO legs is 0 = 0 —
not worth a check; 0 such vouchers on the box.)

Receipts need nothing beyond C1 (party) + C2: allocations, advances and
TCS all land on the party's bills and ledger, which C1 already compares.

## Addendum 2 (same day) — C4: a bill's header must agree with its own rows

Found by the user: bil00702 was amended with header `sbTenderAmt` 4,000 but
tender rows totalling 3,000 (and a 1,000 advance set-off still attached). The
server took it. C1/C2 cannot see this — the books agree with each other; it is
the DOCUMENT that disagrees with itself.

Ask: on create / validate / post / amend, refuse (422 `SALES_HEADER_MISMATCH`,
naming the figure) when

* `sbTenderAmt` ≠ Σ tender rows — **please define which row column** it is
  meant to equal (td_amount / td_received_amt / td_total_amt, and whether
  change and surcharge are in or out); today's data mixes them;
* `sbAdvanceAmt` ≠ Σ ADVANCE adjustments and `sbNoteAdjAmt` ≠ Σ note
  adjustments (believed enforced already — confirm);
* `sbPaidAmt` ≠ counter money + set-offs, `sbBalanceAmt` ≠ `sbBillAmt` − `sbPaidAmt`.

Posted bills on the box whose `sbTenderAmt` ≠ Σ td_total_amt today: 14
(bil00702, 675, 676, 600, 415, 385, 364, 359, 336, 331, 326, 321, 316, 155) —
several are the 24-Sep test runs' doubled rows; use them to pin the definition.

## VERIFIED 2026-09-25 (after deploy)

| item | result |
|---|---|
| fn_party_bill_reconcile / fn_cheques_in_hand_reconcile / fn_books_reconcile | live; ZT-CUST-CREDIT −27,716 = −27,716, ZT-CUST-CASH 740 = 740, Cheques In Hand 9,630 = 9,630 |
| `accounts.reconcile_on_post` | live, default true |
| D1 partial counter payment | **fixed for new bills** — bil00718: 315, cash 100 → alloc 100, pending 215 |
| D1/D2 on OLD rows | **not repaired** — SATHEESH −3,768 (bil00685), Freshora −296 (bil00695 cash 100 + opening sign 196). With the check ON, every post touching these two parties will now be REFUSED until the rows are repaired. Please backfill: bil00685 / bil00695 abl_alloc from their own tenders; Freshora opening bill "89" sign vs acc_opening_balance. |
| D3 cancel burns a bill number | **NOT fixed** — bil00712's cancel (11:39) took `bil00713`; bil00714 is gone too (bil00702's cancel, 11:41) |
| C4 header vs rows | not seen yet — please confirm whether it went in |

## Re-check 2026-09-25 evening (full sale-bill suite 156/0/3 + extras)
fn_books_reconcile: ZT parties 0.00, Cheques In Hand 31,000 = 31,000.
Still open: **D1 old rows** — SATHEESH −3,768 (bil00685); Freshora −100
(bil00695's cash 100; the opening-sign part D2 is fixed). **D3** — the suite's
finding again: "cancel used up 1 bill number(s) — the mirror voucher is type 3
and draws on the bill series".
