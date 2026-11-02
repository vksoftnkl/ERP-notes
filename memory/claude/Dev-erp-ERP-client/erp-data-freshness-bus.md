---
name: erp-data-freshness-bus
description: Client-wide revalidation so popups/modals/dropdowns follow DB changes without a tab reload - lib/data-freshness bus + RTKQ refetch defaults
metadata:
  type: project
---

Added 2026-09-04: every list, modal, popup and dropdown revalidates without reloading the tab.
No server push exists (plain REST), so it is event-driven revalidation, not polling.

`lib/data-freshness` is the one place that decides *when* to re-read:
- signals: window focus, `visibilitychange`→visible, `online`, a save in this tab, a save
  in another tab (BroadcastChannel `erp-data-refresh`, localStorage-ping fallback), `manual`.
- `useDataRefresh(handler, {enabled, minIntervalMs=3000, reasons})` - handler in a ref, skips
  while the tab is hidden; mutation/remote-mutation bypass the throttle, `manual` does not.
- `useDataRefreshToken()` exists for effect deps but the wired code uses the
  useCallback + `useEffect(() => loadX(), [loadX])` + `useDataRefresh(loadX)` shape instead
  (no lint-disable needed).

Wiring:
- `baseApi`: `refetchOnMountOrArgChange: 2` + `refetchOnFocus`/`refetchOnReconnect`, plus
  `setupListeners` in `makeStore`. Tag list exported as `API_TAG_TYPES`.
- `store.ts`: bus subscriber invalidates ALL tags on mutation/remote-mutation/manual
  (skipped for this tab's own `rtk:` scoped events - the endpoint's invalidatesTags did it);
  `announceMutations` middleware puts fulfilled RTKQ mutations on the bus.
- `useApi`: any non-GET success calls `notifyDataChanged(path)`. `autoRefresh` (default OFF)
  repeats the last GET quietly and swaps `data`; `refresh()` is returned for manual use
  (use-master-crud + account-ledger call it so rows and paging metadata apply together).
- `ERPDynamicModalForm.openModal` fires `requestDataRefresh("modal-open")` - covers the case
  of a tab that never lost focus.
- `GlobalLoader` stays quiet for 2s after a refresh signal, or the bar flashes on every alt-tab.
- ~35 load-once effects (widget config, lookups, ui_table/grid config, menu tree, bulk-load
  popup filters, stock list popups, audit/record history) were converted to the callback shape.

Verified in a real browser (see [[erp-client-ui-browser-verification]]): a city INSERTed
straight into Postgres appears in an already-open City dropdown after a focus signal, and on
merely reopening the modal; idle tabs issue zero requests.

## 2026-10-15: live cross-user feed + the three gaps behind "saves don't show up"

User report: "created/updated/deleted doesn't reflect instantly". Three causes, all fixed:

1. **Server cache outlived writes.** `HttpCacheInvalidationInterceptor` only purged the
   written resource's own prefix (`/customers*`), but cached reads are generic
   (`configured-grid-sql/run`, `dropdown-details/run`, `master-lookup`, stock balances,
   audit logs), and it purged in an un-awaited `tap(async)` — the client's instant
   re-read could beat the purge. Now: any successful write purges ALL `http-cache:*`,
   awaited (bounded 2 s) before the response; the 12 leftover `@CacheTTL(60)` are 1.
   Read-only POSTs (`/validate /verify /preview /resolve /format/test /slip-check`) skip it.
2. **Own-tab RTK saves only refreshed their declared tags** (~20 api files declare none).
   The store subscriber no longer skips `rtk:` events → invalidate ALL tags. No double
   fetch: announceMutations dispatches during `next(action)`, before RTKQ's handler runs,
   so 'delayed' invalidation merges both into one batch (pinned by
   `store/data-change-invalidation.test.ts`). GET-mutations are no longer announced
   (baseQuery now returns `meta`; method read off `meta.baseQueryMeta.request`).
3. **Nothing pushed other users' saves.** Server `src/common/data-changes/`:
   `GET /data-changes/stream` (Nest `@Sse`, bearer-auth, `ready{bootId}` / `change{resource,method,at}`
   / `ping` 25 s), published by the write interceptor, own `sid` filtered out;
   in-process Subject (single PM2 instance — multi-instance would need Redis pub/sub).
   `TimeoutInterceptor` skips `__sse__` handlers (else 15 s cut). Client
   `lib/data-freshness/server-change-feed.ts` (fetch-read SSE, Web Lock = ONE stream per
   browser, relays via BroadcastChannel as `remote-mutation`, 2 s leading+trailing
   throttle, backoff 1→30 s, re-reads once on reconnect to the SAME bootId) mounted by
   `components/layout/server-change-feed.tsx` in app/layout. Hidden tabs skip remote
   invalidation (refetchOnFocus covers the return).

Verified live 2026-10-15: untouched browser on City master showed another session's
rename ~250 ms after the save. Each relay costs ~14 GETs per open screen (list + config
+ lookups) — the reason for the 2 s throttle. Rows edited straight in Postgres bypass
the feed (focus still catches them). Till comments in `features/sales/till/**` still
justify quiet queries by the OLD "every mutation is announced" rule — harmless, left.
