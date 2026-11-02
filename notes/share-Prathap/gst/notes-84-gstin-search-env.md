# Notes 84 — GSTIN search still 503: the three GST_LOOKUP_* settings (notes 79 §7) are not set

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## Where we are

Thanks for 82 and 83: the CHARTERED SANDBOX sign-in now **verifies** (GSTIN 34AACCC1596Q002,
user TaxProEnvPON). But the GSTIN fetch button (Supplier, Customer, Company, Branch, Ledger) still
answers live:

```
GET /gst/search?gstin=34AACCC1596Q002
→ {"success":false,"message":"GSTIN search is not configured",
   "errors":[{"field":"gstin","message":"Set GST_LOOKUP_ENDPOINT, GST_LOOKUP_ASP_ID,
              GST_LOOKUP_ASP_PASSWORD in the server environment"}]}
```

`gstin-lookup.service.ts` reads only these environment variables. It doesn't use the GSP
configuration (gst_provider / provider account), so getting Verify to work didn't fix the search.

## The ask (quick, unblocks the button today)

Add to the API environment and restart:

```
GST_LOOKUP_ENDPOINT=https://gstsandbox.charteredinfo.com/commonapi/v1.1/search
GST_LOOKUP_ASP_ID=<the same sandbox aspid saved on the CHARTERED SANDBOX account>
GST_LOOKUP_ASP_PASSWORD=<the same sandbox ASP password>
```

Use **https**: http answers 301, the same trap as notes 83. Checked today without credentials:
`https://gstsandbox.charteredinfo.com/commonapi/v1.1/search` answers TaxPro's own
`GSP001 Missing Parameter 'aspid'`, so the path is right.

The search sends `Gstin` = the logged-in company's GSTIN (`compGstinNo`). On the sandbox, that means
logging in to the company that carries the test GSTIN (THE SCM SILK, 34AACCC1596Q002).

## Worth doing later (your call)

Make `/gst/search` read the **CHARTERED GSTIN_VERIFY service + provider account** instead of the
environment, the same way Verify does. Then the ASP pair would live in one place (encrypted), and
switching sandbox → production would be a credential change, not a server edit. A
`VERIFY_GSTIN` endpoint row already exists for PRODUCTION
(`GET /commonapi/v1.1/search?action=TP&gstin={gstin}&searchgstin={searchGstin}`). There is no
SANDBOX one yet.

No client change is needed either way; the Qt buttons already call `/gst/search`.
