# Receipt — what is left on the backend

**2026-09-18.** Everything below was checked LIVE against 192.168.0.106 today,
not read off a plan. Two sources: the five API gaps from the 2026-09-17 route
test, and the 3.0 feature comparison in `gap-vs-3.0.md` (revision 2).

**Short answer: three of the five old gaps are CLOSED. Two are still open, and
both are one-line fixes. Beyond those, six of the 3.0 items need you before the
client can start; the rest are client-only.**

---

## 0. Closed since 2026-09-17 — nothing to do, recorded so it is not re-raised

| | Was | Now |
|---|---|---|
| `open-items` had no TCS | §4.1 wanted `tcsAmount` + `tcsPending` from `accounts.v_bill_tcs` | **Both present.** `v_bill_tcs` exists and the bills carry them. The bill-wise TCS column is now a CLIENT job. |
| `update-header` edited a DRAFT | documented as posted-only, took a draft anyway | **404 "No receipt … in 2026-2027" on a draft.** Posted-only, as documented. |
| `ck_avh_balanced` was vacuous | `avh_total_debit/credit` were stamped by the writer, so a voucher with no legs passed | **`tr_av_refresh_totals` → `fn_avh_recompute_totals` is deployed** on `acc_vouchers` and all four year partitions, and derives both totals from the legs. Verified: every POSTED voucher created after ~20:10 on 2026-09-17 matches its legs exactly. |

**One tidy-up from that last one.** Three receipts posted in the ten minutes
BEFORE the trigger went on still carry stamped totals that disagree with their
legs — `rct00049` (header 500, legs 1500), `rct00050` (1200 / 1700),
`rct00052` (600 / 1200). Not a defect, just rows that predate the fix. A
backfill closes it:

```sql
SELECT accounts.fn_avh_recompute_totals(avh_voucher_id, avh_acc_year)
  FROM accounts.acc_voucher_header
 WHERE avh_voucher_status = 'POSTED';
```

---

## 1a. CLOSED 2026-09-18 — cancelling a receipt left its PDC voucher posted

**FIXED AND VERIFIED.** Cancelling a receipt with a post-dated cheque now
reverses both vouchers ("2 voucher(s) reversed") and every ledger touched nets
to 0.00. Kept here only so the reasoning is on record.

**The pre-existing orphan rows are NOT a work item** — 192.168.0.106 is a test
database, and findings about rows on it never are. The finding was the
behaviour. Also, for anyone who queries for them later:
`avh_against_voucher_id` means BOTH "PDC child of" and "reversal of", so a
naive query counts reversal vouchers as orphans and overstates the number.

The original report follows.

A receipt with a post-dated cheque writes TWO vouchers (R2): the receipt, and
the PDC voucher dated the cheque, linked by `avh_against_voucher_id`.
Cancelling the receipt **reverses only the receipt.** The PDC child stays
`POSTED`, its legs stay live in `acc_vouchers`, and no reversal voucher is
written against it.

And it cannot be cleaned up from outside, because `/receipts/cancel` on the
child refuses with:

> *"This is a post-dated cheque voucher, not a receipt. Cancel the receipt it
> belongs to and both are reversed together."*

which is exactly what did not happen. So the child is unreachable by any route.

**Reproduced 4 times out of 4 today** through `/receipts/cancel`, cancelling
rct00064 / 65 / 66 / 70 — each left rct00067 / 68 / 69 / 71 POSTED with two
live legs apiece. **It is not only my test data**: across the whole box,

```sql
SELECT count(*), SUM(c.avh_total_debit)
  FROM accounts.acc_voucher_header c
  JOIN accounts.acc_voucher_header p ON p.avh_voucher_id = c.avh_against_voucher_id
 WHERE p.avh_voucher_status = 'CANCELLED' AND c.avh_voucher_status = 'POSTED';
```

returns **9 orphaned PDC vouchers holding 5,400.00**, five of which predate
today's testing.

**What it should do:** cancel should reverse the child in the same
transaction it reverses the parent — the refusal message already promises
that. **This is what was done.**

---

## 1. Still open — please fix these two first

**R-B1. `/receipts/regularise-pdc` has NO tenant scope at all.** `RegularisePdcDto`
is still `{ asOf? }` and nothing else. Any authenticated user regularises
post-dated settlements across **every company in the database**. It is the only
mutating route on the module without the house four keys (`companyId`,
`branchId`, `accYear`, and the voucher key where one applies) — see
`api-house-conventions`. Effect is idempotent, so nothing is corrupted today,
but this is a cross-tenant write and should not ship.

*Also:* it reports "1 bill(s) regularised" on a second run over the same data.
It is counting rows CONSIDERED, not rows CHANGED. Please report rows actually
moved, or the operator cannot tell a real run from a no-op.

