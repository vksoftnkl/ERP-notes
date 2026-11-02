# GSP layer — the endpoint contract (HANDOVER for Prathap)

**2026-09-20.** The wire for `plan-backend-gsp.md`; the sequence is
`gsp_flow.md`. Prerequisite: `schema/11_gst_provider.sql` deployed (line
1361's `fn_create_year_partitions` call replaced by 29's
`ensure_sales_doc_partitions` or dropped), TaxPro SANDBOX rows active,
credentials inserted ENCRYPTED by your script from the environment (never
from a file in the share).

Every document route takes EITHER `{ gdrId }` OR the source document's four
keys (`sbId…` / `srId…` / `sdcId…` / `sdrId…`) — the service resolves the
register row. `?sync=true` runs the lane inline and returns the result;
without it the call enqueues and answers 202.

## 1. Document routes — `/api/v1/gst`

| route | body | response `data` |
|---|---|---|
| `POST /gst/einvoice/generate` | `{ gdrId \| keys, sync? }` | `{ status, irn, ackNo, ackOn, signedQr?, ewb?: { number, validUpto }, jobId?, message? }` |
| `POST /gst/einvoice/cancel` | `{ gdrId \| keys, reasonCode: 1..4, remark }` | `{ status: "CANCELLED", cancelledOn, ewbCancelled: true }` — SYNC always; 409 `GST_IRN_WINDOW_PASSED` past `EINV_CANCEL_HOURS` |
| `GET /gst/einvoice/get?gdrId=` | — | the `gde_` row (refreshes from NIC when PENDING / unknown) |
| `GET /gst/einvoice/preview?gdrId=` | — | `{ payload: {…NIC JSON…}, checks: [ { code: "GST_NIC_FIELD_BUYER_LOC", message, field } ] }` |
| `POST /gst/ewaybill/generate` | `{ gdrId \| keys, sync? }` | `{ status, number, generatedOn, validUpto, jobId?, message? }`; 422 `GST_EWB_TRANSPORT_MISSING` with `fields[]` |
| `POST /gst/ewaybill/vehicle` | `{ gdrId, vehicleNo, mode, reasonCode, remark, fromPlace, fromState }` | `{ status, number, validUpto, vehicleNo }` |
| `POST /gst/ewaybill/extend` | `{ gdrId, fromPlace, fromState, fromPincode, remainingKm, reasonCode, remark, consignmentStatus: "M"\|"T", transitType?, address? }` | `{ status, number, validUpto }` |
| `POST /gst/ewaybill/cancel` | `{ gdrId, reasonCode, remark }` | `{ status: "CANCELLED" }`; 409 `GST_EWB_WINDOW_PASSED` |
| `GET /gst/ewaybill/get?gdrId=` | — | the `gdw_` row |
| `GET /gst/ewaybill/preview?gdrId=` | — | as e-invoice preview |
| `POST /gst/ewaybill/manual` | `{ gdrId, ewbNo, generatedOn, validUpto, vehicleNo? }` | `{ status: "GENERATED", source: "MANUAL" }` |
| `GET /gst/gstin/lookup?gstin=&companyId=` | — | `{ gstin, legalName, tradeName, stateCode, regType, status, address }` |
| `GET /gst/jobs?gdrId=` | — | `[ { galId, action, attemptNo, startedOn, finishedOn, httpStatus, providerCode, ourCode, isSuccess, message } ]` (redacted) |

Status vocabulary: NA · PENDING · GENERATED · FAILED · CANCELLED · EXPIRED · REJECTED.
"Duplicate IRN" (NIC 2150) and "EWB already generated" (312) answer
`GENERATED` with the harvested number — never an error (G5).

## 2. Config routes (masters; grids for lists)

`POST /gst/providers/create|get|delete` (+ `/services`, `/endpoints`,
`/field-maps`, `/error-maps` nested by parent id), `POST /gst/provider-accounts/create|get|delete`
(secrets write-only; `get` returns `hasClientSecret`, `keyVersion`),
`POST /gst/company-credentials/create|get|delete`, `POST /gst/company-credentials/verify { gccId }`,
`GET /gst/company-credentials/status?companyId` →
`[ { service, environment, hasCredential, sessionValidUntil, lastError, balance } ]`.

## 3. Error codes

| code | when |
|---|---|
| GST_NOT_APPLICABLE | register flag false (B2C, bill of supply, company not e-invoicing) |
| GST_NO_CREDENTIAL · GST_AUTH_FAILED · GST_PROVIDER_DOWN | lane cannot start; retryable |
| GST_NIC_FIELD_* | the pre-check named the field |
| GST_EWB_TRANSPORT_MISSING | with `fields[]` |
| GST_IRN_WINDOW_PASSED · GST_EWB_WINDOW_PASSED · GST_EWB_EXTEND_WINDOW | |
| GST_DUPLICATE_HARVESTED | informational in the log, never returned as an error |
| GST_SANDBOX_NOT_PRINTABLE | a print asked for a SANDBOX IRN (G8) |

## 4. Settings (module GST)

`gst.environment` (COMPANY, SANDBOX), `gst.worker_enabled` (server),
`gst.retry_backoff_seconds` (`[30,120,600,3600]`), `gst.einvoice_print_requires_production`
(true), `gst.ewb_expiry_alert_hours` (8).
