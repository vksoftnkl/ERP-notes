# Textile auto batch — addendum 2: size, colour, model on the purchase line

Date: 2026-10-23. New file; it edits neither plan-textile-auto-batch.md nor any earlier plan.

## User decisions (2026-10-23)

- Size, colour and model are chosen on the **purchase line** (and on opening stock), not on the item master. The item stays generic ("Shirt").
- **Size** and **Colour** are masters under Inventory: picked lists with quick-add.
- **Model** is free text, with suggestions drawn from models used before.
- Fabric, season and supplier article are **not needed now**. They stay in plan-textile-lot-tag-gaps*.md for later.

## E1. Masters (new, schema `inventory`)

**`inventory.item_size_master` (prefix `isz_`)**

| Column | Notes |
|---|---|
| `isz_id` uuid PK | |
| `isz_company_id` uuid NULL | NULL = shared by every company |
| `isz_code` varchar(20) NULL | |
| `isz_name` varchar(50) NOT NULL | |
| `isz_size_set` varchar(20) NULL | ALPHA / WAIST / KIDS / free; groups sizes for the size grid later |
| `isz_sort` int NOT NULL DEFAULT 0 | S < M < L < XL; 28 < 30 < 32 — reports sort by this, not by name |
| `isz_is_active`, `isz_is_deleted`, audit, `isz_sync_date` | the usual set |

- `UNIQUE (COALESCE(isz_company_id, nil-uuid), upper(isz_name)) WHERE NOT isz_is_deleted`

**`inventory.item_colour_master` (prefix `icl_`)**
- Same columns minus `size_set`: `icl_id`, `icl_company_id` NULL, `icl_code`, `icl_name`, `icl_sort`, active / deleted / audit / sync.
- Same unique rule on name.

**Routes and lists**
- Routes: `/api/v1/item-sizes/{save,get,delete}` and `/api/v1/item-colours/{save,get,delete}`.
- No /list route: each list is a registered grid (house rule).
- `save` with only a name is the quick-add path. A 409 on an existing name returns that row's id, so the client simply uses it.
- Delete is refused while a batch detail row (E2) uses the row; the client then offers "deactivate".
- Dropdowns: one each for size and colour (active rows, company + shared, ordered by sort then name). **Prathap registers these; I can't write dropdown rows.**

## E2. Where a batch's size / colour / model live: `stock.stock_batch_detail` (prefix `sbd_`)

**Keyed by the batch no, not by the lot id.** Why:
- An auto batch no is unique in the company by construction (plan D1: branch + year + counter).
- A Re-lot (price change) keeps the batch no, so it needs **no copy**.
- A draft has a batch no from its first save, before any lot exists, so the draft line needs no new column.

| Column | Notes |
|---|---|
| `sbd_id` uuid PK | deterministic uuid-v5 of (company, upper(batch_no)), the same pattern as `lotIdentityUuid`; offline-safe |
| `sbd_company_id` uuid NOT NULL | |
| `sbd_batch_no` varchar NOT NULL | stored upper-case |
| `sbd_item_id` uuid NOT NULL | for the model suggestions and the reports |
| `sbd_size_id` uuid NULL | FK → isz_id |
| `sbd_colour_id` uuid NULL | FK → icl_id |
| `sbd_model` varchar(100) NULL | |
| `sbd_src_doc_type`, `sbd_src_doc_id`, `sbd_src_line_id` | the inward line that created it (opening now, purchase later) |
| active / deleted / audit / `sbd_sync_date` | |

**Indexes**
- `UNIQUE (sbd_company_id, sbd_batch_no) WHERE NOT sbd_is_deleted`
- `(sbd_company_id, sbd_item_id, upper(sbd_model))` for suggestions
- `(sbd_company_id, sbd_size_id)` and `(sbd_company_id, sbd_colour_id)` for reports

**Write rules (NestJS only, no DB function or trigger)**
- An inward save (opening stock now, purchase later) writes or updates one row per line that has an auto batch. This happens in the same transaction as the batch number (plan D3).
- Editing a draft line updates its row.
- Deleting a draft line or document soft-deletes its rows.
- Once the document is posted, the row is fixed. A wrong size or colour after posting is fixed with `/stock/batch-detail/update` (one route, a reason is required, logged in txn line audit). Stock does not move.
- **No change** to `stock_lot`, `stock_voucher_item` or `sale_bill_item`.

## E3. Reads

1. **Scan route (plan D5)** `GET /stock/lots/by-batch` also returns sizeId / sizeName, colourId / colourName and model, through a left join on `sbd`.
2. **Model suggestions** `GET /api/v1/stock/batch-detail/models?companyId=&itemId=&q=`: the distinct models used for that item, most recent first, at most 20.
3. **Bill reads** (`/bills/get`, DC, return) add sizeName, colourName and model per line, joined on (company, sbiBatchNo). Nothing new is stored on the sale line.
4. **Printing:** the bill print data source gets the same three fields, so the invoice can print "Shirt · XL · Maroon · Slim fit".
5. **Reports (later):** stock and sales by size and colour = lot or sale line → batch no → sbd. Sort by `isz_sort`.

## E4. Qt

- **Opening stock (and purchase later):** the line gains Size (dropdown + quick-add), Colour (dropdown + quick-add) and Model (text with a suggestion popup). They are shown only when the company setting `stock.auto_batch_no` is on.
- **Sale bill / DC / return:** read-only Size, Colour and Model columns, filled by the scan. They are hidden through the ui_table when the setting is off.
- **Size and Colour master screens:** NexCrud list + popup, one menu each under Inventory masters (menu ids from Prathap).

## Build order (extends plan-textile-auto-batch.md)

1. **Server S1:** as in the first plan, plus the E1 tables, routes and dropdowns, plus E2 written by the opening stock save.
2. **Server S2:** the scan route with E3.1, plus E3.2 and E3.3.
3. **Qt:** the master screens, then the opening stock columns, then the sale bill scan and columns.

## Rule checks

- **Offline:** sbd_id is deterministic and batch nos are unique by construction, so no two branches can mint the same row. No running totals.
- **No DB functions or triggers.**
- **New tables only where nothing existing can hold the data:** two masters plus one detail table; stock_lot is untouched.
- **House rules:** no /list routes; masters are company-nullable (NULL = shared).
