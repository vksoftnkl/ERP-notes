---
name: item-master-notes-137
description: Notes 137 (item master audit) built 2026-10-21 — names/codes/SKUs unique per company any case (user's rule), child DELETEs delete-only, actor always the token's; C1 company scoping deliberately NOT done
metadata:
  type: project
---

Built 2026-10-21, migrations 20261021100000 (item indexes, reorder CHECK, profit default, grid 71 text)
and 20261021110000 (ui_table_columns `ex_ui_tbl_clm_no`, a DEFERRABLE exclusion constraint, not a
partial unique index — a column swap inside one save would trip a plain index). Proof:
`test/items-master-notes-137.e2e-spec.ts` (18 cases, rolled back).

User's decisions (2026-10-21): item name, item_code and item_sku are unique among live items of ONE
company, case-insensitive, and a shared item (no company) clashes with every company's. The
shared-vs-company half lives only in `ItemsMasterService.assertUniqueKeys` (advisory lock per value),
not in an index. **C1 (company scoping of item routes) left undone on purpose**: the token company is
the HOME company, scoping by it would hide items from multi-company users — needs its own design.

Not done / open: D8 is fixed only for item DTOs (`OptionalNotNullBoolean`); the house pipe's
enableImplicitConversion still turns "false" into true for every other `OptionalBoolean`. B9 is
app-side checks only — no FKs added for brand/section/supplier/cust group/ir_branch/ir_godown (would
need Prisma relations on both sides). The Qt path that appended 22 rows to ui_table 17 on 10-12
(VIJAY, uiTblId 17, no replaceColumns, rows named by display labels like "WS Margin", no width) is a
client bug to find; the server now 409s it (UI_TABLE_COLUMN_NO_DUPLICATE).

**Why:** see the notes; A1 let item B's save take item A's barcode by id.
**How to apply:** child routes `/item-prices|ean-codes|unit-conversions|reorders/delete` call
`delete()` (409 when already deleted); `toggleDelete` is now only the composite's tool — never point
a route back at it. Child updates load by id AND item and never rewrite the item id. Related:
[[item-master-delete-restore]], [[dto-hasownproperty-always-true]], [[dev-box-is-192-168-0-106]].
