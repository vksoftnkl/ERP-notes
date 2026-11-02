# notes (67): item master, re-check of notes 50 plus new findings (30-Sep)

The item master was re-checked against the live server on 2026-09-30:
- a throwaway ZT item (`ZT-N50-PROBE-…`, now deleted), created and updated through `/items/create`;
- the Qt read-only drift check over 25 real items;
- the full write round trip, where an unchanged Qt save changes **0 of 151** stored values.

The Qt side is fixed for everything below that a client can work around. The rest is server.

## A. Notes 50, status on the live server today

| # | item | status |
|---|---|---|
| 1 | omitted keys NULLED on update | **still open** for the header. An update without `item_base_unit_id` / `item_company_id` / `item_packing_item_ids` set them to NULL / NULL / `{}`. Fixed for `ipm_godown_id` (kept when omitted). |
| 2 | actor columns on price / conversion rows | **partly.** `ipm_updated_by` falls back to the request user (fixed). `ipm_created_by` on create, and `iuc_created_by` / `iuc_updated_by`, are still NULL when the client sends none. |
| 3 | non-partial unique indexes | **still open.** `uq_item_name_en_global`, `uq_ean_code` and `uq_ir_item_unit_godown` still count deleted rows. The probe item's name is now burned for good. |
| 4 | error field names | **still open.** A bad `item_default_tax_id` is reported as `item_group_id` "Referenced relation does not exist". |
| 5 | `DELETE /items/delete` is a toggle | **still open.** The second call answered "Item restored successfully". |
| 6 | conversion chain drift | not re-tested. Qt sends every row. |
| 7 | orphan unit refs | not re-tested. |
| 8 | item policy always wins over the group | **still open.** A new item with no preset got an ITEM-scope policy `N`. It also stays live (`stp_is_deleted = false`) after the item is deleted. |

## B. New

### B1. Price rows are matched on unit + godown only: one branch's price overwrites another's (bug, data loss)

`syncPrices` matches an incoming row that has no `ipm_id` by `pairKey(ipm_uc_unit_id, ipm_godown_id)` (item-master-update.service.ts:223/239). The table's real key is `uq_item_price_master_scope`: **company + branch + item + unit** (partial on `NOT ipm_is_deleted`).

So a NEW price row for branch Y, with the same unit and the same (or blank) godown as branch X's row, is matched to X's row, takes X's `ipm_id`, and is saved over it. X's price is gone.

Reorders use the same `(unit, godown)` pair (:323/339).

**Ask:** key the match on company + branch + unit (+ godown if godown pricing is meant), the same as the unique index. Qt now refuses to save that case with a message, but the server should not depend on that.

### B2. `item_default_tax_name` is always null (bug)

The GET (and `/items/bulk-load`) resolve the tax name from `inventory.item_tax_master`. All 10,041 items with a tax point at `inventory.tax_rate_master` (`tax_id`), and none at the old table. So the name is always null and the Tax box opened blank.

**Ask:** join `tax_rate_master` (its `tax_name`), in both the get and bulk-load.

*Qt meanwhile:* it fills the label from `/tax-rates/get`, which it already calls for the rate.

### B3. Legacy profit types in the data (data fix)

`ITEM_PRICE_PROFIT_TYPES` is `By % / By Rs / By User`, but 25 live price rows on 14 items still hold `BY_PERCENT` (23) or `BY_AMOUNT` (2). Any save that echoes them is a 400 "must be one of the following values".

Qt now maps them on load (BY_PERCENT → By %, BY_AMOUNT → By Rs). Please add a one-off data migration so other clients and reports see the same values:

```sql
update inventory.item_price_master set ipm_profit_type='By %'  where ipm_profit_type='BY_PERCENT';
update inventory.item_price_master set ipm_profit_type='By Rs' where ipm_profit_type='BY_AMOUNT';
```

### B4. Restore brings back every child ever removed (bug, part of notes 50 #5)

The restore flips **all** soft-deleted children of the item (`findIdsByItemId(itemId, true)`), including unit, price and barcode rows that earlier syncs had removed on purpose. So old base units and old EANs reappear. Please restore only the rows deleted together with the item.

*Qt meanwhile:* in "Show Only Deleted" the action is now labelled **Restore** and warns about this.

## Priority, from the counter's point of view

B1 (silent price loss), 1 (silent header wipe for any non-Qt client), 5 / B4 (restore), B2 (blank tax), B3 (data), then 3 / 4 / 2 / 8.

## Verified live 2026-09-30 (after Prathap's run)

Tested on a throwaway item `ZT-N67-PROBE-125403`, deleted afterwards.

| item | result |
|---|---|
| 1 | **fixed.** An update without base unit / company / packing keeps all three. |
| 2 | **fixed.** `ipm_created_by`, `iuc_created_by` and `iuc_updated_by` fall back to the request user. |
| 3 | **fixed.** The name, EAN and reorder unique indexes are partial on `NOT is_deleted`. |
| 4 | **fixed.** A bad tax is reported on `item_default_tax_id` ("does not name an existing tax rate"). |
| 5 / B4 | **fixed.** A second delete answers "Item is already deleted". `POST /items/restore` restores; a second restore gives 409 "not deleted". |
| B1 | **fixed.** A new price row for Head Office, with the same unit and godown as the Coimbatore row, was saved as its own row. Coimbatore's price was untouched. |
| B2 | **fixed.** `item_default_tax_name` = "GST 18%" on get, and `tax_name` on bulk-load. |
| B3 | **fixed.** 0 legacy profit types remain. |
| 8 | **still as before.** A new item with no preset gets an ITEM-scope `N` policy, so the group's "Tracked as" never applies. It is now deleted with the item. This is a design question: say if it is intended. |

**Qt follow-ups done:**
- the list's Restore now calls `/items/restore`;
- the client-side unit + godown refusal for new price rows is removed;
- the Tax label fallback is kept only for a GET that has no name.

The unchanged-save round trip still changes 0 of 151 values.
