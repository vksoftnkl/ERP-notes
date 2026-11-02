# Notes 142 — ONE previous / next route for every document type (2026-10-23)

**User's ask:** Previous / Next (Ctrl+PgUp / Ctrl+PgDn) on every transaction
screen, by company, branch and voucher type, from the backend.

**Today:** only accounts has it — `/receipts/adjacent`, `/payments/adjacent`,
`/vouchers/adjacent` (`voucher-lookups.service.ts` `adjacent()`). Opening Stock
walks its list grid (99) on the client for now, which reads the whole list.

**Why one route works:** all eight document headers have the same shape (live
check 2026-10-23):

| docType (proposed)  | table                         | id      | date               | slno                | type filter                    |
|---------------------|-------------------------------|---------|--------------------|---------------------|--------------------------------|
| `OPENING_STOCK`     | stock.stock_voucher           | svh_id  | svh_doc_date       | svh_slno            | svh_voucher_type = 'OPENING'   |
| `STOCK_ADJUSTMENT`  | stock.stock_voucher           | svh_id  | svh_doc_date       | svh_slno            | svh_voucher_type = 'ADJUSTMENT'|
| `PHYSICAL_STOCK`    | stock.stock_voucher           | svh_id  | svh_doc_date       | svh_slno            | svh_voucher_type = 'PHYSICAL'  |
| `STOCK_TRANSFER`    | stock.stock_voucher           | svh_id  | svh_doc_date       | svh_slno            | svh_voucher_type = 'TRANSFER'  |
| `SALE_BILL`         | sales.sale_bill               | sb_id   | sb_bill_date       | sb_bill_slno        | sb_doc_type                    |
| `QUOTATION`         | sales.sale_quotation          | sq_id   | sq_quote_date      | sq_quote_slno       | sq_doc_type                    |
| `SALE_ORDER`        | sales.sale_order              | so_id   | so_order_date      | so_order_slno       | so_doc_type                    |
| `DELIVERY_CHALLAN`  | sales.sale_dc                 | sdc_id  | sdc_dc_date        | sdc_dc_slno         | sdc_doc_type                   |
| `DC_RETURN`         | sales.sale_dc_return          | sdr_id  | sdr_return_date    | sdr_return_slno     | sdr_doc_type                   |
| `SALE_RETURN`       | sales.sale_return             | sr_id   | sr_return_date     | sr_return_slno      | sr_doc_type                    |
| `VOUCHER` (+typeCode)| accounts.acc_voucher_header  | avh_voucher_id | avh_voucher_date | avh_voucher_slno | avh_voucher_type_id → vchr_type_code |

Every one also has `*_company_id`, `*_branch_id`, `*_acc_year` (the partition
key), `*_status`, `*_is_deleted` and `*_created_on`. (Please check the exact
`svh_voucher_type` values for the stock rows; the type codes in the first column
are only my proposal.)

## Ask

`GET /txn/adjacent`

| param       | required | meaning |
|-------------|----------|---------|
| `docType`   | yes      | one of the registry keys above |
| `companyId`, `branchId`, `accYear` | yes | the scope; accYear picks the partition |
| `docId`     | no       | the document on screen; absent = a new one (prev → the newest) |
| `direction` | yes      | `prev` / `next` |
| `typeCode`  | no       | VOUCHER only: a single voucher type, as `/vouchers/adjacent` |
| `status`, `fromDate`, `toDate` | no | same filters the list has |

Response: `data.doc = { docId, companyId, branchId, accYear, refno, date, status }`,
or `data.doc = null` at either end. That is enough for the screen to call its own
`/get`.

**Rules (same as `/vouchers/adjacent`):**
- Walk order is `(date, slno, created_on, id)`: `<` and DESC for prev, `>` and ASC for next. This is the order the lists show.
- The current doc is looked up under all four keys (id, company, branch, year); a miss is a 404.
- Rights: only documents on menus the user can VIEW (`user_menus.um_can_view`), exactly as vouchers join `user_menus` today.
- `*_is_deleted = false`. Cancelled documents are included, because the lists show them.
- The registry is a fixed server-side map from docType to table and columns. No identifier ever comes from the request, so the SQL stays safe.

`/vouchers/adjacent`, `/receipts/adjacent` and `/payments/adjacent` can stay
as they are, or become thin wrappers over this route.

## Client

The keys and their handling are already built: `TxnKeys` Previous / Next =
Ctrl+PgUp / Ctrl+PgDn, live on Opening Stock (menu 44). When this route lands,
`OpeningStockEntry::walkList()` is replaced by one call, and every other
transaction screen gets Previous / Next the same way.
