---
name: tcs-billwise-design
description: "TDS/TCS decided IN 2026-09-15 (S2 closed): accounts.tds_rates is a TABLE (we deduct in payment); bill-wise TCS — tcs_basis SALES vs RECEIPT are two different mechanisms; SALES needs ONE column abl_tcs_amount + view v_bill_tcs (no table), RECEIPT is a TCS_PAYABLE leg; 27EQ/26Q get projection tables; 50L threshold computed"
metadata:
  type: project
---

The user has clients who need **bill-wise TCS outstanding and collection**, and
those clients run on **`accounts.tcs_basis = 'SALES'`** (2026-09-15).

**The two bases are two mechanisms and never both apply:**

- **RECEIPT** (default, literal 206C(1H)): TCS is money ON TOP of the bill,
  raised by the receipt as a leg with `av_role = 'TCS_PAYABLE'`. Bill-wise
  collection is a VIEW over `acc_bill_adjustment` joined to `acc_vouchers` via
  `abj_voucher_line_id` — it works because the receipt writes one adjustment
  row per settling deduction per bill, pinned or pro-rata. **There is no
  outstanding** — TCS is not charged until the money arrives.
- **SALES**: TCS is INSIDE the invoice and inside `abl_bill_amount`; the
  receipt raises **no TCS leg at all**. Collection and outstanding are slices
  of the allocation. This is the case the schema could not answer:
  `acc_bill_balance` has no tax breakup (verified — only bill/alloc/disc/
  writeoff/pending).

**The fix is ONE column, not a table:** `acc_bill_balance.abl_tcs_amount`
numeric(14,2) default 0, **stamped by the SALE BILL** when it raises the `abl_`
row (same figure as `gdr_tcs_value`; `sale_bill_posting_flow` needs that line).
Plus the view `accounts.v_bill_tcs` — collected / pending pro-rata of
alloc / pending. Derived so it cannot drift. An explicit `abl_tcs_collected`
only if someone demands TCS collected in full first instead of pro-rata.
`/receipts/open-items` gains `tcsAmount` + `tcsPending` per bill; the receipt
screen shows two read-only columns, only on the SALES basis.

**A table IS justified for the filing projection** — `acc_tcs_register` /
`acc_tds_register` (27EQ / 26Q): section, rate, PAN, challan no/date/BSR,
quarter, certificate no. Written at post, never edited, never re-summed.
**Legal point:** under 206C(1H) liability arises ON RECEIPT even when TCS is
shown on the invoice, so on the SALES basis the register row is raised **at the
receipt**, for the TCS slice of that allocation.

**The 50L threshold is computed, not stored** — it runs on cumulative
*receipts* from the buyer in the year, not cumulative TCS.

**TDS deducted by us is IN too** (user, same day: *"tds rates table also add,
we deduct tds in payment"*). `accounts.tds_rates` is a **TABLE, not the JSON
setting** the payment plan first proposed (§2.9 there): `tdr_section` ×
`tdr_deductee_type` × effective dates → `tdr_rate`, plus `tdr_no_pan_rate` 20
(206AA), `tdr_threshold_single` / `_annual`, optional `tdr_company_id`
(NULL = all), and a **GiST EXCLUDE** so two active rows cannot cover the same
section × deductee × day (the promotion-scheme discipline). Lookup at the
payment date: `led_tds_nature_of_payment` → section, `led_tds_deductee_type` →
deductee (fall back to `ANY`), empty `led_pan_no` → the no-PAN rate; a
company row beats the NULL default. **Thresholds are checked, not stored** —
the annual one against the year's payments to that party. Seed rows for
194C/194J/194H/194I/194Q are in §2.9 and must be checked against the current
Finance Act. A rate-master screen is owed later, not in this phase.

**S2 is fully closed** in [[accounting-scope-register]] — S1 (cost centres) is
now the only schema-shaped decision still open. Notes (14) and (15).
See [[receipt-design]], [[payment-design]].
