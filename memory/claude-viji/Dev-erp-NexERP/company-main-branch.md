---
name: company-main-branch
description: "notes 78 + 45_main_branch_seed.sql (2026-10-02): company create must seed a default 'Main Branch' (+ Main Godown); backfill + one-default unique index; awaiting Prathap"
metadata:
  type: project
---

Company create seeds the fiscal year (notes 72 A1) but NO branch, so a new company still can't be used.
Write-up: `share/admin/notes-78-company-main-branch.md`. SQL: `share/45_main_branch_seed.sql`.

The SQL is an idempotent backfill (insert a Main Branch / HEAD OFFICE copied from the company, mark a lone
branch default, list several-without-default) plus `uq_branch_master_default`. It dry-ran on erp_dry
with all cases + a re-run. Live preview 2026-10-02: 1 insert (LEAPSWITCH), 3 marks (CHANDRA, HAP, SCM).

**Asks for Prathap:**
1. Seed the branch in `createCompany()`.
2. Seed a Main Godown (WAREHOUSE) and set `brDefaultGodownId`.
3. Refuse unticking the current default.
4. Decide whether the last branch of a live company may be deleted.

**Status:** awaiting Prathap. When it lands, re-verify live. Then, once item 2 is in, the client's Company "saved"
message names the seeded branch/godown. Related: [[company-branch-masters-audit]], [[sql-vs-nestjs-rule]].

**2026-10-02 later:** the 45 backfill IS applied on the box (every company has a default branch). Grid 52
was trimmed at the user's ask ("only important columns"). It keeps #/Code/Company Name/GSTIN/State/Main Branch/Branches/Active
and drops Short, Legal Name and GST Type (retired via replace_columns). The old definition is in `share/admin/grid52_before_2026-10-02.json`.
Live run and search were verified. The client reads only comp_id/comp_name from it.
