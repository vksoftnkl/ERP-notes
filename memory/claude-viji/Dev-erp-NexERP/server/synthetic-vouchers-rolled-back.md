---
name: synthetic-vouchers-rolled-back
description: "Recipe for inserting fake acc_voucher_header/acc_vouchers rows in a test transaction that always rolls back, then running a service's private pipeline on that tx"
metadata:
  node_type: memory
  type: reference
  originSessionId: a9c46fef-191b-41d7-9a42-b241bd58e91a
  modified: 2026-10-12T05:37:18.446Z
---

Working example: the "synthetic day book" block of
test/day-summary-http.e2e-spec.ts ([[account-day-summary-module]]).

- Header CHECKs to satisfy: non-DRAFT needs avh_voucher_no / slno / refno
  (ck_avh_no), avh_status_on + avh_status_by (ck_avh_status_on — not shown in a
  truncated `\d`), POSTED needs avh_posted_on, CANCELLED needs
  avh_cancel_reason.
- fk_avh_reversal exists: insert the POSTED mirror first, then the CANCELLED
  original naming it.
- Insert ALL legs of a voucher in ONE statement: tr_av_refresh_totals is an
  AFTER ROW trigger queued to statement end, so ck_avh_balanced sees both sides
  (see [[ck-avh-balanced-fires-per-statement]]).
- Numbers 990000000+ stay clear of real series; date late in the fiscal year
  (e.g. 2027-03-29) stays clear of real rows.
- `prisma.$transaction(async tx => { …; throw ROLLBACK })` and swallow the
  sentinel. HTTP routes cannot see uncommitted rows, so call the service's
  `build*(tx, scope, books)` methods through `app.get<Svc, Pipeline>(Svc)`.
