---
name: gstin-search-runs-on-gsp-rows
description: /gst/search (notes 87, 2026-10-03) reads ONLY the GST Provider rows — active provider + GSTIN_VERIFY service + VERIFY_GSTIN endpoint + account; GST_LOOKUP_* env is gone; CHARTERED live host is deprecatedgstapi
metadata:
  type: project
---

`GstinLookupService` (settings/gstinLookup, imports GstModule) resolves everything per request from the
GSP rows, the way Verify does — there is ONE switch, the GST Provider screen:
- service `gst_provider_service` GSTIN_VERIFY, active, under an active provider; PRODUCTION before SANDBOX,
  then gpv_code, then oldest — first whose whole chain is on: active VERIFY_GSTIN endpoint + active account
  (`resolveProviderAccount`, GSTIN_VERIFY else all-services). Nothing → 503 "GSTIN search is switched off",
  no call, no log row. Placeholders: {gstin} (company's own), {searchGstin}, {aspId}/{aspPassword}
  (= account clientId/clientSecret), {apiKey}. Endpoint success/error/root/redact paths + timeout honoured.
- Every call that goes out → one gst_api_log row (GSTIN_VERIFY / VERIFY_GSTIN, secrets '***'); refusals
  shown as `[code] message` (row error paths, else TaxPro `error.*`, else NIC ErrorDetails[0]).
- `GST_LOOKUP_ENDPOINT/_ASP_ID/_ASP_PASSWORD/_SOURCE_GSTIN` are no longer read; removed from the dev `.env`.

**Live CHARTERED facts (probed 2026-10-03):** sandbox `commonapi` answers GEN5001 to EVERY TP search (first
call after idle: 504 GSP032); `gstapi.charteredinfo.com` wants an `asp-secret` header (GSP002) — unknown
value; `deprecatedgstapi.charteredinfo.com` serves TP search with aspid+password in the query (both
`Gstin`-only and `Gstin`+`SearchGstin` shapes) and only knows REAL GSTINs (sandbox 34AACCC1596Q002 →
SWEB_9035). Box rows: PRODUCTION GSTIN_VERIFY → deprecatedgstapi, endpoint active, PRODUCTION account
active holding the SAME ASP pair as the SANDBOX account (aspid 1704563322); provider on/off is the user's.
Seed `Gsp_Providers.sql` matches. Other servers need the account pair entered (never seeded).

**How to apply:** to test search without flipping the user's switch, run the service on a transactional
prisma proxy in a rolled-back tx that sets gpv_is_active (recipe used for notes 87). The old React route
(`ERP client/app/api/gst/search/route.ts`) hard-codes the ASP pair — never echo that file unredacted.
Related: [[gst-config-module]], [[dev-box-runs-two-apis]].
