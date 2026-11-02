---
name: gsp-integration-schema
description: "GST Suvidha Provider config layer — 11_gst_provider.sql, config-driven multi-provider, response map writes into existing gde_/gdw_ columns"
metadata: 
  node_type: memory
  type: project
  originSessionId: b131386e-5b90-460a-afad-91505c0b1205
  modified: 2026-08-17T11:40:20.022Z
---

Built 2026-08-17. `/home/vk/Shared/Prathap/schema/11_gst_provider.sql` replaces the
first-draft `fixed.gsp_provider_master` / `fixed.gsp_company_service` (single provider,
single `gsp_route`, credentials on the provider row, no FK). Ran clean on a throwaway
PG 18 cluster; every constraint tested by violating it.

**All nine tables live in `public`** (moved there 2026-08-17 at the user's suggestion —
the first cut had config in `fixed`). Two reasons: GST belongs to no single module, the
criterion 10_app_settings.sql states for itself; and `fn_create_year_partitions` scans
only public/sales/accounts, so a partitioned table in `fixed` is silently skipped by the
April 1st ritual and fails at the first insert of the new year. `fixed.state_codes` is
still referenced read-only by `gcc_state_code`.

The chain:
`public.gst_provider` (`gpv_`) + `gst_provider_account` (`gpa_`) → `gst_provider_service` (`gps_`, provider × service ×
environment: base_url, auth scheme, payload encryption) → `gst_provider_endpoint`
(`gpe_`, one row per action + envelope descriptors + `gpe_redact_paths`) →
`gst_provider_field_map` (`gfm_`); plus `gst_provider_error_map` (`gem_`),
`gst_company_credential` (`gcc_`), `gst_auth_session` (`gas_`),
`public.gst_api_log` (`gal_`, partitioned).

Locked decisions:

- **Adding provider #2 is INSERT-only.** Routes, reply shapes and error codes are rows.
- **RESPONSE mapping only.** `gfm_target_column` names the real destination
  (`gde_irn`, `gde_ack_no`, `gdw_no`…) in the ALREADY-EXISTING
  `accounts.acc_voucher_doc_einvoice` / `_ewaybill` — this layer never restates an IRN.
  REQUEST rows are scoped to headers/wrapper substitution; the invoice body is built in
  code from one canonical DTO (~140 conditional, arithmetic-checked government fields).
- **TWO credential layers, at different grains** (settled with the user 2026-08-17):
  CREDENTIAL #1, the GSP's own account (client id / secret / api key identifying this ERP
  installation to the gateway), lives on **`public.gst_provider_account`** at
  provider × ENVIRONMENT grain — a wrapper GSP like Chartered Information Systems sells
  e-invoice + e-way + returns on ONE account, so per-service would copy the secret across
  four rows. `gpa_service` is nullable: NULL = covers every service, set = override for the
  rare provider that issues separate accounts; resolution is most-specific-wins
  (`ORDER BY gpa_service NULLS LAST LIMIT 1`), the same idiom as `gem_service`. A
  direct-to-NIC provider has NO account row — there is no middleman to authenticate to.
  CREDENTIAL #2, the taxpayer's portal user, is `gst_company_credential` — and
  **`gcc_service` is likewise NULLABLE**: NULL = one login covers every service the GSP
  sells (the gateway case), set = one login per service (direct NIC, where e-invoice and
  e-way bill are separate portal registrations with non-interchangeable tokens). Since
  `gst_auth_session` hangs off the credential, this column decides how many live tokens
  exist and how many calls to `/auth` get made — 4 rows means 4 logins and 4 auth calls
  where 1 would do. Both unique indexes use `NULLS NOT DISTINCT` (branch AND service are
  legitimately null); the COALESCE-sentinel idiom was dropped as it can't express a
  nullable varchar cleanly.
- **No GSTIN column on the credential.** `public.companys.comp_gstin_no` is already UNIQUE,
  so a company row IS a registration; branches with their own registration carry it on
  `branch_master`. Resolved as `COALESCE(branch GSTIN, comp_gstin_no)`. Grain is therefore
  company + branch + service + environment + priority; `ux_gcc_primary` allows one
  priority-1 row (COALESCE sentinel for the NULL branch, the `ux_tnd_default` idiom).
- **NIC 2150 / EWB 312 are `gem_treat_as = 'SUCCESS'`** with `gem_extract_path` harvesting
  the already-issued IRN out of the *error* payload. Without this, failover is a
  duplicate-document generator.
- **Auth session grain is PER CREDENTIAL, not per company** — cache key is `gcc_id`.
  `ux_gas_live` allows one live session per credential row, so a company can hold several
  simultaneous tokens (per service / environment / failover) or exactly one, depending on
  how `gcc_service` is set.
- **Session is separate from credential** and holds a LEASE (`gas_lock_by/_upto`) plus
  `gas_token_version` — four tills notice the same expiry in the same second, and NIC
  blocks accounts that stampede `/auth`. `fillfactor=70`, deliberate deviation.
- Secrets are app-encrypted ciphertext (`_enc` + `gcc_key_version`); no pgcrypto anywhere
  in this schema.
