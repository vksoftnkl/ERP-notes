# Notes 88 — No active provider, no GST call: the rule applies to EVERY service

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap
**Extends notes 87.** 87 asked for this for GSTIN search only. The user wants it for **all** GST calls.

## The rule (user, 2026-10-03)

> Without an active provider, don't allow any service.

Any call that leaves our server for a GSP or NIC must find **all** of these active and not deleted:

| Row | Must be active |
|---|---|
| `gst_provider` | yes, the master switch |
| `gst_provider_service` (service × environment) | yes |
| `gst_provider_endpoint` (the action being called) | yes |
| `gst_provider_account` (same provider + environment), when the call needs `{aspId}` / `{aspPassword}` / `{apiKey}` | yes |
| `gst_company_credential`, for the calls that act for a taxpayer | yes (already checked, I believe) |

If any of them fails: **refuse before calling out**, write no outbound request, answer **503** with
the first inactive row named, for example `GST provider CHARTERED is inactive`, or
`GSTIN_VERIFY · PRODUCTION service is inactive`. The Qt screens already show a 503 message as it is.

## What changes

1. **Verify (R8)**: the gst README says *"Inactive providers / services / endpoints / accounts are
   allowed — Verify is how a provider is tested before it is activated."* That exception goes.
   Verify refuses like everything else. To test a provider, the operator activates it, which is
   the user's own rule: enabling it means use it.
2. **GSTIN search (`/gst/search`)**: as notes 87 (rows instead of env, refuse when inactive).
3. **IRN / e-way bill / GSTR / ASP_ADMIN and anything built later**: the same check, in **one
   shared guard** that every caller goes through (e.g. inside the client service that builds the
   request from the endpoint row), so a new call can't forget it.
4. Leave a **live session** (`gst_auth_session`) alone when the provider is switched off, but
   don't use it: the next call refuses at the guard. Deleting it isn't needed.

## How we'll check it (live, CHARTERED is inactive right now)

1. Verify on the THE SCM SILK credential → 503 `GST provider CHARTERED is inactive`, and **no new
   `gst_api_log` row**.
2. `/gst/search?gstin=33ACXPL0457H1ZD` → the same 503, no log row.
3. Activate CHARTERED but leave EINVOICE · SANDBOX inactive → Verify names the service.
4. Activate provider + service + AUTH endpoint + SANDBOX account → Verify succeeds as it does today.

We'll update `tests/gstmasters` on our side once this is live. Today it checks that the Verify
button is enabled on a saved credential; it will also check the 503 text when the provider is off.
