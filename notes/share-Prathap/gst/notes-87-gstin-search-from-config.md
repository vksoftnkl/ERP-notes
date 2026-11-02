# Notes 87 — GSTIN search should obey the GST Provider screen (Active switch), not the env

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## The problem

Thanks for 86: search works (`deprecatedgstapi.charteredinfo.com`, real taxpayer data). But the
user **disabled the CHARTERED provider and the search still returned data**. `/gst/search` reads only
`GST_LOOKUP_ENDPOINT / _ASP_ID / _ASP_PASSWORD` from the environment, so the GST Provider screen has
no effect on it. The user's rule is simple: **if the provider is enabled, use it; if it's disabled,
don't.** Today the only way to turn search off or move it to another host is a server edit.

## The ask

Make `gstin-lookup.service.ts` resolve everything from the GSP rows, the way Verify does.

1. **Pick the service row:** `gst_provider_service` where `gps_service = 'GSTIN_VERIFY'`, active,
   not deleted, under an **active** provider. If more than one qualifies, PRODUCTION wins over
   SANDBOX, then the provider with the lower priority / earlier row (your call; just make it
   deterministic).
2. **Endpoint:** that service's active `VERIFY_GSTIN` row (path + query template). Placeholders:
   `{aspId}` `{aspPassword}` `{gstin}` (the logged-in company's) `{searchGstin}`.
3. **Account:** the active `gst_provider_account` of the same provider + environment (service
   GSTIN_VERIFY or NULL), whose `gpa_client_id_enc` / `gpa_client_secret_enc` give `{aspId}` /
   `{aspPassword}`. This is the same rule as Verify.
4. **Nothing qualifies** (provider, service, endpoint or account inactive / missing): answer 503
   `GSTIN search is switched off: no active GSTIN_VERIFY provider`, **without calling out**. The Qt
   fetch buttons already show a 503 message and let the operator type the details by hand.
5. Keep the env variables only as a fallback while you migrate, or drop them. Once the rows work, we'd
   prefer them dropped, so there's one switch.

Keep 85's behaviour: the nested `error.message` / `error_cd` on the screen, and one `gst_api_log` row
per call.

## Rows to set on the box (live today)

| Row | Today | Needs |
|---|---|---|
| CHARTERED provider | **inactive** (the user just switched it off) | user's switch |
| GSTIN_VERIFY · PRODUCTION | base `https://gstapi.charteredinfo.com`, inactive; endpoint `VERIFY_GSTIN GET /commonapi/v1.1/search?action=TP&gstin={gstin}&searchgstin={searchGstin}`, inactive | the base URL that actually works (`https://deprecatedgstapi.charteredinfo.com`, or whatever Chartered says replaces it); the query should name `aspid={aspId}&password={aspPassword}` and match the parameter case that works today (`Action=TP&Gstin=…&SearchGstin=…`) |
| GSTIN_VERIFY · SANDBOX | base `https://gstsandbox.charteredinfo.com`, inactive, **no endpoint row** | either a VERIFY_GSTIN row, or leave it without one (the sandbox TP search gave GEN5001, and sandbox GSTINs aren't in the live register) |
| Provider account · PRODUCTION | placeholder, **no secrets** | the ASP pair that the env holds today |
| Provider account · SANDBOX | both secrets set (sandbox aspid), inactive | as is |

Please say which ASP pair the env uses today, sandbox or production. Search hits the live register,
so it must be whichever pair Chartered bills for live calls. That pair belongs on the matching account.

## How we'll check it

1. CHARTERED inactive → `/gst/search` answers 503 "switched off", and no `gst_api_log` row is written.
2. CHARTERED + GSTIN_VERIFY PRODUCTION + its endpoint + account active → a real GSTIN returns data,
   and `gst_api_log` shows the row's URL.
3. Change the base URL on the screen → the next search uses it with no restart (or after the 1 s
   grid cache).

No client change needed; the fetch buttons keep calling `/gst/search`.
