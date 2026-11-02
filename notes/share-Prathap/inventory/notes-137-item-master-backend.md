# Notes 137 — Item master: server audit (2026-10-21)

Read from the nexerp-api-ro snapshot of 2026-10-17 and the live catalog on
192.168.0.106. Paths are under `src/modules/Inventory/` unless they start
with `src/`. Earlier notes 50 / 67 / 68 / 69 / 70 / 71 / 73 were re-checked and are still fixed.

The Qt item screen was fixed the same day on its side: it now sends `null`
(not `""`) for every cleared id, header and child rows.

---

## A. Fix first

### A1. A child row id from another item moves that row onto this item
Every child sync matches by id within this item; when the id is NOT found it is
still pushed to save with that id (item-master-update.service.ts:212-221,
324-333, 401-410, 462-471). The child services then load by id alone and
rewrite its item:

- price: items-price-master.service.ts:369-374, 385 (`ipmItemId`)
- EAN: items-ean-code-master.service.ts:335-340, 362
- unit conversion: item-unit-conversion.service.ts:262-267, 278
- reorder: items-reorder-master.service.ts:302-304, 315

**Repro:** save item B with `ean_codes:[{ean_id:<an EAN id of item A>, …}]`.
The answer is 200, and item A has lost its barcode to B.
**Ask:** if a sent child id isn't one of this item's rows, answer 400 (e.g.
`ITEM_CHILD_NOT_OF_ITEM`). Also add `where … itemId` to each child update.

### A2. An item can never be made shared again (company can't be cleared)
`item_company_id` is `@OptionalUuid` (items-master/dto/save-item.dto.ts:70-71),
and `toOptionalUuid` turns null into undefined (src/common/dto/DtoTransforms.ts:55-64).
So `{item_company_id: null}` keeps the stored company (items-master.service.ts:964-966).

The house rule is that a blank company means the item is shared by every
company, and the item card sends null when Company is cleared. Today the save
answers 200 and the item stays company-owned.

There's a side effect. When an item with a branch moves to another company, the
card clears the branch, but the server keeps the old branch and answers 400
`ITEM_BRANCH_OTHER_COMPANY`.

**Ask:** make `item_company_id` nullable the same way `item_branch_id` is
(null clears, omitted keeps). Do the same for `item_expiry_days`,
`item_intimate_before_days` and `item_sort_order`, which can't be set back to
NULL either.

### A3. Null on any yes/no field gives a 500
`OptionalBoolean` is `IsOptional()+IsBoolean()` (src/common/dto/dtoDecorators.ts:237),
so null skips validation and is written straight through
(items-master.service.ts:1080-1200). All 27 item boolean columns are NOT NULL,
so the Prisma error falls to 500 `INTERNAL_ERROR`. The same happens for
`ipm_is_active`, `iuc_is_*` and `ir_is_active`.

**Ask:** treat null as "not sent", or give a 400 that names the field.

### A4. Removing a unit leaves its prices, barcodes and reorders live
syncUnitConversions soft-deletes the conversion row (item-master-update.service.ts:226-229)
but not the rows that point at that iuc_id.

- The barcode keeps scanning.
- GET returns the rows with the unit name null.
- The next save that echoes them fails with 400 `ITEM_UNIT_NOT_CONVERTED`.

**Ask:** either soft-delete those dependants in the same transaction, or refuse
the removal with a 409 that lists them.

---

## B. Data integrity

- **B1. Reorder unique index lacks the branch.** `uq_ir_item_unit_godown` is
  (item, uc_unit, godown) WHERE NOT deleted, but the service key is branch +
  unit + godown (item-master-update.service.ts:440-445). Two branches with the
  same unit + godown get P2002, reported as "Item reorder already exists".
  Rows with a NULL unit or godown are never protected. *Ask:* add
  `ir_branch_id` to the index, plus a `NULLS NOT DISTINCT` / COALESCE form.
- **B2. Reorder update nulls the unit.** `irUcUnitId: dto.ir_unit_id ?? null`
  (items-reorder-master.service.ts:316) breaks the "omitted keeps" rule that
  every other field follows.
- **B3. Barcode case.** `uq_ean_code` is case-sensitive, but
  `/master-lookups/item-by-barcode` matches case-insensitively
  (src/modules/master-lookup/master-lookup.controller.ts:184-190). So "ab12"
  and "AB12" can be live on two items and a scan becomes ambiguous. *Ask:* a
  unique index on `upper(ean_code)`. The lookup also ignores
  `item_default_barcode`, so a barcode typed only in the header never scans.
  Either check that column as well, or document that the header barcode must
  also be an EAN row.
- **B4. GET /item-price-details by barcode is nondeterministic.**
  `findFirst({itemDefaultBarcode})` has no company filter and no order
  (item-price-details/item-price-details.service.ts:25-31).
- **B5. No duplicate checks on code or SKU.** `item_code` and `item_sku` have
  no uniqueness. `item_default_barcode` is not checked against `uq_ean_code`.
