---
name: erp-name-id-lookup-empty-name
description: "name-id master-lookup returns empty `name`; real label is in the module-specific field"
metadata: 
  node_type: memory
  type: reference
  originSessionId: 964dfca5-af7a-4da9-93f7-0cf969f518e2
---

`/api/v1/master-lookups/name-id/all-accounts-and-masters?module=<m>` returns rows shaped like
`{ acc_group_id, acc_group_name, id, name: "" }` — the generic `name` field is an **empty string**,
and the real label lives in the module-specific field (`acc_group_name`, `brand_name`, `itg_name`,
`comp_name`, `state_name`, etc.).

When building dropdown options, the label key list must include the module-specific name key, e.g.
`getFirstDefinedValue(row, ["name", "label", "accGroupName", "acc_group_name"])`. With only
`["name","label"]` the label falls back to the uuid id (the select then shows the id, not the name).
`getFirstDefinedValue` skips `""`, so key order vs `name` doesn't matter — just include the real key.

Most master pages already do this (labelKeys list module key first). Endpoint `limit` is clamped 1–100,
defaults 20. Verify endpoints per [[erp-auth-endpoint-verification]]; also [[erp-menu-lookup-options]].
