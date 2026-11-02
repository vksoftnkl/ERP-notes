---
name: erp-rtk-pending-query-dedupe-abort
description: "RTK Query hands a new initiate() the PENDING request's promise for the same cache key, even with forceRefetch — abort the first and the second never loads; use a GET-mutation for abortable one-shot reads"
metadata:
  node_type: memory
  type: feedback
  originSessionId: f22d285a-769c-4b47-9970-bd9005165d2e
  modified: 2026-09-25T08:36:06.923Z
---

RTK Query's query thunk refuses to start a request whose cache key is already
`pending` — `forceRefetch: true` does NOT override that. A second
`dispatch(endpoint.initiate(sameArgs, {subscribe:false, forceRefetch:true}))`
is handed the first request's promise, so if the first was `.abort()`ed (a
StrictMode double mount, a superseded generation, PgDn back to the ledger just
left) the second resolves with the abort and the data never arrives.

Seen 2026-09-25 on the Ledger Statement ([[erp-ledger-statement-port]]): after
Back the grid never loaded — only a real browser run showed it; the unit tests
(fake client) could not.

**Why:** abort + dedupe on one cache key is a silent no-load, not an error.

**How to apply:** for abortable, uncached one-shot reads through `baseApi`,
declare the endpoint as `builder.mutation` with `method: "GET"`: every
`initiate` is its own request, `.abort()` works, and `.reset()` in `finally`
drops the result from the store. `baseQueryWithAuthHandling` (token + refresh)
is not exported, so this is the way to keep auth without editing the shared
`store/api/baseApi.ts`. Related: [[erp-rtk-lazy-trigger-in-mount-effect]].

**Also without any abort (2026-10-24, quick rate popup):** a mount effect that
does `dispatch(X.initiate(arg, {subscribe:false, forceRefetch:true})).unwrap()`
runs TWICE under dev StrictMode; the second initiate is deduped onto the
pending first one and its `unwrap()` resolved with NO data (TypeError on
`page.items` → a "could not be read" popup), while only one request went out.
Fix used in `features/stocks/selling-price/components/quick-rate-dialog.tsx`:
keep the first `unwrap()` promise in a ref keyed by the args and let every
effect run attach `.then` to that same promise (with its own `active` flag).

**A lazy subscription in ANOTHER screen poisons the load (2026-10-28, sale bill
WhatsApp):** the bill list read a row with `useLazyGetBillQuery` (live
subscription on that bill's `getBill` key). Closing a dialog fired the freshness
bus, which refetched that subscription; a double-click then unmounted the list
and the entry screen's `initiate(key, {subscribe:false, forceRefetch:true})` was
deduped onto that pending refetch — the bill opened BLANK with "Something went
wrong". Only a real-browser run showed it. Fix: a one-shot read that owns no
subscription (`dispatch(endpoint.initiate(key, {subscribe:false,
forceRefetch:true}))` in the click handler). Rule: never hold a lazy-hook
subscription on a key another screen loads with `initiate`.
