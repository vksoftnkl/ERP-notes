---
name: erp-company-branch-qt-parity-plan
description: Plan (2026-10-02) to bring the React company/branch masters up to the Qt forms — written to ERP client plan/company-branch-qt-parity.md; nothing built yet; 7 decisions pending
metadata:
  node_type: memory
  type: project
  originSessionId: e0290841-c622-4b94-bead-23b4bafdf3d4
  modified: 2026-10-02T05:13:37.198Z
---

On 2026-10-02 compared Qt `src/modules/admin/master/{company,branch}` (NexERP 09f8896) with React
`features/masters/settings/{companies,branches}` (dev 0ba5c67) and wrote
`/home/vk/Dev/erp/ERP client/plan/company-branch-qt-parity.md` (5 phases, decisions D1–D7). Not started.

Facts worth keeping:
- Grids 52/56 and dropdowns 21/22/25/26/27 are ALREADY shared by both clients (grid-registry.ts,
  dropdown-registry.ts) — don't re-plan them.
- Biggest React defects found: `compTdsApplicable` never sent (one "TCS/TDS" box), `compAatoClass` and
  `compDcPurposes` absent, `compRegionState` overwritten with `compState` on every save, GSTIN "required"
  even for UNREGISTERED, branch `brGstRegType` is free text (server IsIn → 400), godown dropdown 26 gets no
  `ibranch_id` param, bank selects list every ledger instead of dropdown 25, no Restore (Delete sends DELETE
  on a deleted row → 404).
- Qt is behind in one place: its GSTIN fetch button is a TODO stub; React has a lookup but via the Next
  route `app/api/gst/search/route.ts` (hardcoded GSP creds) — the plan moves it to server `GET /gst/search`.
- Dynamic form supports `checkbox-group`, `disabledWhen(values)`, `file`+`previewImageValueKey`;
  CrudMasterPage has `onDeleteAction`, `isRowEditDisabled`, `deleteConfirmMessage` but the "Delete" caption
  is hardcoded (~line 4407/4823).
- DB defaults: comp_sms_applicable false, br_allow_negative_stock true, comp_aato_class 'LE_5CR',
  comp_dc_purposes {SUPPLY,APPROVAL,JOB_WORK,OTHER}; Qt seeds SMS=true — disagreement is decision D2.

**How to apply:** when asked to build it, start at Phase 1 (shared gst-registration / gstin-lookup /
fiscal-year helpers) and ask for D1–D7 answers first. See [[nexerp-qt-client-source]],
[[erp-notes-72-company-branch]].
