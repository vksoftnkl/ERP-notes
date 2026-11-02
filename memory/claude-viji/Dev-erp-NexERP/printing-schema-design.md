---
name: printing-schema-design
description: "17_printing.sql — data-driven printing in PUBLIC (no printing schema): ppo_/ptl_/ptv_/ptd_/pta_/prf_/plg_ family, stored-SQL datasets, four-rung counter resolution; supersedes the live reports.* tables"
metadata:
  node_type: memory
  type: project
---

Printing subsystem: `17_printing.sql` + `17q_printing_grid_queries.sql` in the share
(see [[db-schema-share-location]]). Verified on a throwaway PG18.6 cluster: 7 tables,
3 partitions, 39 FKs, 68 CHECKs, 115 indexes, **0 FKs without a covering index**,
idempotent over three consecutive runs, all 10 companion queries PREPARE clean.
Reference PDF + its generator: `schema/17_printing_reference.pdf`, `schema/17_docgen/`
(catalog-driven; `printdoc.py` reports any column with no prose and that count must stay 0).

**Tables live in PUBLIC. There is no `printing` schema** — changed 2026-08-27 on the
user's instruction, having originally been its own schema. Before running anywhere,
check these seven names are free in public: print_purpose, print_template,
print_template_version, print_template_dataset, print_template_assignment,
printer_profile, print_log. The live db's `reports` schema has print_template and
printer_profile — different schema, same names, so **never write either unqualified
and never rely on search_path**. Prisma multiSchema needs hand-named models for the pair.

**A previous generation is ALREADY LIVE and the user chose to supersede it, not adopt it** —
`reports.print_template` (pt_, 16 rows), `_revision` (ptr_, 8), `printer_profile` (pp_, 6)
behind 15 working `/reports/*` routes. Asked twice, reaffirmed "design fresh, backend adapts".
§12 is the migration, commented and NOT executed. Nothing live was altered.

**Design decisions that took real argument — do not re-derive:**
- **The body lives on the VERSION.** That makes `plg_version_id` a real FK to the exact
  bytes rendered (RESTRICT, load-bearing) and publish/rollback a pointer move on
  `ptl_published_rev_id`. Circular FK added by trailing ALTER.
- **Datasets hang off the VERSION too**, else editing a query changes what past versions rendered.
- **No `is_default` column.** Default-ness IS a `pta_` row's existence.
- **`ptd_dataset_no` (binding, immutable) vs `ptd_sort_order` (display).**
- **`ptv_engine`** so moving off QtRPT is not a flag day. **No platform column, no language fork.**
- **`ptd_link_fields`** (`parent=child`, comma separated, `ck_ptd_link_fields_shape`):
  the child dataset **runs ONCE for the whole render**, returns the parent key as a column,
  and the renderer groups on it. Not per parent row — that is N+1 and would make
  `ptd_row_limit`/`ptd_timeout_ms` measure the wrong quantity. So `ptd_params.source`
  stays CONTEXT|USER with **no PARENT value**: one mechanism for nesting, not two.
- **`pta_company_id` is NULLABLE = every company** (changed 2026-08-27). Consequences that
  had to move with it, all verified by test: `pta_specificity` is now **4 rungs (0 all
  companies, 1 company, 2 branch, 3 counter)**; `ux_pta_scope` and `ix_pta_resolve` COALESCE
  the company to the nil uuid; new `ck_pta_branch_needs_company`; the 17q resolver decodes
  3/2/1/0 and filters `COALESCE(pta_company_id,nil) IN (nil, :company_id::uuid)`.
- **The cross-company lock.** A global assignment may name only a SHIPPED design, else one
  company's private design renders on every other company's till. Enforced, not trusted:
  generated `ptl_company_key` (owner, NULL folded to nil uuid) + NOT NULL
  `pta_template_company_key` + composite `fk_pta_template` → `ux_ptl_id_company_key`, plus
  `ck_pta_template_scope`. **`MATCH SIMPLE` is why the key needs the sentinel** — a NULL in
  any column makes a composite FK stop checking entirely, exactly for shipped designs.
- **`pta_printer_name`** (added 2026-08-27) is a bare queue name for a scope with no
  registered profile. `ck_pta_printer_one_of` refuses it alongside `pta_printer_id` — a
  denormalised name on a master goes stale, unlike `plg_printer_name` on the immutable log.

**A CHECK constraint is satisfied by NULL.** `ck_pta_template_scope` was first written as
`key = nil OR key = pta_company_id`; on a global row that is `false OR NULL` → NULL → passes,
so the one case the lock existed for was the one it let through. Fixed by COALESCE-ing both
sides. It only surfaced because each test case ran in **its own transaction** — run in
sequence, `ux_pta_scope` refused the row first and the hole hid behind a passing test.

**Stored-SQL guards** normalise into generated `ptd_sql_norm` first. The subtle one:
`ck_ptd_sql_no_quoted_param` is a **regexp_count comparison**, not a pattern — the obvious
regex falsely rejects a literal appearing before a real parameter. Header states plainly that
these are an authoring LINT; the wall is extended-protocol binding + READ ONLY txn +
a privilege-stripped role.

**Client side is greenfield:** no `QPrinter`/`QPdfWriter` anywhere; `printQuotation`/
`printSaleOrder`/`printSaleBill` (`txn_doc_types_sales.cpp:96,226,267`) are stubs.
`sb_print_count` becomes a cache of `COUNT(DISTINCT plg_render_id)` filtered to
PRINT/ESCPOS + PRINTED — **once per render, never per copy**.
Screens: see [[printing-ui-screens]]. Legacy 3.0 templates are plain text at
`/home/vk/Projects/Future/Others/DBVersion/Sql/3.18.sql:2397-4210`.

**Why:** chain-store ERP; printing must never need a release, and tables cannot be altered
after go-live. **How to apply:** run order 12→…→16→17; deliver as a recorded Prisma migration.
See [[promotion-schema-design]], [[sales-backend-schema-design]], [[api-endpoint-docs]].
