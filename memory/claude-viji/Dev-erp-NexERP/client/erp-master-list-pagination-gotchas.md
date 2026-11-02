---
name: erp-master-list-pagination-gotchas
description: "Two traps when wiring a master list page's pager — total lives at data.meta.total, and never echo the response's page/limit back into state"
metadata: 
  node_type: memory
  type: project
  originSessionId: cb64cf7b-ec0a-44b5-9909-b1a03ecc5f68
  modified: 2026-07-18T08:14:09.569Z
---

Master list pages fetch `/configured-grid-sql/run` which replies
`{success, message, data: {items: [...], meta: {page, limit, total}}}`.

Two traps when wiring the "Showing X to Y of Z entries" footer and pager:

1. **The total is two levels deep.** Pagination helpers must descend into
   `root.data.meta`, not just `root` and `root.data`. `features/masters/shared/normalizers.ts`
   does this correctly — prefer it over per-feature copies. A feature-local copy of
   `extractPaginationInfo`/`extractRows` that only scans one level yields
   `totalEntries: null` + `rows: []`, rendering "Showing 0 to 0 of 0 entries" while
   rows display fine. Also watch for helpers redefined *inside* the component body,
   which silently shadow the correct imported ones.

2. **Never feed the response's `page`/`limit` back into state.** The server just
   echoes what was requested, so it adds nothing — and it fights `ReusableTable`'s
   auto-fit sizing: a reply to an in-flight request resets `pageSize` to that
   request's limit, undoing a newer auto-fit value. The table notifies a given
   auto-fit size only once (`notifiedAutoPageSizeRef`), so it never re-sends it and
   the grid deadlocks fetching fewer rows per page than it renders room for —
   leaving records unreachable with Next/Last disabled. Take only the total.

Related: [[erp-audit-logs-list-contract]] (that endpoint instead needs
`include_total=true`), [[erp-grid-columns-config]].

## 2026-10-15: total follows the rows; empty last page steps back

User report: "CRUD master list table doesn't update correctly". Plain add/edit/delete on
City were fine in a real browser; two real bugs were not:
- `useMasterModule` only dispatched `totalEntriesUpdated` from `loadRecords`, but the list
  is ALSO re-read in the background (RTK invalidation from the freshness bus: other tab,
  other user via the server feed, own-editor screens). Rows changed, footer/pager did not
  ("Showing 1 to 7 of 8"). Now `totalEntries` is derived from `listResult.data` (falls
  back to the Redux value before the first response).
- Deleting the only row on the last page: server answers `items: [], page: N, total`
  (no clamp), so the table sat on an empty page. CrudMasterPage now has an effect that,
  once `!loading && data`, sets `currentPage` to `ceil(renderedTotalEntries/pageSize)` when
  it is past it. `setCurrentPage` from the hook is now `useCallback`-stable for that dep.
Browser-test trap: a CDP `Emulation.setDeviceMetricsOverride` dies with the websocket, so
the auto-fit page size snaps back between driver scripts — run a paging scenario in ONE
connection (340 px tall ≈ page size 3 on City master).
