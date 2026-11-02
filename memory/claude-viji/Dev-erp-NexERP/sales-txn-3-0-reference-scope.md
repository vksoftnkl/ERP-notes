---
name: sales-txn-3-0-reference-scope
description: For the sales-transactions work (2026-09-20) 3.0 is a reference for bill / return / loyalty / scheme / e-invoice / e-way ONLY — the delivery challan takes NOTHING from 3.0; plus where the 3.0 DB is
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 0b00a13b-6c14-4cbf-9fa7-d864e9a434dd
  modified: 2026-09-20T05:44:14.485Z
---

User, 2026-09-20, mid-analysis: **"dont take any ref for delivery challan from 3.0"**.

**Why:** 3.0's DC (`app_utils.fx_sale_chln_entry`, `sales_del_challan*`) is not a
design the user wants carried forward. The DC and DC Return are to be designed
from the Tally / GoFrugal delivery-note model and our own tables (stock ledger,
doc register, txn_status_log) only.

**How to apply:** when writing `share/sales/` plans, cite 3.0 for the bill,
return, loyalty, promotion, tender, e-invoice and e-way behaviours (see §7 of
`share/sales/analysis_2026-09-20.md`) and never for the challan — no column,
flow or rule from `fx_sale_chln_entry` may appear there.

**The 3.0 database is on 192.168.0.106 as `3.0`** (86 MB; `psv-shop` 4.8 GB is a
client's data — not opened). `claude` login works after the pg_hba line the user
added; `~/.pgpass` has the row. Function bodies live in `public.fsales`,
`fsales_return`, `feinvoice_system`, `fsales_dispatch`, `floyalty_schemes` and
`app_utils.fx_*`. Dumps of the ones read are in the session scratchpad only.
See [[nestjs-source-readonly]] for the sibling rule on Prathap's code.
