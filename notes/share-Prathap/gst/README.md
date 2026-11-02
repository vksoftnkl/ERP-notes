# gst — the GST Suvidha Provider (GSP) layer: e-invoice, e-way bill, GSTIN lookup

**2026-09-20.** Its own plan, on the user's ask ("add the GSP service
provider plan too"), because it is consumed by three modules (sales
raises the IRN, purchase verifies a GSTIN, accounts files the return) and
belongs to none. The sales plan (`../sales/`) only ENQUEUES work here.

**The whole thing in one line:** a document that posts writes a register
row (`accounts.acc_voucher_doc_register`); when the company's flags and
the party make it e-invoice / e-way applicable, a job row goes to
`public.gst_api_log`; a worker authenticates through a leased session,
builds the NIC payload from the register + detail rows, sends it through
the endpoint row of the CONFIGURED provider, maps the reply through the
field-map rows into `gde_` / `gdw_`, and the error-map rows decide retry,
stop, re-auth, or "that error IS the success" — so a second provider is
an INSERT, not a release.

**Read in this order:**

| file | what it is |
|---|---|
| `../schema/11_gst_provider.sql` | the nine tables, written in August, NOT deployed: `gst_provider`, `_service`, `_account`, `_endpoint`, `_field_map`, `_error_map`, `gst_company_credential`, `gst_auth_session`, `gst_api_log` (partitioned). Chartered / TaxPro seeded INACTIVE |
| `gsp_flow.md` | the spec (§4a = what 3.0 knew — the NIC pre-checks, CRN / URP / DispDtls-ShipDtls-by-transaction-type, EwbDtls-in-IRN, Part-B / extension fields, get-IRN-by-doc, the GetKey asp-secret rule, the register's F-keys and JSON preview): the call sequence per action, the auth lease, the payload build, the response map, the error map, the 24-hour rules, offline EWB entry, what the register row shows at every step |
| `HANDOVER_endpoints.md` | the wire: every `/gst/*` route with body / response, the config routes, error codes, settings |
| `plan-backend-gsp.md` | the NestJS module: config CRUD, `GspClientService`, `GstDocumentService`, the worker, the nine routes, tests against the TaxPro sandbox |
| `plan-qt-gsp.md` | the Qt client: two masters (provider, company credential), the two registers (menus 152 / 153), the badges on the documents; ids to reserve |

**What exists today:** the superseded `fixed.gsp_provider_master` /
`gsp_company_service` and the two backend modules that serve them
(`accountsModule/gspProviderMaster`, `settings/gspCompanyService`) —
retired by this track; no Qt screen at all. TaxPro sandbox credentials
exist (Q13) and are NEVER written in these files or in chat — they go
encrypted into `gst_provider_account` / `gst_company_credential`, keyed
by Prathap's `GST_CRED_KEY`.

**SQL:** `11_gst_provider.sql` deploys in P0 with the sales SQL; the only
addition is `29 §6`'s partition helper covering `gst_api_log`.
