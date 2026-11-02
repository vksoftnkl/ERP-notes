# Notes 83 — CHARTERED (TaxPro) sandbox sign-in: four seed errors, fixed live, please fix the seed

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## What happened

After notes 82 (thanks, the key works), we set up the TaxPro **SANDBOX** account and a credential
(GSTIN 34AACCC1596Q002, user TaxProEnvPON) and pressed Verify. It failed 3 times with
`The portal refused the sign-in (HTTP 405)`. `gst_api_log` shows the request went to
`http://gstsandbox.charteredinfo.com/eivital/v1.04/auth` and came back 405 with an empty body.

We probed TaxPro **without credentials** (so nothing counts toward NIC's 5-per-15-minutes limit):

| Request | Answer |
|---|---|
| `GET http://gstsandbox…/eivital/v1.04/auth` | **301** → `https://…` (the redirect is what ended in our 405) |
| `GET https://gstsandbox…/eivital/v1.04/auth` | 412 `{"status_cd":"0","error":{"error_cd":"GSP001","message":"Missing Parameter 'aspid'…"}}` |
| `GET https://gstsandbox…/eivital/dec/v1.04/auth` | 412, same JSON reply: the route exists |

## The four seed errors (11_gst_provider.sql and 99_seed_all.sql, CHARTERED block)

1. **Sandbox base URLs are `http://`.** TaxPro now answers only on https and redirects http.
   This affects all 5 CHARTERED SANDBOX service rows (EINVOICE, EWAYBILL, GSTIN_VERIFY, GSTR,
   ASP_ADMIN). The seed comment "Sandbox is plain http in their documentation" is out of date.
2. **The EINVOICE AUTH endpoint never sends the e-invoice password.** Its headers are
   `aspid / password(=ASP password) / Gstin / user_name`. TaxPro wants the taxpayer's
   e-invoice password in **`eInvPwd`**, which gsp_flow.md §127 already lists. The path was also
   the non-`/dec/` `eivital/v1.04/auth`, where we would have to RSA-encrypt the body ourselves.
   GENERATE_IRN already uses `/eicore/dec/v1.03`, so this settles the open "/dec/ decision"
   as **/dec/**.
3. **The AUTH endpoint has no RESPONSE field map.** `gst-auth.service.ts` refuses to keep a
   session without `auth_token` ("the AUTH endpoint maps no auth_token"), so even a correct
   sign-in would have failed.
4. **The success and error paths don't match TaxPro's replies.** The success check was
   `$.status_cd = 1`, which is TaxPro's error envelope. A successful `/dec/` reply is expected
   to be NIC-shaped (`Status` / `Data`), not yet confirmed. The error paths were `$.error_cd` /
   `$.message`, but the real error nests them under `error` (see the probe above).

## What we changed on the box today (through the R-routes, as VKPOS)

| Row | Now |
|---|---|
| 5 × CHARTERED SANDBOX services | `gps_base_url = https://gstsandbox.charteredinfo.com` |
| EINVOICE SANDBOX · AUTH | path `/eivital/dec/v1.04/auth`, method GET |
| | headers `aspid:{aspId}  password:{aspPassword}  Gstin:{gstin}  user_name:{loginId}  eInvPwd:{password}` |
| | redact `$.password  $.eInvPwd  $.Data` |
| | success path **NULL** (success = HTTP 2xx plus the required token below) |
| | error code / message `$.error.error_cd` / `$.error.message` |
| AUTH field map (RESPONSE) | `auth_token ← $.Data.AuthToken` (required) · `expires_on ← $.Data.TokenExpiry` |

## The ask

- Put the same values into the CHARTERED block of `11_gst_provider.sql` and `99_seed_all.sql`,
  so a fresh install signs in on the first try.
- PRODUCTION rows are already https. When production goes live, its EINVOICE AUTH endpoint will
  need the same `/dec/` path, `eInvPwd` header and field map. That row isn't seeded yet.
- Once a sandbox sign-in succeeds, we'll send you the reply shape from `gst_api_log`. If it does
  carry `Status: 1`, we'd set the success path to `$.Status = 1` and add
  `$.ErrorDetails[0].ErrorCode` handling for NIC-level refusals (those come back as HTTP 200).
  One endpoint can only hold one error path, so you may want a second one there.

Nothing is needed in NestJS code for the sign-in itself; everything above is configuration data.
