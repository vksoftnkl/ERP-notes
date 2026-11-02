---
name: erp-gst-config-screens
description: "GST Providers (menu 269) + GST Credentials (menu 270) React port of the Qt gst/ screens (2026-10-07) — where it lives, the grid-130 partial-row rule, the lookupKeys.code trap, canPost, and what still blocks live"
metadata:
  node_type: memory
  type: project
  originSessionId: 3129db5e-977d-4318-b9dc-6a012875f1e4
  modified: 2026-10-07T08:25:57.609Z
---

Built 2026-10-07 from Qt `NexERP/src/modules/admin/master/gst/*` and the
share's `Prathap/gst/*_ui_mockup.png` + notes 79/88. Code: `features/settings/gst/`
(domain/ = pure form specs + body shaping, vitest), `store/api/gstConfigApi.ts`,
routes `/settings/gst-providers` and `/settings/gst-credentials` (bound under
Settings → Configuration; menu rows 269/270 exist and are granted).

Non-obvious decisions:
- **Grid 130 (error map) carries only part of the row and there is no /get.**
  An edit from the grid row leaves severity / extract path / canonical field /
  retry-after OFF the body ("(as stored)") — the Qt form sent combo defaults and
  overwrote them on every edit. `fillPartState(..., {partial:true})`.
- New-row defaults follow the TABLE defaults (method POST, severity ERROR, token
  360/15), not the Qt combos' first item (GET/INFO).
- Credential list: `lookupKeys.code` must be `gpv_code` — the shell gives the first
  `*_code` column the row's code value, so `gcc_login_id` there put the login under
  Provider. See [[erp-crud-master-grid-column-heuristics]].
- Lists need `useConfiguredGridColumnsOnly` + `listResponseStyleArrayKey=""`, or
  only S.No renders.
- `ErpMenuPermissionFlags.canPost` (optional) now parsed from usermenu; Verify uses
  `canPost !== false`.
- Stacked dialogs (provider → endpoint → field map) decide Escape per event from the
  DOM (`[data-gst-dialog]` last = topmost) + `defaultPrevented`; see
  [[erp-nested-dialog-escape-race]].
- Old `/master/gsp-*` screens (fixed.gsp_*) are still in the menu; notes 79 says
  retire them once these work — not done, user's call.

**Live:** grids 127–131 and dropdowns 61/62 were missing on live as of 2026-10-03
([[erp-live-config-drift]]); sync them before shipping or the lists come back empty.
Server needs `GST_CRED_KEY` or every secret save 503s.
