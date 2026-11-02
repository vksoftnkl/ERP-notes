---
name: fk-cd-tax-and-ix-cd-tax-missing
description: txn_charge_detail.cd_tax_code has no FK and no index on localhost — the tail of migration 20260912050000 never ran
metadata: 
  node_type: memory
  type: project
  originSessionId: 99e184ae-6cfd-4d68-9fd9-e701b9c28596
  modified: 2026-09-17T14:16:53.168Z
---

On localhost/ERP, `public.txn_charge_detail.cd_tax_code` exists but **both
objects that guard it are absent**:

- `fk_cd_tax` → `inventory.tax_rate_master (tax_id)` — **MISSING**
- `ix_cd_tax` (partial, `WHERE cd_tax_code IS NOT NULL`) — **MISSING**

Both are declared at the tail of
`20260912050000_add_line_tax_id` (lines ~95-105), which is one of the five
migrations found with **checksum drift** — edited after it was applied, so the
database never saw the added text. Same mechanism as
[[voucher-totals-trigger-missing]]; the three sibling indexes from earlier in
the same file (`ix_sbi_tax`, `ix_soi_tax`, `ix_sqi_tax`) all landed.

**Why it matters:** without the FK a charge can name a `tax_id` that does not
exist, and nothing refuses it. This is the same area as the recent commits
*"fix(tax): declare the two charge->tax_rate_master relations"* and *"fix(tax):
point the TAXES dropdown at tax_rate_master"*, so whoever owns the tax work
probably wants it.

**How to apply:** measured 2026-09-17 — `txn_charge_detail` holds 51 rows and
**zero** have `cd_tax_code` set, so both objects can be added with no data
repair and no risk of a violation. It needs a NEW migration; an applied one
cannot be re-run. Not fixed at the time of writing because it sits outside the
receipt work that found it — confirm with the user before adding it.

Related: [[prisma-migration-gotchas]]
