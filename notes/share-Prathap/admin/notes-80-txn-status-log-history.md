# Notes 80 — txn_status_log: the History dialog, and three gaps in what the log records

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-02 · **For:** Prathap

## What the client now does with `public.txn_status_log`

Every transaction list (TxnMainView) and the Opening / Physical Stock entry screens now have
**History — Ctrl+H**. It opens one shared dialog (`src/shared/ui/txn/txn_history_dialog.*`) that shows
the selected document's trail, oldest first:

| # | Event | Status | When | By | Device | Remarks |
|---|---|---|---|---|---|---|
| 1 | Created | DRAFT | 25-09-2026 13:23:31 | VKPOS | raman | |
| 2 | Posted | DRAFT → POSTED | 25-09-2026 13:23:31 | VKPOS | raman | |
| 3 | Retendered | POSTED | 25-09-2026 13:24:24 | VKPOS | raman | card declined, paid cash |
| 6 | Cancelled | POSTED → CANCELLED | 25-09-2026 17:50:44 | VKPOS | raman | ZT test cleanup |

**No new route is needed.** Following the house rule (lists are registered grids, never a `/list`
route), the rows come from a configured grid that the client registered through `/grid-details/create`:

- **Grid 126 "POPUP - TXN HISTORY"**; SQL in `grids/txn_history_grid.sql`, payload in `grids/txn_history_grid.json`.
- **Params:** `idoc_id`, `iacc_year`, `icompany_id`, all bare and compared as text. The document id alone is enough, because a uuid is unique across doc types; the year prunes the partition and the company scopes the tenant.
- **Sorting:** the client sends `sort_by=tsl_seq_no`. The runner wraps the SQL in a subquery, so an inner ORDER BY would not be kept.
- **Joins:** `public.user_master` for the login name and `fixed.device_master` for the device name.

Verified live on 2026-10-02 through `/configured-grid-sql/run` for OPN0067 (an opening voucher) and bil00756 (a sale bill).

Nothing to build for the dialog itself. The three items below are about **what gets written**.
The dialog is only as good as the log.

---

## A. The actor is lost on sale-bill and quotation CREATED rows (bug)

`tsl_changed_by` is the zero uuid (`DEFAULT_ACTOR`) on **253 rows**:

| doc type | event | tsl_created_by | rows |
|---|---|---|---|
| SALE_BILL | CREATED | VKPOS | 242 |
| SALE_BILL | CREATED | VIJAY | 5 |
| QUOTATION | CREATED / CONVERTED | VKPOS, VIJAY | 6 |

Every other event on the same documents (POSTED, CANCELLED, RETENDERED …) carries the real `usr_id`.

**Cause (from `common/txn-status-log/txn-status-log.helper.ts`):** a non-uuid `changedBy` is stored as `DEFAULT_ACTOR`, and the original value goes into `tsl_created_by`. So the CREATED step of these two documents is being handed the row's created-by **login name** (`sb_created_by` / `sq_created_by` text), not `requestContextService.getUserId()`. The other steps already use the uuid.

- **Paths to check:** `SalesDocStore` (`sales/posting/sales-doc-store.ts` around line 726) as called for the bill's CREATED step, and `quotation.service.ts` around line 1665 (`changedBy: actor`) for CREATED and CONVERTED.

**Ask:** pass the request's user id on CREATED / CONVERTED, as POSTED already does.

**Optional backfill:** set `tsl_changed_by = usr_id` where `tsl_created_by` matches a single `usr_login_name`. That's 253 rows.

**Until then:** the grid falls back to `tsl_created_by`, so the dialog shows "VKPOS" rather than "unknown".

## B. Accounts documents never record the device

`tsl_device_id` is NULL on **every** row of these doc types. The sales and stock documents fill it.

| doc type | rows | device NULL |
|---|---|---|
| RECEIPT | 829 | 829 |
| JOURNAL | 605 | 605 |
| PAYMENT | 483 | 483 |
| CHEQUE_ISSUED | 105 | 105 |
| CHEQUE_RECEIVED | 74 | 74 |

The Device column is therefore blank for every receipt, payment, journal and cheque. "Which counter took this receipt" is exactly the question the History dialog is for.

**Ask:** pass `deviceId` (the session's device; the helper already resolves a code to the master's uuid) on these modules' `appendTxnStatusLog` calls.

## C. Opening / physical / adjustment all file as `STOCK_ADJUSTMENT`

`ck_tsl_src_doc_type` has no OPENING or PHYSICAL, so all four stock vouchers log as `STOCK_ADJUSTMENT`: opening (OPN…), physical, adjustment and issue.

**Not a blocker:** the dialog reads by document id and labels the document from the screen.

**It does matter** to any report that filters the log by doc type, e.g. "every opening cancelled this year", or a day-end "unposted by type". Such a report cannot tell an opening from an adjustment without joining back to `stock.stock_voucher.svh_voucher_type`.

**Ask (low priority):** either add `OPENING_STOCK` / `PHYSICAL_STOCK` to the check and the `TxnStatusDocType` enum, or document that `STOCK_ADJUSTMENT` means "any stock voucher" and reports must join for the kind.

---

## Not asked: field-level "what changed"

`audit.audit_log.log_changed_fields` has the before → after of each update (e.g. `sbStatus DRAFT → POSTED`, line items). It could later explain an AMENDED step. It is not usable from the dialog today:

- `log_entity_id` is NULL on all 2,760 `sale_bill` / `stock_voucher` rows, so it can't be joined to the document reliably.
- The field names are raw DTO keys.

If it's wanted later, the first step is filling `log_entity_id` with the document id.
