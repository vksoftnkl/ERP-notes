---
name: ui-tables-5-opening-stock-is-stale
description: fixed.ui_tables id 5 "opening stock" holds 45 columns for the RETIRED osl_ screen, not the new stock_voucher one — do not point the new screen at it
metadata:
  type: project
---

`fixed.ui_tables` id **5 "opening stock"** exists and looks ready to use. It is not. Its 45 seeded columns (`prisma/seed/Ui_Table_Columns.sql:153-198`) describe the **retired** screen that wrote `stock.opening_stock_header` / `_detail`, dropped in migration `20260907070000`:

- ten hidden id columns named `osl item id`, `osl unit id`, `osl base uom id`, `osl godown id`, `osl tracking type`, `osl tax id`, `osl tax perc`, `osl cess type`, `osl cess perc`, `osl cess per unit` — `osl_` is the dropped `opening_stock_detail` line prefix;
- twelve price-level columns (`Price A/B/C/D` with wot + markup, `M.R.P`, `M.S.P`, `Profit type`, `Round off`) which the replacement `stock.stock_voucher_item` (`svi_`) **deliberately does not have** — prices stay in `inventory.item_price_master` and `stock.stock_mrp_price`.

Pointing the new `/api/v1/stock/opening` screen at table 5 binds a grid to fields that API never returns.

`fixed.menu_master` 44 "Opening Stock" *does* already exist and is fine (`prisma/seed/Menu_Master.sql:156`). Note also there is no `ui_table_master` table — that is only the NestJS module name; the table is `fixed.ui_tables`.

**Why:** the obvious check ("does a ui_tables row for opening stock exist?") answers yes and hides the real problem, which is that it answers for a screen that no longer exists.

**How to apply:** agree a new column set with whoever owns the Qt client, then seed it on a NEW pinned id (template `prisma/seed/Quotation_Charges_Grid_Web.sql`) rather than editing table 5 — and note `Ui_Table_Columns.sql` guards per table, so a table that already has any column is skipped entirely and edits to that file are a no-op on existing databases. A seeded row is still only half the change; see [[ui-table-columns-drive-grids]] for the client-side meaning the column also needs. Related: [[stock-engine-ddl-lives-outside-repo]].
