---
name: txn-adjacent-walk-order
description: Ctrl+PgUp/PgDn (/txn/adjacent) jumps wrongly because svh_slno is per DEVICE; notes-152 sent 2026-10-24
metadata:
  type: project
---

/txn/adjacent walks `(date, slno, created_on, id)`, but stock voucher slno is counted per device while refno is per branch. So docs from two PCs interleave: OPN0023 (user PC, slno 5) → next = OPN0015 (dbg, slno 15). Reported in share inventory/notes-152 (ask: walk by date, created_on, id). Server-side only; no client workaround.

Also: live tests create + cancel ZT- documents on Coimbatore, so a cancelled test doc can truly be the "newest" for Ctrl+PgUp from a new document. That part is not a bug.

**How to apply:** when the user reports prev/next oddities, check slno vs refno per device first ([[txn-keys-standard]]).
