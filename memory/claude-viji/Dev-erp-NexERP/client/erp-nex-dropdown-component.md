---
name: erp-nex-dropdown-component
description: "components/design-system/dropdown — the config-driven dropdown component (Qt NexDropdown port), its data traps, and what is left to migrate"
metadata: 
  node_type: memory
  type: project
  originSessionId: b6ca6d53-2d17-4a55-8e11-1a65886a395d
  modified: 2026-09-05T07:45:47.717Z
---

`components/design-system/dropdown/` is the React port of the Qt `NexDropdown`: a remote-search
combobox whose columns, formatting, widths, completion and sort are read from
`fixed.dropdown_details` + `fixed.dropdown_columns` rather than hardcoded per screen.
All seven phases landed 2026-09-05: pure `config.ts`/`format.ts`/`api.ts`/`masters.ts` (67 vitest
cases), `use-dropdown-search.ts`, `use-dropdown-keys.ts`, `use-dropdown-master.ts`,
`dropdown-popup.tsx`, `NexDropdownSingle`, `NexDropdownMulti`, RTK endpoints in
`store/api/dropdownApi.ts`, bench page at `app/dev/nex-dropdown` (31 headless-browser checks).
**Left to do: migrating the ~40 existing call sites** (`useLazyConfiguredDropdown`,
`DropdownCombo`, `SearchableSelect`) — nothing in the app uses the new component yet.

Alt+C / Alt+A go through `masters.ts`: `registerDropdownMaster(39, {menuId: 10, render})`, called by
each feature (never re-exported from the barrel, or registrations become eager). Permission comes
from the MASTER's menu id via `usePagePermissions({menuId})` and a refusal is a `toast.error`.

**Alt+C / Alt+A landed on the OLD `DropdownCombo` too (2026-09-19)**, so the sale bill's Existing
Customer field has them without being migrated to `NexDropdownSingle`. Three things that took
finding, all of which the next Alt+C wiring will hit:
- Dropdown ids are resolved BY NAME at runtime, so a feature cannot `registerDropdownMaster` at
  module scope. `masters.ts` now publishes `subscribeDropdownMasters` and `useDropdownMaster` reads
  through `useSyncExternalStore`; register with `useRegisterDropdownMaster(id, {menuId, render})`,
  which keeps a STABLE entry whose `render` delegates to a ref (a fresh entry per render would
  re-render every subscribed field).
- The master screen is the real one mounted `inlineModalOnly` (`hideListPage`), the pattern
  opening-stock already used for Item/Godown. `features/masters/sales/customer/` has
  `inline-customer-master.tsx` (the bridge) + `use-customer-dropdown-master.tsx` (the registration,
  `next/dynamic` so the master stays out of the entry screen's bundle). `CustomerPage` grew
  `inlineModalOnly` / `onCrudControllerReady` / `onModalOpenChange` / `onCustomerSaved`.
- **`controller.openUpdateById()` must NOT be called from the mount phase.** CrudMasterPage reads
  the record through `useMasterModule`, whose `details.run` is an RTK Query LAZY trigger; fired
  while its own screen is mounting it has no subscription, the request is issued and then aborted
  (`net::ERR_ABORTED`), `unwrap()` resolves `undefined`, and the edit modal opens on a BLANK record
  with no error anywhere. Defer it (`setTimeout(…, 0)` inside an effect whose cleanup clears it, so
  React's dev double-mount cancels the first attempt). Same trap `use-quotation-draft` documents and
  dodges with `initiate({subscribe:false})`.

Deliberate divergences from the Qt widget, all commented in place: the list opens on click rather
than on focus (open-on-focus + focus-advance-after-pick cascaded a popup and a request down every
field of the form), multi keeps the list open after a pick, and paging is infinite-scroll at 25 a
page with the total in the footer.

**Why:** every dropdown in the app currently hardcodes its own `idKeys`/`labelKeys`, so the same
Company field is configured five different ways and none of them honour the configured columns.

**How to apply:** load a definition with `/dropdown-details/get?dropdownId=N` — a *list* endpoint,
so the response is `{data:[detail]}` with `columns` nested. Then:

- **The id is the first column BY ORDER, never `dropdown_columns_no === 0`.** The numbering is
  0-based for dropdowns 21/38/39 but **1-based for dropdown 8 (company)**.
- **`dropdown_completion` is null on 9 of 46** (2, 3, 5, 7, 8, 9, 10, 15, 28) — i.e. most of what
  the master screens use — so the "first visible non-id column" fallback is a normal path.
- **Configured columns need not exist in the rows.** Dropdown 8 configures four over a
  `SELECT comp_name, comp_id` that returns two; the extras have no `sql_name` so their key falls
  back to a display name ("company code") no row carries. Drop columns absent from the rows.
- **Widths are weights, not percentages** — they sum to 80 (dropdown 8), 100 and 110. Normalize.
- Every configured column in this deployment is `dataType: 'Text'` today.
- `limit` is capped at 100 server-side; `dropdown_max_visible_items` is a popup height, not a
  request limit (using it as one is what made the Qt list unable to reach match 11).

**Dropdowns 38 (EMPLOYEES) and 45 (SALES AGENTS) declare SQL placeholders** — bare identifiers
`iemp_branch_id` / `isa_branch_id`, textually substituted — so running them without
`dropdown_param` does not return an unfiltered list, it **400s** with
`Invalid dropdown SQL configuration`. They are the only two of the 46 that do.
**Known consequence, not yet fixed:** Quotation and Sale Order wire Salesman and Agent to dropdown
38 through `DropdownCombo`, which never sends `dropdown_param` (its own comment says so), so those
fields cannot be populating on either screen.

Related: [[erp-configured-dropdowns]] (the older per-screen lazy pattern this replaces),
[[erp-quotation-entry-port]], [[erp-client-ui-browser-verification]] (how phase 4 was verified),
[[erp-global-ui-scale]] (the popup measures with `layoutRect`), [[erp-portaled-dropdown-outside-click-bug]].

**`keepEmptyParams` (2026-09-24).** `usableParams` drops any blank param,
because an unguarded `col = itoken::uuid` (dropdown 38 EMPLOYEES, 45 SALES
AGENTS) fails on `''::uuid`. But a GUARDED statement —
`NULLIF('iarea_id','') IS NULL OR cus_area_id = NULLIF('iarea_id','')::uuid`,
which is dropdown 54 CUSTOMERS BY AREA — spells "no filter" as exactly that
empty string, and dropping the key leaves the literal word `iarea_id` in the
SQL so the run answers 400. On screen that looks like a customer list that is
simply empty, which is what "no beat chosen shows no customers" was.

Nothing about a dropdown id says which shape it has, so the caller names the
guarded placeholders: `keepEmptyParams={["iarea_id"]}` on `NexDropdownSingle` /
`NexDropdownMulti`, threaded through `use-dropdown-search` → `DropdownRowsArgs`
→ `buildDropdownRunQuery`, and into the RTK cache key (blank and chosen are
different result sets). Verified live: `{"iarea_id":""}` → 200 with every
customer; no `dropdown_param` → 400 *Invalid dropdown SQL configuration*.
