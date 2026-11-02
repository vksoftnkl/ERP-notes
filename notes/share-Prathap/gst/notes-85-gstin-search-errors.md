# Notes 85 — GSTIN search: show TaxPro's real error message, and log the call

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## Where we are

Thanks for 84: the GST_LOOKUP_* settings are in, and the search now gets past TaxPro's ASP check.
But every live search fails, and we can't tell why:

```
GET /gst/search?gstin=34AACCC1596Q002          (as VKPOS → company CHANDRA SAW MILL, 33ACXPL0457H1ZD)
→ {"success":false,"message":"GST service error",
   "errors":[{"field":"gstin","message":"The GST service answered HTTP 400"}]}
```

The first call after your restart answered HTTP 504; since then it's 400 every time (about 0.3 s).

We probed TaxPro with a **dummy** aspid. GSP-level refusals come back as 412 / 401 with the message
**nested**:

```
{"status_cd":"0","error":{"error_cd":"GSP020","message":"Invalid AspID, no matching ASP found."}}  [401]
{"status_cd":"0","error":{"error_cd":"GSP001A","message":"Error: Invalid Length of 'aspid' …"}}    [412]
```

So a 400 means our aspid / password were accepted and the GST system behind TaxPro refused the
search itself. Our guess is the source `Gstin`: it's the logged-in company's GSTIN, and the sandbox
only knows 34AACCC1596Q002. We can't confirm that, because the reply never reaches us.

## The ask

1. **Read the nested message** in `gstin-lookup.service.ts` → `messageOf()`. Today it checks
   `message | error | detail | status_desc | statusDesc` as strings only. TaxPro's `error` is an
   object, so it falls back to "answered HTTP 400". Please also read `error.message`, and include
   `error.error_cd` as `[GSP020] Invalid AspID…`. NIC-shaped replies may carry
   `ErrorDetails[0].ErrorMessage` in the same way.
2. **Log each search to `gst_api_log`**, as Verify does (service `GSTIN_VERIFY`, action
   `VERIFY_GSTIN`, with `aspid` / `password` redacted from the URL). Today a failed search leaves
   no trace on the box. With a log row, we can read the reply ourselves instead of guessing.

Item 1 alone is enough to unblock us. Item 2 is what we'd want before production.

## On our side

We're re-testing in the app while logged in to **THE SCM SILK** (34AACCC1596Q002). If that works,
the source-GSTIN guess is confirmed. Even then, production will search as each company's own GSTIN,
so item 1 is still needed for any refusal there.
