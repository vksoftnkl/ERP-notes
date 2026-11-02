---
name: erp-setting-reference-fields
description: App-settings UUID values that point at a master render a real picker; binding lives in lib/value-sources.ts, label resolves via master-lookups ?module=&id=
metadata:
  type: project
---

Application Settings draws controls by DATA, never by `asdKey` — with one exception added
2026-09-19: a UUID setting whose id belongs to a master (`sales.default_customer_id` →
`sales.customers`). `app_setting_def` has no column saying WHICH table a uuid points at, so
that one fact lives in `features/settings/app-settings/lib/value-sources.ts`
(`dropdownKey` + server `lookupModule`); every bound setting then gets the same
`NexDropdownSingle` picker. Delete the table the day the catalog grows `asd_ref_module`.

Key API fact: `/master-lookups/name-id/all-masters?module=<key>&id=<uuid>` returns that ONE
row (`[]` when nothing matches) — the only way to turn a stored id back into a name, because
a configured dropdown's SQL searches text columns and never matches a uuid. Client hook:
`useGetMasterNameByIdQuery` in `store/api/lookupsApi.ts`. `module` spellings are the server's
`LOOKUP_MODULE_KEYS`. Note the two name sources can differ in case for the same row
(lookup vs `cus_name`).

**Why:** the walk-in customer was a hand-typed 36-char uuid seeded by a migration.
**How to apply:** binding a new referencing setting = one line in `SETTING_VALUE_SOURCES`, no
control code. Related: [[erp-app-settings-contract]], [[erp-nex-dropdown-component]],
[[erp-configured-dropdowns]], [[erp-name-id-lookup-empty-name]].
