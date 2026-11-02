---
name: erp-record-history-tabs
description: Record History modal folds all audit rows into one date/time accordion (sub-entity tabs removed 2026-07-28); composite screens fetch-all + paginate groups client-side
metadata: 
  node_type: memory
  type: project
  originSessionId: d004256d-3bb5-4248-82ea-6ddb498804e0
  modified: 2026-07-28T06:02:35.934Z
---

Record History modal ([features/masters/record-history/page.tsx] `RecordHistoryModalContent`) shows **one accordion item per date/time (minute precision)**, not one row per audit log. The per-sub-entity tabs (Item / Unit Conversion / EAN Codes / Reorders / Price List, added 2026-07-25) were **removed on 2026-07-28** at the user's request: one save fans out across child tables and they read that as a single change. Expanding a group lists each sub-entity with its own label chip, action badge, notes and Field/Old/New diff table.

Key facts that shaped it:
- The server's `/audit-logs/list` treats `screen_name="Item Master"` as a **composite** screen and fans out to child screens (`RELATED_AUDIT_SCREENS_BY_SCREEN_NAME`), merging their rows into one paginated stream. Item List passes `auditHistory={{ screenName: "Item Master" }}` ([item-master-page.tsx:5343]).
- The reliable sub-entity discriminator is **`screen_name`** (`Item Master`, `Item Unit Conversion Master`, `Item EAN Code Master`, `Item Reorder Master`, `Item Price Master`), NOT `log_table_name` (inconsistent free text). `HISTORY_SUB_ENTITY_LABELS` maps those to short chip labels; membership in that map is also what flags a composite screen.
- The list DTO has NO table/screen sub-filter param and `forbidNonWhitelisted` 400s unknowns. Composite screens therefore **fetch all pages** (limit 100, cap 50) and paginate the **groups** client-side (10/page) — a date/time group must not straddle a server page. Plain masters keep the server-paginated path and hide the Sections column.

See [[erp-audit-logs-list-contract]], [[erp-item-entry-spec-port]].
