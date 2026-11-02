---
name: erp-linked-table-lazy-dropdown-mismatch
description: "ItemLinkedRecordsEditor select columns only filter client-side over whatever options they're given — never reuse a lazy configured-dropdown's options array for one"
metadata:
  type: project
  originSessionId: 01f73713-8c81-471e-9411-f04478ad0d4c
  modified: 2026-07-21T11:01:49.258Z
---

Item Master's Price List table (`item-linked-records-editor.tsx` +
`item-linked-records-search-select-cell.tsx`) renders `type:"select"` row
columns as a fully client-side searchable select: it filters in-memory over
whatever `column.options` array it's handed, with no per-cell async
open/search hook (unlike `ERPDynamicModalField`, which does support
`onSearchOpenChange`/`onSearchQueryChange` — see [[erp-configured-dropdowns]]).

The Price row's Company column (`ipm_company_id`) was wired to reuse
`company.options` — the header item field's **lazy** `useLazyConfiguredDropdown`
state, which starts as just `[DEFAULT_COMPANY_OPTION]` and only grows when
the header field itself is opened/searched. Result: the row's Company dropdown
showed at most 1-2 stale entries and silently fell back to displaying the raw
company UUID for any row whose `ipm_company_id` wasn't already loaded — which
is common, since price rows legitimately carry a **different** company than
the header item (see [[erp-item-company-token-override]]). Users could not
search or pick an arbitrary company for a price row at all.

**Fix** (2026-07-21): gave the price row table its own eagerly-fetched, full
option list — `COMPANY_LOOKUP_QUERY = {module:"companies"}` via the existing
eager `LOOKUP_ENDPOINT` (`/master-lookups/name-id/all-masters`), fetched once
alongside Branch/Unit/Godown/HSN — instead of the header's lazy dropdown
state. Only 5 companies exist, so eager loading is trivially cheap; this
mirrors how Branch already worked in this exact table. The header's own
Company field keeps its lazy dropdown untouched.

**How to apply:** before wiring any `ItemLinkedRecordsEditor`/similar
in-table select column to an options source, check whether that source is a
`useLazyConfiguredDropdown` result — if so, it is NOT safe to reuse for a
row-table column; fetch a proper eager list for the table instead (the
`master-lookups/name-id/all-masters?module=X` endpoint works for `companies`,
`branches`, `units`, `godownLocations`, etc., confirmed via
`toLookupOptions`/`buildLookupOptions`).
