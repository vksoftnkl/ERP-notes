# Notes 79 — GSP masters: the routes the two screens need (2026-10-02)

For Prathap. The Qt screens for **GST Providers (menu 255)** and **GST Credentials (menu 256)**
are designed, but nothing is built yet. The mockups are next to this file:
`gst_provider_ui_mockup.png` and `gst_credential_ui_mockup.png` (each has a `.py` that renders it).
The screens cannot be built yet because nothing on the server reads or writes the new tables.
This note lists exactly what the screens will call. The full design is still
`plan-backend-gsp.md` §2. This note narrows it to the first slice and fixes the field contract.

## 1. Where things stand (checked live + source, 2026-10-02)

| | state |
|---|---|
| `11_gst_provider.sql` tables | **deployed**. 2 providers (NIC active, CHARTERED inactive), 2 provider accounts, 11 services, 23 endpoints, 25 error-map rows, **0 credentials** |
| Prisma models | present (`GstProvider`, `GstProviderAccount`, `…Service`, `…Endpoint`, `…FieldMap`, `…ErrorMap`, `GstCompanyCredential`) |
| Services / controllers on them | **none**. No `.ts` file references them |
| Routes in the API | only the OLD `/gsp-provider-masters/*` and `/gsp-company-services/*` (on `fixed.gsp_*`, which the plan retires) and `/gst/search` (GSTIN lookup) |

Building on the old routes would mean screens that write tables we are about to drop, so the
client waits for the routes below.

## 2. Routes, first slice (config only; no document routes yet)

The house rules apply: create = upsert by id; `get` by id returns the full row; no `/list`
routes, because the lists are grids I register; delete is soft, and there is a `/restore` like the item masters.
Field names use camelCase from the column prefix (`gpvCode`, `gpsBaseUrl`, `gccLoginId`…).

