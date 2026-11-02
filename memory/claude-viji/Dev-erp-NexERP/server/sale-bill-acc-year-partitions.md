---
name: sale-bill-acc-year-partitions
description: nine tables are LIST-partitioned by acc_year; a new fiscal year needs ensure_acc_year_partitions() before any save works, and P2002 errors report partition-local index names
metadata: 
  node_type: memory
  type: project
  originSessionId: 6273fff3-5a3d-4ccb-80f3-bd057d22abc7
  modified: 2026-08-11T07:25:59.934Z
---

`sales.sale_bill`, `sales.sale_bill_item` and `accounts.acc_tender_detail` are declaratively
partitioned by `LIST (acc_year)`. Their migrations create only the parent shells, so Postgres
rejects every insert with `no partition of relation "sale_bill" found for row` until a child
partition exists for that year. Migration `20260804112944_add_acc_year_partitions` (added
2026-08-04) defines `public.ensure_acc_year_partitions('YYYY-YYYY')` — idempotent — and runs it over
every year in `public.fiscal_years`.

Each new partitioned table extends that one helper rather than creating partitions in its own
migration, so the list has grown: `20260808120000_add_txn_status_log` added `public.txn_status_log`,
`20260808132323_add_sale_order_tables` added `sales.sale_order`, `sale_order_item` and
`sale_order_advance_alloc`, and `20260810130000_partition_sale_quotation_by_acc_year` added
`sales.sale_quotation` and `sale_quotation_item` — **nine tables** as of 2026-08-10. When adding
another, `CREATE OR REPLACE` the whole function (re-listing every existing table, or you silently
drop them) and re-run the `fiscal_years` bootstrap DO block at the end of the migration.

**Converting an EXISTING table** (what 20260810130000 did — Postgres cannot ALTER an ordinary table
into a partitioned one): rename the table AND every index out of the way (`ALTER TABLE … RENAME
CONSTRAINT` for PK/unique, `ALTER INDEX … RENAME` for the rest — index names are schema-scoped,
constraint names are only table-scoped), create the partitioned parent, extend the helper, create
partitions for every `fiscal_years` year UNION every year the parked rows carry, copy with explicit
column lists, assert the row counts match, drop the parked tables. Generate the column list from
`pg_dump --schema-only` rather than hand-typing it. Dry-run the whole file with
`psql -c 'BEGIN;' -f migration.sql -c 'ROLLBACK;'` before `migrate deploy` — DDL is transactional.

**Why:** partition creation is not wired into any application code (there is no fiscal-year CRUD
module), so it is a manual bootstrap step.

**How to apply:** whenever a new fiscal year is opened, run
`SELECT public.ensure_acc_year_partitions('2027-2028');` or bills/tenders in that year fail with a
500. Related: [[prisma-scalar-list-null]], [[prisma-migration-gotchas]].

**Unique-violation errors name the PARTITION, not the parent index.** Verified 2026-08-08: a
duplicate against `ux_so_order_no` on `sales.sale_order` reports
`sale_order_2026_2027_so_company_id_so_branch_id_so_acc_yea_idx4`. The name is auto-generated,
positional (`_idxN`) and different per year, so matching `P2002` `meta.target` against a literal
index name cannot work for any partitioned table. `quotation.service.ts` solves it with
`resolveDuplicateIndex` (added 2026-08-10 when sale_quotation was partitioned): one `pg_inherits`
join maps the reported child index back to the parent index name, then the existing literal
matching still works. Copy that rather than matching on the truncated name.

**The helper's list drifts — re-read it from the DB before editing it.** Verified 2026-08-11:
`20260811060000` re-declared the function without `public.txn_charge_detail` (added 20260810140000)
or `public.txn_hold` (added 20260810160000), so the live definition silently stopped creating those
two partitions. `pg_get_functiondef('public.ensure_acc_year_partitions(character)'::regprocedure)`
is the source of truth; the newest migration that touches it is not.

**`accounts.acc_bill_balance` is deliberately NOT partitioned** (since
`20260811080000_replace_acc_bill_balance_and_adjustment`, which replaced both bill tables). A bill
stays open across financial years, so it is one table keyed on `abl_id` alone with `abl_acc_year` as
a reporting filter. `accounts.acc_bill_adjustment` IS partitioned — by the year of the SETTLEMENT
(`abj_acc_year`), so an April receipt against a March bill lands in April's partition, and the
cross-year reference back to the bill is a plain single-column FK.

**Not covered by that helper:** `accounts.acc_voucher_header` and `accounts.acc_vouchers`, added
2026-08-06 by `20260806090000_add_voucher_header_bills_pdc`, are also `LIST (acc_year)` partitioned
but `ensure_acc_year_partitions` does **not** create their partitions — only `2026-2027` exists.
They deliberately have no DEFAULT partition, so a voucher in any other year fails outright rather
than landing in a catch-all. Opening a new FY means adding those two partitions by hand, or
extending the helper to cover them. Both tables use a composite PK `(id, acc_year)` — every FK
into them must carry both columns.
