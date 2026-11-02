---
name: gsp-masters-blocked
description: "GSP masters (menus 255/256) not built: 11_ tables are live but NO NestJS routes on them — notes 79 asks Prathap for R1–R9; mockups in share/gst; old id reservations are stale"
metadata:
  type: project
---

Asked 2026-10-02 "why we skipped front end of gsp master and configuration". The reason: it is blocked on the backend.

- The `11_gst_provider.sql` tables ARE deployed: 2 providers, 23 endpoints, 25 error maps, 0 credentials.
- Prisma models exist, but no service or controller uses them. The only routes are the OLD `/gsp-provider-masters`, `/gsp-company-services` (fixed.gsp_*, to be retired) and `/gst/search` (GSTIN lookup, live).

**Done 2026-10-02 (user said yes):**
- Mockups `share/gst/gst_provider_ui_mockup.png/.py` (menu 255: header + Services & endpoints tab + Error map tab; provider account = credential #1 on a right panel, secrets write-only).
- Mockup `gst_credential_ui_mockup.png/.py` (menu 256: list on vw_gst_credential + entry; GSTIN read-only/resolved; Verify F7 = one AUTH call; session panel).
- `share/gst/notes-79-gsp-config-routes.md`: routes R1–R9 plus the write-only secret contract (absent/"" = keep, `clear: []`, responses carry `hasX` + keyVersion only).

**Stale reservations:** plan-qt-gsp §0 ids are void. Grids 121/122/125, dd 58–60 and ui_table 43/44 are taken. Menus 255/256 are still free.

**Next:** when Prathap ships R1–R9, register the grids (server assigns ids), build both screens to the mockups ([[mockup-fidelity]]), then wire the GSTIN fetch buttons to /gst/search. Related: [[gsp-integration-schema]], [[plans-separate-file]].

**Menus written 2026-10-02 (user: "write new menus"):** `share/46_gst_menus.sql` adds GST Providers (8.30) and GST Credentials (8.40) under Configuration (60). Both are HIDDEN until the screens ship. The ids come from the sequence (267/268 on erp_dry), so the plan's 255/256 are void. Credentials carry the POST verb, which is Verify. Prathap runs the SQL; register the screens under the ids he reports.

**2026-10-02 evening: Prathap DONE.** Routes R1–R9 are live under /gst/providers, provider-services, provider-endpoints, provider-field-maps, provider-error-maps, provider-accounts and company-credentials (with verify/status/restore). The source is in nexerp-api-ro/src/modules/gst. Menus are **269 GST Providers / 270 GST Credentials** (hidden; NOT 267/268). Read the DTOs in modules/gst/dto before building the screens.

**Build 2026-10-02 (user: "yes build now"):**
- **VKPOS rights:** granted 269 (V/C/E/D/Export) and 270 (+Post) by a full-echo save. Diff: 85 rows to 87, only +269 and +270. The routes enforce rights (403 GST_RIGHT_VIEW).
- **Grids registered:**
  - 127 MAIN LIST - GST PROVIDERS (igpv_is_deleted)
  - 128 GST PROVIDER - SERVICES (igpv_id)
  - 129 GST PROVIDER - ENDPOINTS (igps_id)
  - 130 GST PROVIDER - ERROR MAP (igpv_id)
  - 131 MAIN LIST - GST CREDENTIALS. It reads the TABLE with non-secret columns only (igcc_is_deleted), because vw_gst_credential hides deleted rows and Restore needs them.
- **Dropdowns:** 61 BRANCHES BY COMPANY (ibr_comp_id, always sent) and 62 GST PROVIDERS.
- **Route facts:** providers/get returns services[] (with endpointCount) and accounts[] (has* flags) but no endpoints or error maps. Hence the grids. provider-endpoints/get returns fieldMaps[].

**SCREENS BUILT 2026-10-02** in `src/modules/admin/master/gst/`:
- **GstProviderList / GstProviderEntry:** a QDialog built to the mockup. The tabs use grids 128/129/130, accounts are cards from /get, and children are edited in popups.
- **GstPartForm:** NexCrudForm plus a GstPartSpec covering parent key, nullable, "(all)" = null, array/object line fields, secrets and clear[]. Shaping goes through a holder in the delegated constructor.
- **GstSecretField:** write-only. Undefined means keep.
- **GstFieldMapPanel:** lives inside the endpoint popup.
- **GstCredentialList / GstCredentialEntry:** Verify F7 is greyed for 60 s; Post is read from permissionsFor(270).umCanPost. /status shows in the session panel.
- Registered with `fm.reg<…>(269/270)`.

**Live test** `tests/gstmasters/run.sh`: 31 ok. It writes a ZT<hhmmss> provider and soft-deletes it, so ZT providers stay soft-deleted on the box, and a code can't be reused even when deleted.

**BLOCKER:** the server has NO `GST_CRED_KEY`, so every secret write is a 503 (notes 79 §8). The credential create (password required) and the secret round trip are untested; re-run with `GST_CRED_KEY_SET=1`. Verify was never pressed live, because it calls an outside portal.

**Shared fixes made alongside:**
- NexJsonModel shows a null value as blank, not 0.
- The bool pill is red only for "Inactive"; a plain "No" is neutral.

**Menus 269/270 are still HIDDEN** (menu_visiblity=false). The PATCH /menu-masters/visibility route is GLOBAL, so the user decides.
