---
name: gst-cred-key-missing
description: DONE 2026-10-03 — Prathap set GST_CRED_KEY (notes 82); tests/gstmasters 41/41 with GST_CRED_KEY_SET=1; TaxPro sandbox field mapping
metadata:
  type: project
---

2026-10-03: saving the CHARTERED SANDBOX provider account answered 503 "GST credential key unavailable". The API has no `GST_CRED_KEY` in its env, and the guard in gst-crypto.service.ts refuses any secret write or decrypt without it. That blocks provider-account secrets, credential passwords and Verify (R8). It is not a client bug.

Asked Prathap in `share/gst/notes-82-gst-cred-key.md`: set `GST_CRED_KEY` (openssl rand -hex 32) plus `GST_CRED_KEY_VERSION=1`, back the key up, restart the API. GST_LOOKUP_* from notes 79 §7 can go in at the same restart.

TaxPro sandbox field mapping (told to the user): provider account Client ID = aspid, Client secret = ASP password ({aspId}/{aspPassword}). Credential: User name / Password = the e-invoice login, Client ID/secret BLANK (they would override the account pair), App key and public key ref blank (payload encryption NONE). The GSTIN comes from the company/branch. Seeded SANDBOX base URL is gstsandbox.charteredinfo.com, but Chartered gave einvapi.charteredinfo.com.

**DONE 2026-10-03:** notes 82 applied. `GST_CRED_KEY_SET=1 bash tests/gstmasters/run.sh` passes 41/41 (fixed: SANDBOX account pick, dangling f->close() crash, CHARTERED-active check now compares with the server). The user saved the CHARTERED SANDBOX account (ref 1704563322, both secrets) and activated CHARTERED; the credential and Verify come next.

**How to apply:** a GST 503 now means the key is gone or changed on the server, not a client bug. A crashed gstmasters run leaves ZT rows behind; delete them through the API. Related: [[gsp-masters-blocked]], [[gstin-fetch-buttons]].

**Sandbox Verify 405 (2026-10-03), notes 83:** the seeded CHARTERED rows were wrong. Fixed LIVE through the API: 5 SANDBOX base URLs http→https (http answers 301 → 405); EINVOICE SANDBOX AUTH now `/eivital/dec/v1.04/auth` with the `eInvPwd:{password}` header; success path NULL; error paths `$.error.error_cd` / `$.error.message`; RESPONSE field map auth_token←$.Data.AuthToken (req), expires_on←$.Data.TokenExpiry. Prathap applied notes 83 at 09:54 in HIS seed, nexerp-api-ro/prisma/seed/Gsp_Providers.sql (the share copies 11_/99_ are NOT updated; his seed is the one that matters). Not yet re-verified. The success-reply shape is unconfirmed (probably NIC Status/Data); read gst_api_log after the next Verify. NIC blocks a GSTIN after 5 sign-ins in 15 min, so probe TaxPro WITHOUT credentials. Scripts are in the session scratchpad only.

**2026-10-03 ~10:10: sandbox Verify SUCCEEDED** with the notes 83 config. GSTIN search (`/gst/search`) is separate: it reads env GST_LOOKUP_ENDPOINT / _ASP_ID / _ASP_PASSWORD only, not the GSP tables. Still unset, so notes 84 re-asks (79 §7) with the https sandbox URL and suggests reading the GSTIN_VERIFY config later.

**Notes 84 applied (2026-10-03):** the env is set, and search passes TaxPro's ASP check, but it answers HTTP 400 with the reason hidden: messageOf() ignores the nested `error.message`, and search writes no gst_api_log row. Likely cause: the source Gstin is the login company's (VKPOS → CHANDRA SAW MILL 33ACXPL0457H1ZD), while the sandbox knows only 34AACCC1596Q002 (THE SCM SILK, comp 019cc7fc-…). TaxPro's GSP errors are 412/401 with a nested message. Notes 85 asks for the nested message plus logging. The user is re-testing while logged into SCM SILK.

**Notes 85 applied:** search errors now show and are logged (gst_api_log GSTIN_VERIFY / VERIFY_GSTIN). Both live searches answer 400 `[GEN5001] Missing Mandatory Params`, even searching as 34AACCC1596Q002, so the cause is the request shape, not the source GSTIN. Notes 86: try `Action=TP&Gstin=<searched>` with no SearchGstin (TaxPro doc form; buildUrl uses the GSP-direct Gstin+SearchGstin pairing), then production; fallback = IRP `/eivital/dec/v1.03/Master/gstin/{gstin}` on the verified e-invoice session. We cannot test with the real ASP pair ourselves.

**Search works (2026-10-03 10:13)**, on deprecatedgstapi.charteredinfo.com (the LIVE register; sandbox GSTINs answer SWEB_9035). It reads env only, ignoring provider Active flags and company credentials. CORRECTION: the user WANTS the provider Active switch to control search ("if I enable provider means use it"; disabling it and still getting data surprised them). Fixing that needs a server change: /gst/search reads the CHARTERED GSTIN_VERIFY service and account rows instead of env. Asked in notes 87 (2026-10-03): resolve search from the active GSTIN_VERIFY service/endpoint/account, 503 "switched off" when none, drop the env.

**User rule 2026-10-03: no active provider → no GST call of ANY kind**, Verify included, which currently allows inactive rows on purpose. Notes 88 (extends 87) asks for one shared guard: provider + service + endpoint + account (+ credential) must all be active, otherwise 503 naming the first inactive row, with no outbound call and no log row. When live, update tests/gstmasters to expect that 503 with the provider off.

**Notes 87+88 LIVE (verified 2026-10-03):** with CHARTERED off, /gst/search and Verify both answer 503 `GST_SWITCHED_OFF` ("GST provider CHARTERED is inactive") and write no gst_api_log row. Search now reads the GSTIN_VERIFY PRODUCTION row (deprecatedgstapi, Action=TP&Gstin&SearchGstin) plus the PRODUCTION account. Client: the gstin_lookup fetch button and the credential Verify strip show errors[0].message for that code ("Switched off: …"). tests/gstin has 3 new mock checks; tests/gstmasters 42/42 (Verify on the inactive ZT provider → switched off).
