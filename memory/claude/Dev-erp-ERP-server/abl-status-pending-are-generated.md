---
name: abl-status-pending-are-generated
description: "acc_bill_balance abl_status / abl_pending_amount are GENERATED ALWAYS from the amounts; a soft-deleted (cancelled) bill keeps reading OPEN with its pending — house-wide, by design"
metadata:
  node_type: memory
  type: project
  originSessionId: 3b4d1528-fc7f-40d0-a068-0ea6ddc6abc3
  modified: 2026-09-26T08:40:48.225Z
---

`accounts.acc_bill_balance.abl_status` and `abl_pending_amount` are `GENERATED ALWAYS`
from bill/alloc/disc/writeoff amounts. No code can set them to CLOSED / 0. Every
cancel path (sale bill, sale return, order advance, opening, /vouchers/cancel)
only sets `abl_is_deleted = true`, so cancelled bills still read OPEN with a
pending (checked 2026-09-26: ~390 such rows).

**Why:** the convention is "readers filter `abl_is_deleted = false`"; all four
partial indexes (ix_abl_open, ix_abl_overdue, ix_abl_party_credit,
ix_abl_salesman) carry that predicate. Faking allocations to force CLOSED
would fight `BillBalanceRecomputeService`, which re-derives abl_alloc_amount.

**How to apply:** when a note reports "a cancelled bill still reads OPEN", the
fix is the reader (add the deleted filter), not the cancel path. Making
status/pending deletion-aware means changing the generated expressions on a
partitioned table (drop+re-add column, rebuild the 4 indexes) — ask first.
See [[voucher-register-module]].
