---
name: erp-audit-logs-list-contract
description: "/audit-logs/list quirks — include_total is opt-in (else pagination sticks at 1 page), no sort param exists, and unknown params 400"
metadata: 
  node_type: memory
  type: reference
  originSessionId: 4a39015f-7a53-4678-8e34-b39f5f337f69
  modified: 2026-07-18T05:30:33.865Z
---

`/api/v1/audit-logs/list` (server: `src/modules/audit-log/`, DTO `dto/list-audit-log-query.dto.ts`):

- **`include_total=true` is required for real counts.** It is opt-in to skip the `COUNT(*)`; omit it and `meta.total` / `meta.total_pages` come back **`null`**, not 0. Client fallbacks like `meta.total ?? data.length` then compute `total_pages = 1`, so the next-page button is permanently disabled while the data actually has thousands of rows. (Real numbers seen: 7011 logs / 351 pages at limit 20.) Note `ListMeta` in `utils/types.ts` types `total` as `number`, so the null is invisible to TypeScript.
- **No sorting exists.** There is no `sort`/`order_by`/`direction` param; order is hardcoded `logDate desc, logId desc` to match a composite index. Client-side sorting a server-paginated page would only reorder the visible rows — don't fake it.
- **Unknown params are a hard 400**, not ignored — `main.ts` sets `whitelist: true` **and `forbidNonWhitelisted: true`** globally. This applies to every module's list endpoint.
- **`action` accepts only** `new`/`insert`/`update`/`approve`/`cancel` (case-insensitive; `new` ⇄ stored `insert`, echoed back as `"New"`). **`delete` is NOT valid** → 400 `Unsupported audit action: delete`.
- **Record-scoped history filter** (used by the Record History modal, `features/masters/record-history/page.tsx`): pass **`screen_name`** = the screen *display name* (e.g. `UI Table Master`, not the id) **plus `record_pk` = the record's numeric PK**. `record_pk` matches the numeric key, **not** the `log_pk`/`log_display_name` string echoed in responses — filtering by the displayed string (e.g. `record_pk=Item price list`) silently returns `total:0`. `crud-master-page.tsx` derives it via `resolveAuditHistoryRecordId(row)`.
- `limit` max is 100 (DTO), so a "200 / page" option would 400.
- `date_from`/`date_to` accept `YYYY-MM-DD` (expanded to UTC `T00:00:00.000Z` / `T23:59:59.999Z`, so `date_to` is day-inclusive) or full ISO. Day-expansion is UTC-only — a full-ISO `date_to` is NOT expanded.

Local base URL is `http://127.0.0.1:3011/api/v1` (`NEXT_PUBLIC_API_BASE`) — note the **`/api` global prefix**; probing without it 404s on every route and looks like a missing endpoint. Unauth probe = 401 (exists) vs 404 (wrong path/missing).

Related: [[erp-stale-dist-404]], [[erp-auth-endpoint-verification]], [[erp-server-500-debugging]]
