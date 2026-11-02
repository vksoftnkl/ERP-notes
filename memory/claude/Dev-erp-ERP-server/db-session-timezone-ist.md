---
name: db-session-timezone-ist
description: The dev DB session TimeZone is Asia/Kolkata — a DATE column compared with a JS Date bound as ::timestamptz is off by 5h30 and silently matches nothing
metadata:
  node_type: memory
  type: project
  originSessionId: 69f8d120-dca7-4e3e-92c1-a0d4f1fcd266
  modified: 2026-09-29T02:41:09.027Z
---

`SHOW timezone` on the dev database answers **Asia/Kolkata**. A `date` column compared with a
JS `Date` bound into raw SQL as `${d}::timestamptz` casts the column to timestamptz in the
SESSION zone: `'2026-09-29'::date` becomes `2026-09-28 18:30Z`, which is LESS than the bound
midnight-UTC `2026-09-29T00:00Z`. So `h.avh_voucher_date >= ${from}::timestamptz AND <= ${to}::timestamptz`
over a one-day window matches nothing, and a row-tuple walk keyed on the date sorts every
same-day row below the row being walked from.

**Why:** found 2026-09-29 building `/payments/adjacent`, a verbatim copy of
`/receipts/adjacent` (`receipt/open-items.service.ts` `adjacent()`), whose e2e walk visited 4 of
the register's rows. The payment's copy was fixed first; the receipt's was fixed the same way
by notes (61) the same day.

**How to apply:** for a `@db.Date` column in `$queryRaw`, bind a `'YYYY-MM-DD'` string and cast
`::date` on both sides (`toDateString(...)`). Prisma's own `findMany` with a `Date` on a
`@db.Date` field is fine — it formats the date in UTC. Grid SQL already does
`h.avh_voucher_date::date >= NULLIF(ifrom_date, '')::date`. See [[payment-module]].
