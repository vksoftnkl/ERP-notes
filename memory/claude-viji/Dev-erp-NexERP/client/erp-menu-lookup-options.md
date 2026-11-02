---
name: erp-menu-lookup-options
description: Building menu (menu_id) dropdowns — endpoint returns a nested tree that must be flattened
metadata: 
  node_type: memory
  type: reference
  originSessionId: f7c9a95c-87c2-493b-bdfc-2b85fa8b6f68
---

To populate a menu/screen dropdown (FK to `menu_master.menu_id`, e.g. widget-master `sectionMenuId`):

- Endpoint: `GET /menu-masters/get` (client const `MENU_ENDPOINT`). Response is `{ data: [...] }` of **camelCase** nodes `{ menuId, menuName, children: [...] }`, permission-filtered for the user. It is a **nested tree** — `includeChildren=false` still nests, and the top level only has ~9 roots, so deep leaf screens (e.g. menuId 36 = "Item Brand Master") are NOT at the top.
- Flatten recursively (walk `children`) before mapping to options; `buildLookupOptions`/[[erp-grid-columns-config]] helpers do NOT recurse into children.
- Convention: wrap the loader so `useMasterOptions({ definition, load })` gets a flat `{ data: [...] }`, with `idKeys:["menuId","menu_id"]`, `labelKeys:["menuName","menu_name"]`. Render with a `type:"select", searchable:true` field (stores menuId, shows menuName). See `features/masters/widget-master/page.tsx` (`flattenMenuRecords`, `MENU_LOOKUP_DEFINITION`) and the `features/masters/tax/page.tsx` ledger-dropdown pattern.
- Top-level menu names carry "&1/&2…" mnemonic prefixes (e.g. "&1 Sales"); leaf names are clean.

Related: [[erp-auth-endpoint-verification]], [[erp-workspace-layout]].
