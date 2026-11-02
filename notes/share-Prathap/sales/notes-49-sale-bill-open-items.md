# notes (49) — sale bill: what is still open after the 25-Sep full retest

2026-09-25. The Qt sale bill passed a full live retest today:
`tests/salebill/run-full.sh` **156 pass / 0 fail / 3 skip**, plus the cheque,
bounce, set-off, partial-payment, amend and hold scenarios, all green. The
books reconcile (`accounts.fn_books_reconcile`) on every ZT party and on
Cheques In Hand (31,000 = 31,000).

These are the items still open. **All are server-side.** Items 1 and 2 are
blocking: with `accounts.reconcile_on_post` ON, every new post for
SATHEESH or Freshora is refused until item 2 is done.

---

## 1 · D3 — cancelling a bill burns an invoice number  (BLOCKING for GST)

The cancel's mirror (reversal) voucher is voucher type 3 and takes its refno
from the **sale-bill series**, so the next bill skips a number. The GST
invoice series must be continuous; every gap needs an explanation in GSTR-1.

Scale today: **214 vouchers** carry a `bil…` number that no sale bill has
(bil00027 … bil00776, 22–25 Sep; many are the test suite's own cleanup
cancels). The suite reports it on every run:
*"cancel used up 1 bill number(s) — the mirror voucher is type 3 and draws on
the bill series"*.

**Ask:** give the reversal its own number, e.g. the original's refno plus a
suffix (`bil00687/CN`, like re-tender's `/RT<n>`) or a separate reversal
series. It must not advance the sale-bill counter.

**Check:**
```sql
-- after the fix, cancel one bill and this must return 0 new rows
select h.avh_voucher_refno, h.avh_created_on
  from accounts.acc_voucher_header h
 where h.avh_voucher_refno like 'bil%'
   and not exists (select 1 from sales.sale_bill b where b.sb_bill_refno = h.avh_voucher_refno)
 order by h.avh_created_on desc;
```
Also say what to do with the 214 existing ones (renumber the mirrors, or
record them as cancelled-number gaps). The box is test data; the rule is what
matters for go-live.

---

## 2 · D1 — repair two bill-balance rows written before your fix  (BLOCKING)

Your D1 fix is verified for NEW bills (bil00718: 315, cash 100 → alloc 100,
pending 215). Two rows written before it still leave out the bill's own
counter payment:

| bill | bill amt | own counter tender | receipts since | abl_alloc now | should be | abl_pending now | should be |
|---|---|---|---|---|---|---|---|
| bil00685 (SATHEESH) | 4,000 | cash 3,768 (+ credit 232) | 5 (rct00055) | 5.00 | **3,773.00** | 3,995.00 | **227.00** |
| bil00695 (Freshora) | 800 | cash 100 (+ temp credit 700) | 150 (rct00056) | 150.00 | **250.00** | 650.00 | **550.00** |

abl ids: bil00685 → `01a0d3e5-e046-7303-af8e-95e4cbe026a7`,
bil00695 → `01a0d705-c389-789b-a76b-cbdf5f583327`.

Please repair them the way your D1 fix writes a new bill (the allocation
entry for the counter tender), not by editing the totals by hand, so the
allocation trail matches. The TEMP_CR balance on bil00695 then follows
`abl_pending_amount` (550), as it should.

**Check:**
```sql
select check_kind, ledger_name, ledger_bal, other_bal, diff
  from accounts.fn_books_reconcile('019c8ea6-19e9-78a8-b15f-749e1cde7292', '2026-2027')
 where abs(diff) > 0.005;          -- must return 0 rows
```
(Freshora's opening-sign part, D2, is already fixed; its remaining −100 is
exactly bil00695's cash.)

---

## 3 · C4 — the bill header must agree with its own rows  (please confirm)

Asked in notes 47, addendum 2. I can't tell from the box whether it went in.
bil00702 was accepted with header `sbTenderAmt` 4,000 but tender rows
totalling 3,000. Please confirm whether create / validate / post / amend now
refuse a header whose tender, advance, paid or balance figures don't match its
rows, and **which tender column `sbTenderAmt` is meant to equal**
(td_amount / td_received_amt / td_total_amt, with change and surcharge in or
out).

---

## 4 · notes 45 — amend of a bill that EARNED loyalty points  (please confirm)

bil00687 (220 points) answered 500 on amend. From the 24-Sep source, the
re-post's `loyalty.earn()` writes EARN row_no 1 while the original EARN row 1
is still live → `ux_lld_src_row`. The suite cannot re-test it:
`loyalty.enabled` is false for the test company. Please confirm it is fixed.
If you can't tell, switch loyalty on for Acme Foods for 10 minutes and I'll
run the amend.

---

## 5 · Cosmetic — cheque bounce reply sign

`/cheques/bounce` → `billsReopened[].settledByThisCheque` comes back as
**-315** for a cheque that settled 315. Positive would read right; the Qt
bounce preview shows it.

---

When 1 and 2 are done, tell me and I'll re-run the suite and the
reconciliation. Nothing in this note needs a client change.

---

## VERIFIED 2026-09-25 17:50 (after your run)

| # | item | result |
|---|---|---|
| 1 | cancel burns a bill number | ✔ **fixed** — bil00882's cancel posted `rev00025` (type 23); the next bill was bil00883, 0 new orphan `bil…` refnos. The 224 old gaps (all before the fix) are still there — say what you want done with them for GSTR-1. |
| 2 | old bill-balance rows | ✔ **fixed** — bil00685 alloc 3,773 / pending 227, bil00695 alloc 250 / pending 550; `fn_books_reconcile` returns **0 rows** company-wide |
| 3 | C4 header vs rows | ✘ **not in** — `/bills/validate` with header `sbTenderAmt` 500 and one cash tender row of 118 answers `ok: true` (same as the consistent 118/118). If /create recomputes the header from the rows, say so and this is only a validate gap; otherwise please add the check. |
| 4 | loyalty amend 500 | not testable here (`loyalty.enabled` false) — please confirm, or switch it on briefly |
| 5 | bounce reply sign | ✔ **fixed** — `settledByThisCheque: 315` |
