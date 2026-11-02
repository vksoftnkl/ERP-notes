# Notes 144 — Price change log: APPROVED, with qty slabs (2026-10-23)

The design is `plan-price-change-log.md` (with `plan-price-change-log_mockup.png`).
That file is unchanged. This note records the user's decisions and the
additions they require. Where this note and the plan differ, this note wins.

## User's decisions (2026-10-23)

| Question | Answer |
|---|---|
| Use `audit.audit_log`, or a new table? | **New separate table**, `inventory.item_price_change_log`. `audit.audit_log` keeps getting its rows as today. |
| Include the qty-wise slab prices (menu 244, `inventory.item_qty_price`)? | **Yes**, in the same table, `ipcl_price_table = 'IQP'` (plan Q1). |
| Who may see the report? | Not answered yet. The plan's default stands: a normal menu right. |

## What "with qty slabs" adds to plan §1 / §2 (overrides)

1. **Action RESTORE.** The qty price service restores soft-deleted slabs (`items-qty-price-master.service.ts` `restore()` → `setDeletedItemQtyPrice`). Add it:
   `CHECK (ipcl_action IN ('CREATE','UPDATE','DELETE','RESTORE','REKEY'))`, shaped like UPDATE (before and after both present).
2. **Where an IQP row applies.** Fill these from the slab:

   | log column | slab column |
   |---|---|
   | `ipcl_company_id` / `ipcl_branch_id` / `ipcl_item_id` | `iqp_company_id` / `iqp_branch_id` / `iqp_item_id` |
   | `ipcl_unit_id` | `iqp_item_unit_id` |
   | `ipcl_party_id` | `iqp_party_id` |
   | `ipcl_price_level` | `iqp_price_level` |
   | `ipcl_effective_from` / `_to` | `iqp_effective_from` / `_to` |

   For IQP rows, `ipcl_godown_id`, `ipcl_bucket_mrp` and `ipcl_bucket_sp` stay NULL. Also snapshot the party's name: add `ipcl_party_name varchar(250)`.
3. **Generated columns for slabs**, next to the IPM ones, so the report can filter and sort slab changes:
   ```sql
   ipcl_old_slab_price numeric GENERATED ALWAYS AS ((ipcl_before->>'iqp_price')::numeric)    STORED,
   ipcl_new_slab_price numeric GENERATED ALWAYS AS ((ipcl_after ->>'iqp_price')::numeric)    STORED,
   ipcl_slab_from_qty  numeric GENERATED ALWAYS AS (COALESCE(ipcl_after, ipcl_before)->>'iqp_from_qty')::numeric STORED,
   ipcl_slab_to_qty    numeric GENERATED ALWAYS AS (COALESCE(ipcl_after, ipcl_before)->>'iqp_to_qty')::numeric   STORED
   ```
4. **The writer.** The qty price service's writes move into `PriceMasterWriter` with the IPM ones:
   - `createItemQtyPrice` (~308)
   - `updateItemQtyPrice` (~362)
   - `setDeletedItemQtyPrice` (~248), which handles both delete and restore

   The guard test covers `itemQtyPrice.(create|update|updateMany|…)` too.
5. **No-op check for slabs.** Compare the business columns only:
   - compare: from/to qty, price mode, disc %, flat off, price, tax-incl, effective dates, active, deleted, party, price level, unit.
   - ignore: `iqp_modified_on`, `iqp_sync_date`.
6. **Field labels** for the report map (plan §3):

   | column | label |
   |---|---|
   | `iqp_from_qty` / `iqp_to_qty` | From qty / To qty |
   | `iqp_price_mode` | Price mode |
   | `iqp_disc_pct` | Disc % |
   | `iqp_flat_off` | Flat off |
   | `iqp_price` | Slab price |
   | `iqp_is_tax_incl` | Tax incl. |
   | `iqp_price_level` | Price level |
   | `iqp_party_id` | Party |

7. **On the report**, a slab change reads as a chip like "10+ qty: 128 → 125 ▼2.3%" (as in the mockup). The scope reads "Price level B · party Ravi Stores" when those are set. Its source is `QTY_PRICE` (menu 244).

## Server work, in order (plan §5, unchanged)

1. Create the table as in plan §1 plus points 1–3 above, with the four indexes, and append-only grants.
2. Build `PriceChangeLog.record()` and `PriceMasterWriter`. Move all writes onto it:
   - item card / items-price-master
   - Change Selling Price gateway (6 writes)
   - `price-bucket.service.ts rekeyItem` (not audited at all today)
   - qty price create / update / delete / restore
3. Add the guard test (plan §2.5) and the gap count (plan §2.6).
4. Add the four read routes, `/item-price-log/list|get|days|item-history` (plan §3). The filters take `priceTable` (IPM / IQP / both) as well.
5. Tell us when it's on .106. We'll build the Qt report (plan §4) against the live routes, with a live test that changes a ZT- item from all four sources and checks one log row per change.
