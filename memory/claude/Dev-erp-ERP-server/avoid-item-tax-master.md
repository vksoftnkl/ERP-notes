---
name: avoid-item-tax-master
description: User rule — never read inventory.item_tax_master (module/routes/grid 5 retired 2026-09-30, table kept); an item's / group's / tax-history's tax id is a tax_rate_master id (FK since 20260912110000); cess basis PER_UNIT/BOTH, not UNIT
metadata:
  type: feedback
---

Never read `inventory.item_tax_master` (Prisma `itemTaxMaster`) for a tax. `item_default_tax_id`,
`itg_default_tax_id` and `ith_tax_id` all hold `tax_rate_master` ids (FK
`item_master_item_default_tax_id_fkey`, migration 20260912110000); on the local DB all 10,041 taxed
items resolve there and none in the old table, so any old-table read silently gave null name / 0%.

**Why:** the user said "completely avoid item tax master table" (2026-09-30, notes 67 B2). Every
reader outside the old CRUD modules was repointed that day (items-master get + bulk-load,
item-price-details x2, selling-price-bulk, master-lookup `itemTaxes`, opening-stock lookup,
audit-log name resolver).

**How to apply:** map old → new: `tax_gst_rate_total`→`tax_rate_perc`, `tax_cess_type`→`tax_cess_basis`
(NONE | PERCENT | PER_UNIT | BOTH — the old table said UNIT), `tax_cess_unit`→`tax_cess_per_unit`,
`tax_taxability_type`→`tax_taxability`; no purchase-specific rates and no ledger ids exist
(ledgers: acc_ledger_map / tax_rate_ledger). RETIRED 2026-09-30 on the user's "yes retire": the
`Inventory/items-tax-master` module and its `/item-taxes` routes, grid 5 (migration
20260930130000 + both grid seeds), the "Item Tax Master" audit-SQL constant. The TABLE and its Prisma
model remain (10 rows, no reader, no FK into it) until the user asks for a drop.
`items-tax-history-master` is NOT part of it — `ith_tax_id` points at tax_rate_master. Root-level
`utils/swaggerDocs.ts` is a dead pre-Inventory copy (main.ts uses src/utils/). The web "ERP client"
opening-stock page still calls `/item-taxes/get` (it 404'd anyway for tax_rate_master ids).
`src/modules/item-price-details` (outside Inventory/) is an unwired copy.
