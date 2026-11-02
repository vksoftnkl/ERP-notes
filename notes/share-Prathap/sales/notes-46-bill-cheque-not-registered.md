# notes (46) — a CHEQUE tender on a sale bill never reaches the cheque register

2026-09-25. Found by the user: bil00696 (Freshora, 2,000, cheque KLOPA,
American Express Banking Corp, dated 2026-09-25) — nothing on the Cheques
screen (menu 51) and nothing on the receipt's pending-cheques panel.

## What the box holds for bil00696

* `acc_tender_detail` row `01a0d708-7108-777c-a567-bac37c570e01`: type 5
  CHEQUE, 2,000, ref KLOPA, bank, instrument date — all present.
* Voucher legs: DR **Cheques In Hand** 2,000 / CR Freshora 2,000 — posted.
* `acc_pdc_register`: **no row** (`apd_tender_id` = that td_id → 0 rows).

## Why

`plan-backend-sales.md:401` books CHEQUE like CASH/UPI (DR tender ledger /
CR party) and stops. Nothing in `modules/sales/bill/` writes
`acc_pdc_register`. The sale ORDER does — `order-pdc-posting.helper.ts`
registers every type-5 tender as HELD / ON_RECEIPT with the voucher named,
and cancels the row when the tender goes away. The bill needs the same.

Consequences today: the cheque cannot be deposited, cleared or bounced; the
bill reads fully PAID although the money is still a piece of paper; and
`/bills/retender`'s `SALES_RETENDER_PDC_MOVED` guard can never fire because
there is no register row to have moved.

## Ask

1. **On post** — for each type-5 tender: one `acc_pdc_register` row, tra
   type R, HELD, ON_RECEIPT, `apd_voucher_id` = the bill's voucher,
   `apd_tender_id` = the td row, party = bill party, instrument no / date /
   bank / drawer from the tender row (same mapping as the order helper —
   reuse it).
2. **On cancel / amend** — HELD row → CANCELLED (reason); DEPOSITED / CLEARED /
   BOUNCED → refuse (409), the way the order helper and the receipt unwind do.
   On amend the re-post writes the new row; a td row that keeps its id should
   keep its register row.
3. **Re-tender** — voiding a cheque row cancels its register row; a new cheque
   row registers one. `pdcMoved` then means something.
4. **Bounce** — the C4 cascade finds the bills a cheque settled through
   `acc_bill_adjustment.abj_tender_id`. A bill's own tender writes no
   adjustment row (it settles inside the bill's voucher), so a bounced bill
   cheque would NOT re-open the bill's balance. Decide how the bounce finds
   it: either the bill writes a tender allocation row, or bounce also looks
   up `apd_tender_id → td_src_doc_id` for SALE_BILL.
5. **Backfill** — existing posted bills with type-5 tenders and no register
   row (bil00696 at least):

       select b.sb_bill_refno, t.td_id, t.td_amount, t.td_ref_no
         from accounts.acc_tender_detail t
         join sales.sale_bill b on b.sb_id = t.td_src_doc_id and b.sb_acc_year = t.td_acc_year
        where t.td_src_doc_type = 'SALE_BILL' and t.td_tender_type_id = 5
          and not t.td_is_deleted and b.sb_status = 'POSTED'
          and not exists (select 1 from accounts.acc_pdc_register p
                           where p.apd_tender_id = t.td_id and not p.apd_is_deleted);

No client change is needed: the screen already sends the instrument number,
bank and date on the tender row.

## VERIFIED 2026-09-25 (after deploy) — all five items work

| check | result |
|---|---|
| backfill | bil00696 KLOPA 2,000 → HELD, ON_RECEIPT, voucher = the bill's, party ok; 0 unregistered cheques left |
| post (bil00697) | HELD |
| amend rev 1→2 | old row CANCELLED, new row HELD (new td row, so a new register row — fine) |
| cancel | both CANCELLED |
| deposit → bounce (bil00700) | DEPOSITED → BOUNCED; `billsReopened` names bil00700; bill 315 OPEN again |
| Cheques In Hand ledger vs register (notes 47 C2) | 9,315 = 9,315 |
| ZT-CUST-CASH ledger vs bills (C1) | 740 = 740 |

One cosmetic point: `billsReopened[].settledByThisCheque` comes back as
**-315** — negative for "this cheque settled 315". Positive would read right.