**R-B2. `/receipts/update-header` accepts TWO collectors; `/create` refuses
them.** `UpdateReceiptHeaderDto.avhEmployeeId` is an unbounded array. Verified
today: a PUT with two ids returned 200 and wrote both to `avh_employee_id`,
while `/create` refuses the identical payload with *"A receipt is collected by
ONE person."* One of the two rules is wrong; `/create`'s is the one the plan
and the screen follow. (I reverted the probe — `rct00049` is back to one.)

**R-B3. `party-context` 200s for a party that does not exist**, returning empty
lists, where `open-items` 404s on the same id. Low stakes but they should agree,
and the 404 is the useful one: the client cannot tell "no history" from "wrong
id".

---

## 2. New work — what the 3.0 comparison needs from you

From `gap-vs-3.0.md` rev 2. **Only these six need backend; everything else on
that list is client-only, so do not wait on us.**

**R-B4. Previous / next voucher.** *(gap A1 — the biggest single item.)* 3.0
walks the day's receipts on Ctrl+PgUp / Ctrl+PgDown (`prevTransId` /
`nextTransId`). We have no equivalent, so every reopen is a search. Needs one
route:

    GET /receipts/adjacent?voucherId&companyId&branchId&accYear&direction=prev|next

returning the neighbouring voucher key in the register's own order (date, then
voucher no), honouring the same status filter the register uses. An ordered id
list would also do, but the route is cheaper for the client.

**R-B5. The deposit account must be choosable per tender row.** *(gap A5 —
blocking with more than one bank account.)* 3.0's tender delegate lets a row
override the ledger the money lands in, gated by the tender master's
`edit_ledger` flag. We take `tnd_ledger_id` from the master and never offer it,
so **an operator cannot say which bank a cheque is banked into.** Needs:
- `edit_ledger` (or equivalent) exposed on the tender master DTO, so the client
  knows which rows may override;
- `/create` to accept and honour a per-row ledger override on `tenders[]`
  instead of always resolving `tnd_ledger_id`.

**R-B6. Duplicate-receipt guard.** *(gap B10.)* 3.0 asks, on every tender
amount edit, whether this party already has a receipt for this amount on this
date, and makes the operator confirm. On a beat run that is the only thing
between a re-key and a double receipt. Needs a cheap read:

    GET /receipts/duplicate-check?partyId&companyId&branchId&accYear&voucherDate&amount&excludeVoucherId

answering with the matching receipts (refno, date, amount) — a WARNING for the
client to raise, never a refusal.

**R-B7. Profit left after the settlement discount.** *(gap B12.)* 3.0 carries
`sale_profit` per bill and shows profit and profit-after-discount, recomputed as
the discount is typed. It is what tells the operator whether the discount they
are about to give costs the margin. `open-items` `bills[]` carries no such
field. Needs `billProfit` (and we compute net ourselves). **Please say whether
`sale_profit` is even derivable here** — if the margin is not stored per bill,
this dies and we drop the item rather than guess.

**R-B8. The customer's own reference, per bill.** *(gap C17.)* 3.0 shows
`user_refno` on each bill row and sends it back on the allocation. It is how an
operator matches a customer's remittance advice to our bills. `open-items`
`bills[]` has `docRefno` (OUR reference) only — `usrRefno` exists on the receipt
HEADER but not per bill. Needs the bill's own `usrRefno` on `open-items`.

**R-B9. Party totals the band needs.** *(gaps C14 / C16.)* `party-context`
gives us last receipts and pending cheques. Two figures 3.0 shows and we cannot
compute client-side from what we are sent:
- **total balance** for the party across everything, beside the bill-wise
  outstanding we already show;
- **total of cheques still out** (3.0 sums each bill's `cheque_amount`).

We can sum `pdcHeld` over the bills we are given for the second one, so it is
only truly owed if that list is ever paged. The first is not derivable.

---

## 3. Not yours — listed so the split is clear

These came out of the same comparison and are **client-only**; I am not waiting
on anything for them: auto-adjust as a mode rather than one-shot (A4), Save&New
/ Save&Print (A2), Clear (A3), percentage entry in the discount cell (B11),
per-bill payment history (B6 — reads existing data), card capture and surcharge
(B7 — DTO already carries the fields), WhatsApp (B8), audit-log view (B9 — data
exists), Place on the party band (C13 — already in the payload), and the
bill-wise TCS column (now unblocked by §0).

---

## 4. Older items still not deployed, for the record

Neither blocks the receipt screen today, both were in the original plan:

- **`fn_abl_recompute` / `tr_abj_refresh_balance` (schema 09) are not on the
  box** — there is no function matching `abl_recompute` and no trigger on
  `acc_bill_adjustment`. The service recomputes allocation from the adjustment
  rows instead, which is why every OPENING bill is restated on its first
  receipt. Still worth closing at source.
- `/receipts/amend` and `/receipts/delete` are **new since the last route test**
  and have NOT been exercised yet. I will test them next pass unless you would
  rather I left them alone.
