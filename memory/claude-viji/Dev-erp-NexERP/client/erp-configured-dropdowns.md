---
name: erp-configured-dropdowns
description: "Lazy server-side dropdowns via /dropdown-details/run (configured dropdown_id), the alternative to eager master-lookups"
metadata: 
  node_type: memory
  type: reference
  originSessionId: 26047153-64ea-43e1-be9c-6f0da8504cf1
---

`GET /api/v1/dropdown-details/run?dropdown_id=<n>&page=1&limit=20&search=<q>&dropdown_param=<json>`
runs configured SQL from `fixed.dropdown_details` and returns `{ data: { items, total } }` — the
dropdown analogue of `configured-grid-sql/run`. It supports **server-side search** (`search=`) and
pagination, so it's the way to do lazy, on-open, searchable dropdowns instead of the eager
`/master-lookups/name-id/all-accounts-and-masters` endpoint (see [[erp-name-id-lookup-empty-name]]).

**Since 2026-09-18 no screen hardcodes a `dropdown_id`.** `lib/configured-dropdowns` keys each
dropdown by the `dropdown_name` Dropdown Master stores it under and resolves the id at runtime —
`useDropdownId(key)` in React, `getDropdownId(key)` off it, and `useLazyConfiguredDropdown` takes
`dropdownKey` in place of `dropdownId`. Dropdown Master lists **Desktop rows only**, so the old
un-typed rows every screen used to read (2, 3, 5, 6, 8, 9, 10, 11, 28) were invisible there; those
call sites now read the Desktop rows beside them (29 CUSTOMER STATES, 33 CUSTOMER GROUPS, 32
BRANCHES, 42 ITEMS, 22 COMPANYS, 21 GST - STATE CODES, 13 AREA LIST, 31 SUPPLIER GROUP). The one
number left is printing's `PURPOSE_DROPDOWN_ID = 47` — a Web row whose Desktop twin (48) selects
`ppo_id, ppo_code` only, and the picker needs `ppo_name`/`ppo_src_module`. See
[[erp-configured-grid-registry]] for the same pattern over grids.

The ids below are still the reference-database values and the column shapes are still what each
dropdown returns — read them as the map behind the registry's names, not as something to hardcode.

Configured `dropdown_id`s (from `fixed.dropdown_details`): 6=items, 21=GST state codes
(`state_code,state_name`), 22=companys (`comp_id,comp_name`), 23=account groups
(`acc_group_id,acc_group_name`), 24=branch active list (`br_id,br_name`). The Customer master
(`features/masters/sales/customer/`, `customer-dropdowns.ts`) uses a different set: 5=branch
(`br_id,br_name`), 8=company (`comp_name,comp_id`), 9=state code (`state_code,state_name`),
10=area (`arm_id,arm_name,arm_alias,arm_short`), 28=customer group (`cgr_id,cgr_branch_id,cgr_name`).
The City master (`features/masters/sales/city/page.tsx`) State select uses 2=state dropdown
(`SELECT stm_id, stm_name FROM sales.state_master`) — NOTE this is the `states` master (stm_id), a
different entity than the `state_code`/`fixed.state_codes` dropdowns 9/21.
The Supplier master (`features/masters/purchase/suppliers/`, `dropdowns.ts`) uses 8 company, 5 branch,
9 state (field value = state NAME, like the ledger/region pattern), 11 supplier groups. **Dropdown 11
shipped with a broken `dropdown_sql`** that quoted camelCase identifiers (`SELECT "spgId","spgName"...`)
which don't exist — the real columns are snake_case (`spg_id,spg_name,spg_alias,spg_short`); fixed the
SQL + each `sql_name` to snake_case and set filter=true. Lesson: a dropdown returning `column "X" does
not exist` (400) has wrong column names in its stored `dropdown_sql`/`dropdown_columns_sql_name`.
Another 400 mode: `Comments are not allowed in configured query` — the stored `dropdown_sql` has a
`--` line comment (dropdown 26 GODOWNS had `--AND gdl_godown_id, gdl_branch_id`); strip the comment
from `dropdown_sql`. Branches master uses 8=company, 9=state (name), 26=godown (gdl_id/gdl_name).
The Item Group master (`features/masters/inventory/item-group/page.tsx`) Parent Group select uses
17=ITEM GROUPS (`itg_id,itg_name,itg_short` from `inventory.item_group_master` where active) — this
one was ALREADY search-configured (filter=t on name/short), so no DB fix was needed; `/item-groups/get`
returns `itg_parent_id`+`itg_parent_name` for edit-seeding. Always check `dropdown_columns` first —
some configured dropdowns already work.
Reusable hook: `features/masters/shared/use-lazy-configured-dropdown.ts` exports
`useLazyConfiguredDropdown({dropdownKey, idKeys, labelKeys, defaultOption})` ->
`{options, handlers, seedSelected}`. Wire `handlers` (onSearchOpenChange/onSearchQueryChange/
onValueChange) + `serverSearch:true` onto the select field; call `seedSelected(id,name)` in
mapFormValues (edit) and `onModalOpenChange` master-create (reset). Pass STABLE module-constant
configs. Used by User Administration (8=company, 5=branch). Prefer this hook over the older
per-page inline copies (city/item-group/item-brand/item-category each still inline the pattern).
Rows use the snake_case SQL column names (NOT the `id`/`name` shape master-lookups returns), so build
options with explicit id/label keys per dropdown.

**Server-side `search=` only works if `fixed.dropdown_columns` is configured for that dropdown_id**:
each searchable column needs both `dropdown_columns_sql_name` set (the real SQL column) AND
`dropdown_columns_filter=true`. With none set, `deriveSearchableFieldNamesFromColumns` returns []
and the run query matches NOTHING (search silently returns 0 rows). 21-24 ship configured; 5/8/9/10/28
shipped with empty sql_name + filter=false and were fixed by `UPDATE fixed.dropdown_columns` on their
name/code columns (state made searchable by both code and name). This config is runtime data (set via
dropdown-designer), not seeded by a migration.

Lazy pattern used on the Account Ledger form (`features/masters/accounts/account-ledger/`,
`dropdowns.ts` + `page.tsx`): no fetch on mount; fetch when the field opens (delay 0) and on
debounced typing (`search=`, DEBOUNCE_MS) via a `useEffect` keyed on `[openSearchField, searchQueries]`;
one `useApi(DROPDOWN_RUN_ENDPOINT)` hook per kind (independent abort); skip client-side filtering for
these fields (server already filtered); seed each field with just its selected `{value,label}` from
the record's `ledCompanyName`/`ledBranchName`/`ledGroupName`/`ledStateName` on edit so the saved value
still shows before the lazy load. RTK-Query equivalent already exists: `getItemOptions` +
`useLazyGetItemOptionsQuery` in `store/api/lookupsApi.ts` (dropdown_id 6).


Superseded for new work by the config-driven component: [[erp-nex-dropdown-component]].
