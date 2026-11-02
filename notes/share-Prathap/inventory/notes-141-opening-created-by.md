# Notes 141 — Opening Stock GET: who created the document, and when (2026-10-23)

**User's ask:** the Opening Stock screen (menu 44) should show "created on" and
"created by" at the top of the document.

**The data is there; the GET doesn't send it.** `stock.stock_voucher` has
`svh_created_on`, `svh_created_by`, `svh_modified_on` and `svh_modified_by`
(live check: OPN0006 was created 2026-10-23 10:15 by user 019e441b-…). But
`OpeningStockHeaderDto` (`GET /stock/opening/get`) has none of the four. It
already sends `postedOn` / `postedBy` / `postedByName`, which the screen shows.

## Ask

Add four fields to the header of `/stock/opening/get`, built the same way
`posted_by_name` is in `stock-voucher.service.ts` (around line 1766, where
`usr.usr_display_name AS posted_by_name`):

| field            | from                                              |
|------------------|---------------------------------------------------|
| `createdOn`      | `svh.svh_created_on` (ISO timestamp with zone)    |
| `createdBy`      | `svh.svh_created_by` (user id)                    |
| `createdByName`  | `usr_display_name` of that user (LEFT JOIN, null if none) |
| `modifiedOn`     | `svh.svh_modified_on`                             |

`modifiedOn` is optional. If the shared stock-voucher header builder adds them
once, physical stock and transfer get them too.

## Client (already built, 2026-10-23)

`OpeningStockEntry::showAudit()` reads `createdOn` plus `createdByName`
(falling back to `createdBy`) and shows "Created 23-10-2026 10:15 AM by VKPOS"
under the title, with the Posted line under it. Until the fields arrive, the
Created line is just not shown. No other client change is needed.