| # | route | body / query | notes |
|---|---|---|---|
| R1 | `POST /gst/providers/create` · `GET /gst/providers/get?gpvId=` · `POST /gst/providers/delete` · `POST /gst/providers/restore` | header columns of `gst_provider` | `get` also returns `services[]`, `accounts[]` (secret-free, see §3) and the counts `endpointCount` / `errorMapCount`. Delete is refused (409) while a live credential points at the provider |
| R2 | `POST /gst/provider-services/create\|delete` | `gpsGpvId` + columns; `gpsFallbackUrls: string[]` | (service, environment) is unique per provider → 409 `GST_SERVICE_DUPLICATE` |
| R3 | `POST /gst/provider-endpoints/create\|delete` · `GET /gst/provider-endpoints/get?gpeId=` | `gpeGpsId` + columns; `gpeHeaders` / `gpeRedactPaths` as JSON | `get` returns `fieldMaps[]`, so the field-map popup needs one call |
| R4 | `POST /gst/provider-field-maps/create\|delete` | `gfmGpeId` + columns | |
| R5 | `POST /gst/provider-error-maps/create\|delete` | `gemGpvId` + columns; `gemService` nullable = every service | |
| R6 | `POST /gst/provider-accounts/create\|delete` · `GET …/get?gpaId=` | `gpaGpvId, gpaEnvironment, gpaService?, gpaAccountRef, clientId?, clientSecret?, apiKey?` + validity / remarks / active | **secrets are write-only (§3)** |
| R7 | `POST /gst/company-credentials/create\|delete\|restore` · `GET …/get?gccId=` | `gccCompanyId, gccBranchId?, gccGpvId, gccService?, gccEnvironment, gccPriority, gccLoginId, password?, clientId?, clientSecret?, appKey?, gccWhitelistedIps: string[], gccValidFrom, gccValidUpto?` + remarks / active | the GSTIN is NOT a body field. The response carries `gstin` resolved like `vw_gst_credential` (branch GSTIN, else the company's). One priority-1 row per company + branch + service + environment (`ux_gcc_primary`) → 409 `GST_CREDENTIAL_PRIMARY_EXISTS` |
| R8 | `POST /gst/company-credentials/verify` | `{ gccId }` | **one** AUTH call through the lease. Stamps `gcc_last_verified_on` / `gcc_last_error_message`. Returns `{ ok, message, errorCode?, tokenValidUntil?, creditBalance? }`. If the lease is held, return 409 `GST_AUTH_BUSY` and do not call the portal: NIC blocks a GSTIN after 5 auths in 15 minutes |
| R9 | `GET /gst/company-credentials/status?gccId=` | | `{ hasLiveToken, tokenExpiresOn, issuedOn, leaseFree, lastVerifiedOn, lastErrorMessage, creditBalance }`, read from `gst_auth_session` + the credential, with no portal call |

Rights: menu 255 and 256 are admin. `um_can_edit` saves and `um_can_post` runs Verify (it reaches an outside system).

## 3. The write-only secret contract (R6, R7)

The client never sees a secret, plain or encrypted. This contract has to be exact, because the
form's "set · key v1" / "not set" pill and "an empty box keeps the stored value" depend on it.

* **In:** each secret key is optional plain text. *Absent or `""`* = keep what is stored.
  *A value* = encrypt with `GST_CRED_KEY`, store, bump `*_key_version`, and for the password stamp
  `gcc_password_changed_on`. To clear one: `"clear": ["clientSecret"]`.
* **Out:** never the value. Instead `hasPassword`, `hasClientId`, `hasClientSecret`, `hasAppKey` (or
  `hasApiKey` on R6) as booleans, plus `keyVersion`. The same holds for `get`, the create
  response and the audit log (the audit row must not carry the plain text either).
* `vw_gst_credential` stays the grids' read boundary. I will register the credential grid on the
  view, not the table.

## 4. What I do on the client once R1–R9 are live

* Register the grids (the server assigns the ids):
  * provider list on `gst_provider`;
  * credential list on `vw_gst_credential`;
  * child grids for services, endpoints, field map and error map (`ui_table`s for the editable children);
  * provider and service dropdowns.
* Build menu 255 / 256 to the two mockups.
* Then wire the GSTIN fetch buttons on customer / supplier / ledger / ship-to to `/gst/search`,
  which is already live.

## 5. Stale id reservations in `plan-qt-gsp.md` §0

These ids were reserved on 2026-09-20 and have since been taken by other screens:
* grids 121 / 122 / 125 (issued cheques, stock adjustment list, app themes);
* dropdowns 58–60 (customers by beat, voucher parties, payment payees);
* ui_tables 43 / 44 (payment tenders, change selling price rows).

The menus themselves are now **`../46_gst_menus.sql`**: GST Providers (8.30) and GST Credentials
(8.40) under Configuration (60), hidden until the screens ship. Their ids come from the sequence,
which made them **267 / 268** on the dry run (not 255 / 256), so please send back what the final SELECT prints.
152 / 153 already exist as the e-Invoice / e-Way Bill menus.
Since the server picks grid, dropdown and ui_table ids on create, the plan's numbers are void.
The ones actually created will be recorded when they exist.

## 6. Still open (not for these screens, but blocking going live)

* Chartered: the **aspid + ASP password** (goes in through R6, never a file).
* The real **e-way bill path**: all 14 EWB endpoints are `/CONFIRM-PATH`. One value fixes all 14.
* 9 e-invoice paths and the NIC numeric error codes (only GSP001–GSP505 are seeded).

After the two screens work on R1–R9, retire `accountsModule/gspProviderMaster`,
`settings/gspCompanyService` and `fixed.gsp_*` (plan-backend-gsp §2, last line).

## 7. GSTIN fetch buttons — client DONE, server setting missing (2026-10-02)

The fetch button next to GSTIN on Company, Branch, Supplier, Ledger and Customer now calls
`GET /gst/search` and fills name, legal / mailing name, registration type, PAN, state and address.
Live, every call answers **503 "Set GST_LOOKUP_ENDPOINT, GST_LOOKUP_ASP_ID, GST_LOOKUP_ASP_PASSWORD"**.
Please set those three in the server environment (sandbox:
`https://gstsandbox.charteredinfo.com/commonapi/v1.1/search` with the sandbox aspid / ASP password).
Until then the operator sees "GSTIN search is not configured on the server yet" and types the
details by hand. The client was tested against a mock in your `GstinLookupPayload` shape
(`tests/gstin/run.sh`, 28/28).

## 8. Client screens BUILT on R1–R9 (2026-10-02) — one server setting missing

Menus 269 / 270 are coded in the Qt client (grids 127–131, dropdowns 61 / 62) and tested live as
VKPOS (`tests/gstmasters/run.sh`, 31 checks). Every route works **except where a secret is written**:
`GST_CRED_KEY` is not set on this server, so each secret save answers
**503 "GST_CRED_KEY is not set on this server, so no GST secret can be stored or read"**.
That blocks two things:
* provider-account secrets (client id / secret / API key). An account with no secret saves fine;
* every new company credential, because the password is required on create.

Please set `GST_CRED_KEY` in the server environment. The client needs no change. Once it is set,
re-run `GST_CRED_KEY_SET=1 tests/gstmasters/run.sh`, which also checks the credential and the
secret round trip.
