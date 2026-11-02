---
name: sale-bill-alloc-is-seeded
description: FIXED 2026-09-25 (notes 49) — a sale bill's counter payment is now ALLOCATION adjustment rows (abj_tender_id), so the recompute keeps it; retender recomputes instead of overwriting
metadata:
  type: project
---

Until notes (49), a POSTED sale bill's `abl_alloc_amount` was only SEEDED with its counter tenders, and `BillBalanceRecomputeService` (which derives alloc from adjustment rows only) dropped it on the first receipt — D1 (bil00685 read 5 settled instead of 3,773).

Now `/post` writes one `ALLOCATION` row per settling tender via `bill/bill-counter-allocation.helper.ts` (`abj_tender_id` = td row, `abj_cheque_id` for cheques). All 126 posted bills on the box were backfilled 2026-09-25 (`scripts/backfill-bill-counter-allocations.ts`, capped at header paid − advance − note; reconcile went to 0 rows). The seed is still written too and equals Σ rows.

**Why:** the recompute is the one writer receipts/cheques/sweeps use; anything not in rows is lost.

**How to apply:** any new path that changes how a bill was paid must move these rows (drop/add) and then call `recomputeBills` — never write `abl_alloc_amount` directly. Guards that ask "has someone else allocated against this bill?" must exclude rows whose `abj_tender_id` is one of the bill's own tender rows (assertCancellable and BillReadService.lockCounts do). A bill cheque bounce now goes through the standard `abj_cheque_id` reversal; `sale-bill-cheque.helper` only moves the sale_bill header caches. Related: [[books-reconcile-trial-check]].
