# GSP layer — backend plan (NestJS)

**For Prathap. 2026-09-20, all defaults.** The spec is `gsp_flow.md`.
Module folder: `src/modules/gst/` (new; the layer belongs to no existing
module's folder — the same reasoning that put the tables in `public`).

**SQL first:** `../schema/11_gst_provider.sql` (P0, with the sales SQL),
the `gst_api_log` partitions through `29 §6`, TaxPro SANDBOX rows set
`gps_is_active true` / `gpa_is_active true` and the credentials inserted
ENCRYPTED by a one-off script that reads them from the environment — the
values never appear in a file in the share or in a chat.

---

## 1. Module layout

```
src/modules/gst/
  gst.module.ts
  config/                       masters — CRUD, admin only
    gst-provider.controller.ts / .service.ts          /gst/providers        (+ services, endpoints, field-maps, error-maps as nested resources)
    gst-provider-account.controller.ts / .service.ts  /gst/provider-accounts
    gst-company-credential.controller.ts / .service.ts /gst/company-credentials
    crypto/gst-crypto.service.ts                      AES-256-GCM of every *_enc column with GST_CRED_KEY; gcc_key_version / gpa_key_version for rotation
    dto/…                                             write DTOs carry plain secrets ONCE (write-only); read DTOs come from public.vw_gst_credential (no _enc)
  client/
    gsp-client.service.ts                             auth lease, envelope, send, log, map reply, error map  (flow §3, §5)
    gsp-auth-lease.ts                                 the UPDATE … RETURNING lease + refresh margin
    gsp-envelope.ts                                   path template, headers, wrapper, body encryption per gps/gpe rows
    gsp-response-mapper.ts                            gst_provider_field_map RESPONSE rows → gde_/gdw_ columns
    gsp-error-mapper.ts                               gst_provider_error_map → {ourCode, treatAs, retryable, reauth, retryAfter, extract}
  payload/
    einvoice-payload.builder.ts                       NIC e-invoice v1.1 JSON from register + detail + company + transport
    ewaybill-payload.builder.ts                       NIC EWB v1.03 JSON; transaction type from the four address blocks
    payload.spec.ts                                   golden JSON per document kind (INV B2B intra, INV inter with EwbDtls, CRN, DC job-work, DC return)
  document/
    gst-document.controller.ts                        the nine document routes (§3)
    gst-document.service.ts                           applicability re-check, enqueue, sync cancel-IRN, manual EWB
    gst-job.queue.ts                                  enqueue(action, gdrId, keys) → gst_api_log row; nextPending(lane)
  worker/
    gst-worker.service.ts                             @Interval lanes per credential; back-off; sweeps (EWB expiry, unknown-outcome GET)
  payload/nic-schema-check.ts                       the NIC field rules (lengths, PIN, GSTIN, HSN 4–8, UQC, vehicle ⇒ distance) — WARN at the document's /validate + /post, REFUSE before the call
  gst.guards.ts                                       rights (152 / 153 menus' um_can_post = generate, um_can_cancel = cancel), environment, window checks via StatutoryService
  gst-exception.filter.ts                             codes GST_*
```

`StatutoryService` is imported from `sales/posting` (the windows:
`EINV_CANCEL_HOURS`, `EWAY_CANCEL_HOURS`; the value limits are the
document's business, already decided at post).

---

## 2. Config routes (masters; grids for lists)

| route | notes |
|---|---|
| `POST /gst/providers/create` `GET /get` `POST /delete` | + `/services/create|get|delete`, `/endpoints/…`, `/field-maps/…`, `/error-maps/…` keyed by parent id. A second provider is rows through these — no code |
| `POST /gst/provider-accounts/create|get|delete` | secrets write-only; `GET` returns `hasClientSecret: true` and the key version, never the value |
| `POST /gst/company-credentials/create|get|delete` | same; `POST /gst/company-credentials/verify` runs one AUTH call and stamps `gcc_last_verified_on` / `gcc_last_error_message` |
| `GET /gst/company-credentials/status` | per company × service × environment: credential present, session valid until, last error, credit balance (`gpa_credit_balance` refreshed by a daily BALANCE action if the provider has one) |

Retire `accountsModule/gspProviderMaster` and `settings/gspCompanyService`
+ drop `fixed.gsp_*` once the two Qt masters (plan-qt-gsp §1) are on the
new routes.

---

## 3. Document routes — all keyed by `companyId, branchId, accYear` + `gdrId` (or the source doc keys), all idempotent on `gde_/gdw_status`

| route | does |
|---|---|
| `POST /gst/einvoice/generate` | applicability re-check (register flag true, company flag, party GSTIN, environment); enqueue GENERATE; 202 with the job id. `?sync=true` (register screen "Generate now") runs the lane inline and returns the result |
| `POST /gst/einvoice/cancel` | `{reasonCode 1–4, remark}`; window check; SYNC call (G6); on success `gde_status CANCELLED`, `gde_cancel_reason`, `gde_canceled_on`, and the linked EWB CANCELLED; the DOCUMENT's cancel/amend calls this service method directly inside its own flow |
| `GET /gst/einvoice/get` | by gdrId → refresh from NIC (GET action) when status is PENDING/FAILED-unknown; returns the `gde_` row |
| `POST /gst/ewaybill/generate` | EWB_BY_IRN when an IRN exists, else EWB_GENERATE; needs the transport block (returns `GST_EWB_TRANSPORT_MISSING` listing the fields) |
| `POST /gst/ewaybill/vehicle` | Part-B `{vehicleNo, mode, reasonCode, remark, fromPlace, fromState}` |
| `POST /gst/ewaybill/extend` | `{fromPlace, fromState, remainingKm, reasonCode, remark, consignmentStatus}`; window ±8 h enforced by the error map as NIC returns it, pre-checked here |
| `POST /gst/ewaybill/cancel` | `{reasonCode, remark}`; window check |
| `GET /gst/ewaybill/get` | refresh + return `gdw_` |
| `POST /gst/ewaybill/manual` | `{ewbNo, generatedOn, validUpto, vehicleNo?}` → `gdw_status GENERATED`, `gal_our_code 'MANUAL'` (G7); rights: `um_can_post` on 153 |
| `GET /gst/gstin/lookup?gstin=` | GSTIN_GET; returns the taxpayer block; used by customer / supplier / ledger screens |
| `GET /gst/einvoice/preview` · `GET /gst/ewaybill/preview` | by gdrId → the exact payload the worker would send (redacted headers) — the register's "Show JSON", 3.0's `einvoicedetail` dialog; also runs `NicSchemaCheck` and returns its findings |
| `GET /gst/jobs?gdrId=` | the `gst_api_log` rows of one document (the register screen's log panel), redacted |

The document services (sales plan) call `GstDocumentService.enqueueFor
Register(gdrId)` in their after-commit hook and `cancelIrnSync(gdrId,
reason)` inside cancel / amend.

---

## 4. The worker

`GstWorkerService`: every 15 s, for each active credential lane not
already running: `nextPending(lane)` → `GspClientService.run(job)` →
write result → `appendTxnStatusLog` on the source document (IRN_
GENERATED / IRN_CANCELLED / EWB_GENERATED / EWB_CANCELLED, module
`SALES`, the document's keys) → finish or reschedule. Sweeps: hourly EWB
expiry (`gdw_valid_upto < now()` → EXPIRED); every 10 min "unknown
outcome" (a GENERATE whose HTTP call timed out) → GET by doc no before
any retry (a retry without the GET is how duplicates happen; the error
map catches it anyway). Concurrency: `SELECT … FOR UPDATE SKIP LOCKED`
on the job row; `gal_idempotency_key` unique per attempt series.

Single instance per deployment (a `gst.worker_enabled` DEVICE/SERVER
setting); the lanes are the parallelism.

---

## 5. Settings (`public.app_setting_def`, module GST)

| key | default | scope |
|---|---|---|
| `gst.environment` | `SANDBOX` | COMPANY |
| `gst.worker_enabled` | `true` | (server) |
| `gst.retry_backoff_seconds` | `[30,120,600,3600]` | COMPANY |
| `gst.einvoice_print_requires_production` | `true` (G8) | COMPANY |
| `gst.ewb_expiry_alert_hours` | `8` | BRANCH |

---

## 6. Tests (`test/gst/`, TaxPro SANDBOX, credentials from the environment)

Unit (no network): payload builders against golden JSON (5 kinds, incl. transaction types 1–4 with/without DispDtls / ShipDtls, CRN, URP buyer); NicSchemaCheck on each 3.0 rule;
response mapper with recorded TaxPro replies; error mapper: 2150 →
SUCCESS + IRN harvested, 312 → SUCCESS + EWB harvested, auth-expired →
reauth, 5xx → retryable, schema error → FAILED; envelope: redaction,
wrapper, encryption on/off.

Live (sandbox): auth lease under two lanes (only one AUTH call); generate
IRN for a B2B bill → GET → cancel within window (reason 2) → the linked
EWB is cancelled; duplicate generate → GENERATED via 2150; EWB by IRN
after filling transport; Part-B update; extend (expect NIC's window error
mapped, not a crash); EWB for a DC (job work) and a DC return (inward);
manual EWB; GSTIN lookup of the sandbox GSTINs; unknown-outcome sweep
(kill the call mid-flight, next tick GETs, no duplicate).

---

## 7. Phases

| phase | delivers |
|---|---|
| P0 | 11 deployed, TaxPro sandbox rows active, credentials inserted encrypted |
| P1 | config module + crypto + `GspClientService` + payload builders + unit tests; `/gst/gstin/lookup` live (first user-visible win: the fetch buttons) |
| P2 | document routes for the bill (IRN + EWB), worker, sandbox tests; sales bill post enqueues |
| P3 | DC / DC return EWB (with the sales P3) |
| P4 | credit-note IRN + return EWB (with the sales P4) |
| P5 | Qt — `plan-qt-gsp.md` |
