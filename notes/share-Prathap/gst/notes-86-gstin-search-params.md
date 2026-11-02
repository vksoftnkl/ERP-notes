# Notes 86 — GSTIN search: TaxPro says "Missing Mandatory Params" (GEN5001)

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## Where we are

Thanks for 85: the message and the `gst_api_log` row are exactly what we needed. Both live
searches today fail the same way, **whichever company we search as**:

| When | Request (logged, secrets redacted) | Reply |
|---|---|---|
| 10:06 | `…/commonapi/v1.1/search?aspid=***&password=***&Action=TP&Gstin=34AACCC1596Q002&SearchGstin=34AACCC1596Q002` | 400 `{"status_cd":"0","error":{"error_cd":"GEN5001","message":"Missing Mandatory Params"}}` |
| 10:10 | same, `Gstin=33ACXPL0457H1ZD` (CHANDRA SAW MILL) | same 400 GEN5001 |

So it isn't the source GSTIN (34AACCC1596Q002 is the sandbox's own test GSTIN). The ASP pair is
accepted: a wrong one would give 401 GSP020 / 412 GSP001A. The **request shape** is wrong.
We can't try variations from our side, because the real ASP pair exists only in your env and
encrypted in the DB.

## Please try, in this order (one call each; search isn't an NIC sign-in, so no 5-per-15 limit)

1. **TaxPro's documented form: `Gstin` is the GSTIN being searched, with no `SearchGstin`.**
   ```
   GET https://gstsandbox.charteredinfo.com/commonapi/v1.1/search?aspid=…&password=…&Action=TP&Gstin=34AACCC1596Q002
   ```
   That's how TaxPro's public-API help page shows "Search Taxpayer" (our recollection of
   gsthelp.charteredinfo.com, please check the page). `buildUrl()` sends the company's GSTIN as
   `Gstin` and the searched one as `SearchGstin`. That pairing is the GST-portal (GSP direct) style.
   Our own seeded PRODUCTION row uses it too (`?action=TP&gstin={gstin}&searchgstin={searchGstin}`),
   so that row needs the same check.
2. If 1 still gives GEN5001: the same URL on **production** `https://gstapi.charteredinfo.com` with
   the production pair, if you have it. The sandbox's public-API mirror may not serve TP search at
   all, and GEN5001 is the generic answer you'd get then. If that's the case, ask Chartered support,
   quoting the logged request above.

## Fallback that already works on the sandbox

The e-invoice sign-in now verifies, and the IRP has its own **Get GSTIN details** call on the same
session. TaxPro's `/dec/` form is:
```
GET https://gstsandbox.charteredinfo.com/eivital/dec/v1.03/Master/gstin/{searchGstin}
headers: aspid, password, Gstin (ours), user_name, AuthToken   (the AUTH row's set + {authToken})
```
It returns legal / trade name, address, state and status: everything the fetch button fills.
If TP search stays blocked on the sandbox, `/gst/search` could fall back to this through the
EINVOICE service's session. This would also be the first real use of the endpoint-row engine
outside Verify.

No client change in either case; the Qt buttons keep calling `/gst/search`.
