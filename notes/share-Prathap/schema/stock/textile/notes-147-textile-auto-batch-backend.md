# notes-147 — Textile: auto batch no + size / colour / model (server work)

Date: 2026-10-23. From: Viji / Claude. To: Prathap.

## What changed

The shop rejected the separate tag page (notes-136).
- **Notes-136 is PARKED, so please don't build it.** Keep the design for later.
- The new flow:
  - **Purchase / opening line:** item, size, colour, model, qty, rate, MRP, selling price. The batch no is filled in automatically on save.
  - **Sale bill:** scan the batch no and the line fills itself.

The full specs, in this folder:
- `plan-textile-auto-batch.md` — D1–D8: batch format, counter, preset, scan route, sale bill.
- `plan-textile-auto-batch-2.md` — E1–E4: size / colour masters, batch detail table, reads.

## Server work, in order

**S1**
1. **Preset `TEXTILE_BATCH`:** batch + MRP + selling price (signature BMS), LOT_ACTUAL, FIFO, negative ALLOW.
2. **Company setting** `stock.auto_batch_no` in app_setting_def (default false).
3. **Counter-only voucher type `BATCH`** in `acc_voucher_types`: no prefix, width 5, reset NONE, affects_accounts false.
4. **Batch number allocation** with `allocateVoucherNumber()`:
   - Format: `<br_bill_prefix or br_code><YY>-<NNNNN>`, upper-case, for example `CBE26-00001`.
   - Refuse the save if the branch has neither code.
   - Given at the first save of an opening stock document, for each line whose batch is blank on a batch-tracked item.
   - Please do **not** use `common/utils/batch-number.util.ts`: MAX + 1 races and is unsafe offline.
5. **Masters (E1):** `inventory.item_size_master` (isz_) and `inventory.item_colour_master` (icl_):
   - routes `/item-sizes` and `/item-colours` `{save,get,delete}`;
   - a list grid for each;
   - a dropdown for each;
   - two menu ids under Inventory masters.
6. **`stock.stock_batch_detail` (sbd_, E2)**, keyed by (company, batch_no), written by the opening stock save in the same transaction.

**S2**
1. `GET /api/v1/stock/lots/by-batch?companyId=&branchId=&batchNo=[&includeEmpty=]`: lot, item, unit, MRP, selling price, on-hand per godown, size, colour, model.
2. `GET /api/v1/stock/batch-detail/models?companyId=&itemId=&q=`: model suggestions.
3. `POST /stock/batch-detail/update`: corrects size, colour or model after posting; a reason is required.
4. `/bills/get`, the DC and return reads, and the bill print data source return sizeName, colourName and model per line, joined on batch no.

**Later:** the purchase module (its own plan) reuses S1 items 4 and 6 unchanged.

## Questions

1. Can the counter-only `BATCH` row sit in `acc_voucher_types` without showing in voucher registers or menus? If not, tell me which column hides it, or suggest another home with the same lock.
2. Please send the menu ids and dropdown ids for Size and Colour once they are registered.

## STATUS (Prathap fills in)

- S1:
- S2:

## Client status (2026-10-23, Viji / Claude)

Seen on the box: tables isz_ / icl_ / sbd_, menus 291 / 292, grids 181 / 182, dropdowns 70 / 71,
audit screens 153 / 156. Thank you. Grids and dropdowns run through the runner (200, 0 rows).

**Qt Size (291) and Colour (292) screens are BUILT** and wait only for the routes, which are 404
today. What the screens call:

| Call | Size | Colour |
|---|---|---|
| save (create AND update; the id in the body selects update) | `POST /item-sizes/save` | `POST /item-colours/save` |
| get one | `GET /item-sizes/get?isz_id=` | `GET /item-colours/get?icl_id=` |
| delete | `DELETE /item-sizes/delete?isz_id=` | `DELETE /item-colours/delete?icl_id=` |

- **Body keys = the column names:**
  - Size: `isz_id`, `isz_name`, `isz_code`, `isz_size_set` (ALPHA / WAIST / KIDS / FREE / null), `isz_sort`, `isz_company_id` (null = shared), `isz_is_active`.
  - Colour: the same `icl_` keys, without size_set.
- **GET** returns `{ data: { …the same keys…, isz_company_name } }`, like /item-brands/get.
- **Delete of a used row** → 409 with a plain message. The user then unticks Active.

**Grants:** 291 / 292 have no user_menus rows yet; they need VIEW / CREATE / EDIT / DELETE for the users.

## Client status 2 (2026-10-23 evening) — Size / Colour LIVE-tested

The routes came as `/create` + `/restore` (the Brand family), not `/save`. The Qt screens now follow them.
- API test (tests/itemmasters/api_size_colour.py): **30/30**.
- Screen test (SRC=test_size_colour_screen): **all pass**.
- Covered: create shared + own, update, case-insensitive duplicate 409, grid scope / sort, dropdowns 70 / 71, inactive leaves the dropdown, delete, deleted view, restore.

Two small server points (no hurry):
1. **`/item-sizes/get` and `/item-colours/get` return no `isz_company_name` / `icl_company_name`.** The popup fills the name from the session for now, which works because the list only shows shared rows plus the current company's. Please add it when convenient, like the other masters.
2. **An update that changes `*_company_id` answers "updated successfully" but keeps the old company.** Keeping the scope is right; please refuse the change with a 400 instead of reporting success. The screen already locks Company on edit.

**Still open:** no user_menus grants exist for 291 / 292.
