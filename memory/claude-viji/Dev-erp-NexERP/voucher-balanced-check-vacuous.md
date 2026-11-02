---
name: voucher-balanced-check-vacuous
description: "FIXED — ck_avh_balanced now works: tr_av_refresh_totals re-derives avh_total_debit/credit from the live legs, so Dr = Cr IS enforced by the DB; only a POSTED voucher with zero legs slips through"
metadata:
  type: project
---

**Current state (verified 2026-09-25 on .106):** Dr = Cr is enforced by the DB.
`accounts.acc_vouchers` has `tr_av_refresh_totals` (AFTER INSERT/UPDATE/DELETE,
per row) → `fn_avh_recompute_totals(voucher, year)`, which sets
avh_total_debit/credit to the sums of the NON-deleted legs (and recomputes the
voucher a leg moved away from). `ck_avh_balanced` (POSTED ⇒ debit = credit) then
means what it says. Writers build the voucher as DRAFT and flip it to POSTED
once the legs are in, because the CHECK is immediate, not deferred.

**Why:** the 2026-09-12 finding (totals were stamped by the writer, so the CHECK
compared figures against themselves) was sent to Prathap as notes (4), and
he implemented the trigger fix suggested there.

**How to apply:** do not propose a separate "voucher balances" check. The
user corrected me on 2026-09-25 when I proposed one as C3 in notes 47. The only
remaining corner is a POSTED header with no legs (0 = 0), and there were 0 such
vouchers on the box.

Related: [[bill-ledger-reconcile]], [[sale-bill-posting-dr-cr]]