- Prefix is `gpv_` not `gsp_` — the old table being dropped owns `gsp_`, and `gsp_`/`gps_`
  side by side are a transposition trap.

**NOT built, deliberately — open with the user:** a `gst_api_job` queue table (today only
`gde_next_attempt_on` / `gdw_next_attempt_on` were added, no lease, no worker), the GSTR
filing tables (`grp_`/`gru_`/`grd_`), and a `gst_taxpayer` GSTIN cache.

**Client screens pending user-assigned ids** (menu + grid; highest in use: menu 246,
grid 85). Two masters under `src/modules/admin/master/gst_provider/` and
`gst_credential/`, 4 files each, `NexCrudMasterPage` + `NexCrudForm`. They fill three
existing stubs: `btnQuickEInvoice`/`btnQuickEWay` (no handler), the GSTIN fetch buttons'
`// TODO: call the GSTIN lookup API`, and the `IRN`/`EINVOICE` badge tone.

**`public.vw_gst_credential`** resolves the GSTIN once —
`COALESCE(br_gstin_no, comp_gstin_no)` — and is the READ BOUNDARY: it selects no `_enc`
column at all, so the list grid and any read-only role are granted on the view and revoked
on the table (verified: the role reads the view, gets `permission denied` on the table).

**`schema/ref_existing_masters.sql`** (added 2026-08-17) captures `public.companys` and
`public.branch_master` verbatim — pre-existing, app-owned, NOT part of the 00→11 run chain,
all `IF NOT EXISTS`. They FK-anchor the whole chain but were invisible without a dump.
`companys` also FKs `public.app_theme_master`, so a throwaway validation database needs a
`thm_id integer PRIMARY KEY` stub first. Note `br_gstin_no` is nullable and NOT unique
(several outlets legitimately share one state registration), while `comp_gstin_no` IS unique.

**Chartered Information Systems (TaxPro GSP)** — real config seeded 2026-08-17 from their
public docs at gsthelp.charteredinfo.com, INACTIVE pending aspid + ASP password:
auth `…/eivital/v1.04/auth` (GET), IRN `…/eicore/dec/v1.03/Invoice?QrCodeSize=250` (POST);
hosts `gstsandbox.charteredinfo.com` (http) / `einvapi.charteredinfo.com` (https) with
published failover clusters (`einvapimum1`, `einvapidel2`) in `gps_fallback_urls`; headers
`Gstin, user_name, AuthToken, aspid, password`; envelope `status_cd`='1' / `error_cd` /
`message`; token 360 min. **NIC blocks a GSTIN after 5 auth calls in 15 minutes** — the
hard justification for the `gas_lock_*` lease. The `/dec/` path = Chartered does the NIC
encryption, hence `gps_payload_encryption = 'NONE'` with `NIC_SEK` auth.
Their SDK splits credentials the same way we did (`eInvoiceAPISetting` = installation,
`eInvoiceAPILoginDetails` = taxpayer). Response model `RespPlGenIRN` is FLAT — no Data
wrapper. e-invoice dates are NIC-format but e-way dates are 12-hour
`dd/MM/yyyy hh:mm:ss a`, which is why `DATETIME_MASK` + `gfm_format_mask` exist.
Their e-invoice API is a **byte-for-byte passthrough of NIC's IRP** (their words: same
headers, body and query, only the host differs, ClientID injected server-side) — so any
missing path comes from NIC's IRP v1.03 spec, never invented. **Three hosts**:
`einvapi` (e-invoice/EWB), `gstapi` (search, returns, /aspapi), `gstsandbox` (all, http).
Multi-IRP: `irp` header = NIC1|NIC2, same URL; **the first character of AckNo says which
portal issued the IRN** (1=NIC1, 2=NIC2) — that is how `gde_irp_code` is derived.
E-way bill is NIC v1.03 passthrough driven by `?action=` (GENEWAYBILL, VEHEWB, CANEWB,
REJEWB, UPDATETRANSPORTER, EXTENDVALIDITY, GENCEWB, REGENTRIPSHEET, MULTIVEHMOVINT/ADD/UPD,
CLSEWB) — 12 verified values, path still `/CONFIRM-PATH`.
**Still missing:** the EWB path (one value unlocks 14 rows), 9 e-invoice paths, NIC numeric
error codes (only their GSP001–GSP505 gateway table is published, and it IS seeded).

See also [[db-schema-share-location]], [[sales-backend-schema-design]].

**2026-09-20:** `share/gst/` now holds the full plan set (README, gsp_flow, plan-backend-gsp, plan-qt-gsp). gsp_flow §4a is the list of what 3.0's `feinvoice_system` / `geneinvoice.cpp` / `gsteinvoiceapi.cpp` knew and where each fact landed (NIC pre-checks, CRN/URP/Disp-Ship-by-transaction-type, EwbDtls in the IRN call, Part-B/extension fields, get-IRN-by-doc, TaxPro GetKey asp-secret, register F-keys + JSON preview). Generation happens FROM the document screen first; registers 152/153 are for cancel/extend/bulk/log. `gdr_igst_on_intra` added by 30 §5.