- **B6. Item name uniqueness is global and case-sensitive.**
  `uq_item_name_en_global (item_name_en)` means "Rice" and "rice" can both
  exist, while company A and company B can't both have "Rice". *Decision
  needed with VK:* unique per company (shared items unique across all), on
  `lower(item_name_en)`.
- **B7. Header base unit vs conversions.** `item_base_unit_id` is never
  compared with the conversion row marked `iuc_is_base_unit`
  (items-master.service.ts:763, 980-982). The Qt card always sends the
  matching pair; other clients may not.
- **B8. No duplicate refusal inside one payload for EANs or unit
  conversions.** Prices and reorders get `refuseDuplicateKeys`; EANs and
  conversions don't. Two new rows with the same code come back as 409
  `ITEM_EAN_CODE_TAKEN`, which reads as if another item has the code.
- **B9. References with no FK and no check.** `item_brand_id`,
  `item_section_id`, `item_supplier_id`, `item_cust_group`, `item_mfgr_id`,
  `item_company_category_id` and `item_barcode_sticker_id` have no FK, and
  neither do `ir_branch_id` and `ir_godown_id` on item_reorders. Any UUID is
  stored, and GET then shows a null name. Deleted or inactive
  group/category/unit/tax/preset masters pass the FK too (only the inherited
  defaults filter on active).

## C. Scope and security

- **C1. No company scoping.** Inventory never calls
  `requestContextService.getCompanyId()`. Any logged-in user can GET, update,
  delete or restore any company's item by id. `/items/bulk-load` with no
  company returns every company's items (items-master.service.ts:398-408).
- **C2. Actor can be spoofed.** `item_created_by` and `item_modified_by` (and
  `ean_*`, `ir_*`) from the body override the token user (`resolveActor`,
  src/common/utils/module-shared.utils.ts:269-281). *Ask:* always take the
  actor from the token.
- **C3. Company moves are unchecked.** An item can move from shared to A, or
  from A to B, while another company holds its stock or prices. The stock
  check runs only on delete.
- **C4. Price row scope.** A price row's company and branch aren't checked
  against the item's company or against each other, and `ipm_godown_id` isn't
  checked against the branch (items-price-master.service.ts:575-580).

## D. Smaller ones

- **D1. Standalone child routes skip the item's rules.** `/item-prices`,
  `/item-ean-codes`, `/item-unit-conversions` and `/item-reorders`:
  - create/delete don't check that the item is live;
  - delete is still a TOGGLE, so a second DELETE restores
    (items-price-master.service.ts:265, items-ean-code-master.service.ts:220-221);
  - EAN and reorder create don't check that the unit belongs to the item.
- **D2. Restore** maps only unique violations (items-master.service.ts:628-652).
  An `ex_ipm_overlap` 23P01 gives 500. Restore also doesn't call
  `priceBucketService.rekeyItem`.
- **D3. Delete guard checks only stock_balance** (items-master.service.ts:87-94).
  It doesn't look at open orders or quotations, promotion or loyalty items, or
  use as another item's `item_packing_item_ids`.
- **D4. `handleWriteError` maps every P2002 to `ITEM_NAME_TAKEN`**
  (items-master.service.ts:1299-1306). A policy-sync 23P01 inside the same try
  gives 500.
- **D5. No Min(0)** on `item_expiry_days`, the intimate days, the reorder
  levels and days, or the iuc weight. Reorder levels have no DB CHECK either.
- **D6. HSN** is validated only for length ≤ 10. No digits-only check.
- **D7. `ipm_profit_type`** has DB default 'MANUAL', which isn't one of the
  allowed values.
- **D8. `enableImplicitConversion: true`** (src/common/validation/house-validation.ts:33-34)
  turns a body string `"false"` into `true`.
- **D9. Grid 71's `grid_description`** still says "strict" company. The SQL
  was fixed (notes 73); only the text is stale.

## E. ui-table-masters: append trap (found through the item price grid)

On 2026-10-12, ui_table 17 (ITEM MASTER - PRICE) got 22 extra column rows
(ids 1918-1939). They reused column numbers 4-33, and the save was made as
VIJAY from a Desktop client. `/ui-table-masters/create` without
`replaceColumns` APPENDS. The Qt grid then hid Unit, Godown, Cost, Price A
margin, MRP and Disc %, so no new price row could be entered.

- Fixed on the box: the 22 rows were soft-deleted through
  `/ui-table-masters/column-delete`. Copy:
  `inventory/ui_table17_rows_removed_2026-10-21.csv`.
- Fixed in Qt: NexTable now keeps the first row of a column number and
  ignores the rest.
- **Ask:** refuse a create that would leave two live rows with the same
  `ui_tbl_clm_no` on one table, and add a partial unique index
  `(ui_tbl_clm_table_id, ui_tbl_clm_no) WHERE NOT ui_tbl_clm_is_deleted`.
  Also: which screen made the 10-12 save? Its save path should send
  `replaceColumns: true`.
