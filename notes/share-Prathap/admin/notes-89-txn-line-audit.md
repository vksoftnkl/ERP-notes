# Notes 89 — Transaction audit: one BEFORE/AFTER snapshot per save, stable line ids

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap
**Plan:** `plan-txn-line-audit.md` · **Mockup:** `txn_audit_history_mockup.png` (both in this folder)

## Why

The user wants an admin to see **exactly what each save of a transaction changed**. Their example:
opening stock saved with 3 items, then some lines changed, dropped or added. Today the audit can't
show that:

- Stock voucher edits log `{svhId, lineCount}` "Opening stock lines replaced" with **no line content**
  (`stock-voucher.service.ts` ~1293, logged as `insert` because the old lines are never read back).
- The header `update` row has `log_changed_fields = NULL`.
- Bill / quotation log each line on insert only, never before/after on an edit.
- Stock voucher lines are **deleted and re-inserted** every save, so a line has no identity across saves.

## The ask

### A. One audit row per transaction save (shared helper, every transaction service)

Inside the save's transaction:
1. **Before** touching anything, read the document in the **same shape as its `/get` response**
   (header + `items[]` with item / unit / godown names).
2. Save as today.
3. Read it again: **after**.
4. Write **one** `audit.audit_log` row: `log_table_name` = header table, `log_pk` **and**
   `log_entity_id` = doc id, `log_original_record` = before (NULL on create), `log_modified_record` =
   after, `log_changed_fields` = **header** fields `{field:{from,to}}` (as masters), `log_device_name`
   = the device / counter, plus the revision number (Q1).
5. **Drop** the separate line rows ("lines replaced", "Bill item created") for that save. The
   snapshot carries them.

A suggested home is `AuditLogService.logDocumentRevision({ screen, docId, before, after, actor, device }, tx)`,
which computes header `changedFields` and the rev no. Each service passes its own `getPayload(tx, id)`.

### B. Stable line ids on update (stock vouchers first)

On update: a line **with** its id (`sviId`) updates that row, a line **without** one inserts, and ids
**missing** from the payload are deleted. That's the same rule the bill and quotation DTOs already
document for `sbiId` / `sqiId`. Please add `sviId?` to the stock voucher line DTO and switch
`replaceLines` from `deleteMany + createMany` to this upsert. The client will send the id; it
already holds it in the grid.

### C. Start with opening stock (audit screen 67), then the other 17

Opening stock first: it's the user's example and has the worst gap. Then physical stock,
adjustment, transfer, then the sales and accounts documents (audit screens 72–106, type
`transaction`).

## Questions

1. **Revision number:** a new `log_rev_no smallint` column on `audit.audit_log` (preferred:
   sortable, and the History list can show it), or `log_notes = 'rev N'`? Count per `(log_pk,
   log_screen_id)`.
2. **Status steps:** POSTED / CANCELLED / AMENDED already go to `txn_status_log`. Should they *also*
   write a snapshot row? We'd like **yes for AMENDED** (it changes content), and no for pure status
   changes.
3. **Size guard:** any limit you want on snapshot size? We expect 500-line documents at most. jsonb is
   compressed, so we see no need.

## Nothing new on the read side

`/audit-log/list?screen_id=<n>&record_pk=<doc id>` already returns `log_original_record` and
`log_modified_record`. The Qt History dialog will read it and compute the diff (added / dropped /
changed lines, header from → to) itself.

## How we'll check it (live, opening stock)

1. Create an OS doc with 3 lines → 1 audit row, rev 1, `original` NULL, `modified` has 3 items.
2. Edit: change qty on line 1, drop line 2, add a new line → 1 row, rev 2, `original` 3 items,
   `modified` 3 items. Line 1 keeps its `sviId`, line 2's id is gone, and the new line has a new id.
3. The header `changedFields` lists only what changed (e.g. remarks, total value).
4. No "lines replaced" row is written any more.
