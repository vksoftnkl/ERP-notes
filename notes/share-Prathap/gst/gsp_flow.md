# GSP flow — how a document reaches NIC and what comes back

**2026-09-20, all defaults.** Read against `../schema/11_gst_provider.sql`,
the live `accounts.acc_voucher_doc_register` / `_doc_detail` /
`_doc_einvoice` (`gde_`) / `_doc_ewaybill` (`gdw_`) columns, the NIC
e-invoice schema v1.1 and e-way bill API v1.03 field lists, and 3.0's
`feinvoice_system` (flags 14/16) for the gestures — not for the shape.

---

## 0. The pieces

| table | role |
|---|---|
| `gst_provider` | who (CHARTERED = TaxPro, seeded; others by INSERT) |
| `gst_provider_service` | per provider × service (EINVOICE, EWAYBILL, GSTIN_VERIFY, GSTR) × environment (SANDBOX, PRODUCTION): base URL, auth scheme, token TTL, encryption flag |
| `gst_provider_account` | our contract with the provider (client id / secret, encrypted, key version) |
| `gst_provider_endpoint` | one row per ACTION (AUTH, GETKEY, GENERATE, CANCEL, GET, GET_BY_DOC, EWB_BY_IRN, EWB_GENERATE, EWB_VEHICLE, EWB_EXTEND, EWB_CANCEL, EWB_GET, GSTIN_GET — 11 already seeds most of these plus the consolidated / multi-vehicle set): method, path template, headers, wrapper, redact paths, success / error paths |
| `gst_provider_field_map` | REQUEST: our field → their path; RESPONSE: their path → our column (`gfm_target_column` = `gde_irn`, `gde_ack_no`, `gdw_no`, …) with transform / mask |
| `gst_provider_error_map` | their code → our code, `gem_treat_as` (ERROR / SUCCESS / WARNING), retryable, re-auth, recovery action, canonical field |
| `gst_company_credential` | the company's (or branch's) NIC login for a service + environment, priority (fail-over), whitelisted IPs, validity |
| `gst_auth_session` | the live token per credential, with a LEASE (`gas_lock_by / _on / _upto`) and a version |
| `gst_api_log` | every call: request (redacted), response, http status, provider code, our code, duration, `gal_idempotency_key`, `gal_attempt_no`; also the JOB QUEUE (a row with `gal_started_on NULL` is a pending job) |
| `acc_voucher_doc_register` + `_detail` | the document as GST sees it — the SOURCE of every payload |
| `_doc_einvoice` `gde_` / `_doc_ewaybill` `gdw_` | the RESULT: status, IRN / ack / QR / signed JSON; EWB no / valid-upto / vehicle |
| `acc_voucher_doc_api_log` `gdl_` | live, older, per-register log — kept as the document-side view; the worker writes BOTH (one row here per attempt, summarised) |

---

## 1. Who decides that a document needs an IRN / an EWB

The DOCUMENT's post (sales plan) — `DocRegisterService` computes
`gdr_is_einvoice_applicable` and `gdr_is_ewaybill_applicable` from the
company flags, the party, the value and the statutory rows
(`../sales/sales_posting_flow.md` §8). This layer never re-decides; it
serves what the register row asks for. Manual generation from the
register screen sets the flag to true first (logged), then enqueues.

---

## 2. The job

A job = a `gst_api_log` row: `gal_service`, `gal_action`, `gal_gdr_id`,
`gal_src_doc_type / _id`, `gal_company_id / _branch_id / _acc_year`,
`gal_idempotency_key` = `sha256(gdr_id + action + gde/gdw revision)`,
`gal_attempt_no 0`, `gal_started_on NULL`. Enqueued in the document's
after-commit hook, or by a register-screen action, or by the retry
sweep. `gde_status` / `gdw_status` = PENDING from the moment the job
exists.

The worker (one process, N lanes, one lane per company credential so the
auth lease is never fought over by the same company): picks the oldest
pending job whose credential is not leased, runs §3–§6, writes the
result, and either finishes the job or reschedules it (`gal_attempt_no +
1`, next attempt after `gem_retry_after_seconds` or the service's
back-off: 30 s, 2 min, 10 min, 1 h, stop at `gps_max_retries`).

---

## 3. Auth — the lease

1. Find the credential: `gst_company_credential` for (company, branch or
   NULL, service, environment) by `gcc_priority`; skip expired / inactive.
2. Find its session: `gst_auth_session` where `gas_expires_on − refresh
   margin > now()` and not locked by another worker → use the token.
3. Else take the lease (`UPDATE … SET gas_lock_by = me, gas_lock_upto =
   now() + 90 s WHERE gas_lock_upto < now() RETURNING`), call the AUTH
   endpoint row (TaxPro: client id + secret + the company's NIC login,
   `gps_payload_encryption` says whether the password is RSA-wrapped
   with `gcc_public_key_ref`), decrypt the session key, store token /
   session key / expiry encrypted, bump `gas_token_version`, release the
   lease.
4. NIC blocks a GSTIN after 5 auth calls in 15 minutes: the lease + the
   refresh margin are what make that impossible; a 4xx on AUTH sets
   `gcc_last_error_message` and stops the lane for 15 minutes.
5. Any call that returns a `gem_should_reauth` error invalidates the
   session (`gas_is_active false`) and retries once after a fresh auth.

---

## 4. The payload — built in CODE, mapped by ROWS

The canonical NIC e-invoice JSON (`Version`, `TranDtls`, `DocDtls`,
`SellerDtls`, `BuyerDtls`, `DispDtls`, `ShipDtls`, `ItemList`, `ValDtls`,
`EwbDtls`) is assembled by `EinvoicePayloadBuilder` from the register
row + detail rows + the company row + the transport block; the e-way bill
JSON likewise by `EwaybillPayloadBuilder`. This is CODE because the NIC
schema is one schema; what differs per provider is the ENVELOPE (wrapper,
headers, encryption) and where the answer sits — those are the endpoint
and field-map rows. A provider that wants a different field name maps it
in `gst_provider_field_map` (`gfm_direction 'REQUEST'`).

Mapping facts that bite: `DocDtls.Typ` = INV / CRN / DBN by
`gdr_doc_type`; `SellerDtls.Gstin` = `comp_gstin_no`; `BuyerDtls.Pos` =
`gdr_place_of_supply_code`; B2C never gets an IRN (the register flag is
false, §1); `ItemList[].HsnCd` digits per the company's class (the
document was warned at post); `Qty` / `Unit` = base qty / `unit_uqc`;
`ValDtls.TotInvVal` = `gdr_bill_value`; `EwbDtls` present only when the
transport block is complete (`gdw_transport_mode`, distance, vehicle or
transporter id) — then ONE call yields both IRN and EWB; else the EWB is
its own job (EWB_BY_IRN) once the transport block is filled from the
register screen.

E-way sub-supply types (`gdw_sub_supply_type`) from the document:
invoice → Supply; credit note → Sales Return (inward); DC purpose →
Job Work / Others / For Own Use / Line Sales / Exhibition (flow §4);
DC return → inward Sales Return-shaped "Others".

---

## 4a. What 3.0 knew (read 2026-09-20: `public.feinvoice_system` flags 5–22, `app_utils.fx_sales_ewaybill`, `GST/geneinvoice.cpp`, `GST/genewaybill.cpp`, `Utilities/gsteinvoiceapi.cpp`, `Sales/einvoicesystem.ui`)

Carried into this design, each with where it lands:

| 3.0 fact | ours |
|---|---|
| e-invoice applicable only when the party is REGULAR **and the bill carries tax** (cgst > 0 or igst > 0) and the company flag + date say so (flag 21) | `DocRegisterService`: IRN only on `INVOICE` / `CREDIT_NOTE` / `DEBIT_NOTE` with a party GSTIN and at least one taxable line — a BILL_OF_SUPPLY / all-exempt document never asks for one |
| e-way applicable = company flag + date only (flag 22); the client asked "no vehicle — generate IRN without e-way?" and went on (msg 462); vehicle without distance refused (msg 424) | §1 + the value limit; **IRN may go without the EWB** — the band opens itself but does not block the IRN; vehicle ⇒ distance > 0 is a pre-check |
| **NIC schema pre-checks before any call** (geneinvoice.cpp 250–330): seller name 3–100, addr1 1–100, location 3–50, PIN 6 digits, buyer GSTIN 15, buyer name 3–100, addr1 1–100, loc 3–50, state code present, HSN 4–8 digits, unit = UQC | `NicSchemaCheck` (plan §1) runs at the document's `/validate` + `/post` (WARN, code `GST_NIC_FIELD_*` naming the field) and again as a REFUSE just before the call — the operator fixes the master, never the payload |
| `DocDtls.Typ` = CRN for an inward supply (the return), INV otherwise; `DispDtls` omitted when transaction type ∈ {1 Regular, 2 Bill-To-Ship-To}; `ShipDtls` omitted when ∈ {1, 3 Bill-From-Dispatch-From} (flag 14) | the same rule in `EinvoicePayloadBuilder`, transaction type derived from the four address blocks (flow §6) |
| `ValDtls.AssVal` = gross − discount, `Discount 0`, `OthChrg` = TCS + other charges; per item `TotAmt = AssAmt = taxable`, `Discount 0`, `FreeQty 0`; `ItemList` per ITEM line (from a per-line tax snapshot `voucher_taxlog`), `Unit` = UQC | `acc_voucher_doc_detail` (`vtx_`) IS per item line (item id, qty, rate, unit) — the builder reads it, never the sale tables; free items go as `FreeQty` on the paid line where the scheme says so, else a 0-value line |
| `EwbDtls` inside the IRN call only when a vehicle no is present → one call returns IRN + EwbNo + EwbDt + EwbValidTill (flag 14 + geneinvoice.cpp 114–120) | §4 — identical; `_doc_ewaybill` row written from the IRN reply |
| EWB JSON (flag 16): `supplyType` O/I, `fromGstin` / `toGstin` = **URP** for an unregistered party, from/to blocks swap for an inward document, `docType` INV / BIL, `transMode` null when no vehicle, `transactionType` 1–4, `dispatchFromGSTIN` / `shipToGSTIN` | `EwaybillPayloadBuilder`; ours adds `CHL` for the challan and `vehicleType` R / O (ODC) from `vehicle_master` |
| Part-B (flag 17): ewbNo, vehicleNo, fromPlace, fromState, reasonCode, reasonRem, transDocNo/Date, transMode, vehicleType | `/gst/ewaybill/vehicle` — same fields |
| Extension (flag 19): fromPlace, fromState, **fromPincode**, remainingDistance, extnRsnCode, extnRemarks, **consignmentStatus M/T**, transitType + address lines when in transit | `/gst/ewaybill/extend` — fields added (§6) |
| Get IRN **by document details** (`irnbydocdetails?doctype&docnum&docdate`) when the generate reply was lost | the unknown-outcome sweep (plan §4) uses exactly this |
| Token cached with its validity on `gsp_master` (`eauth_token`, `eauth_token_validity`); headers `aspid / password / user_name / eInvPwd / Gstin` on TaxPro's **eicore/dec/v1.03** ("dec" = TaxPro does the NIC encryption) | `gst_auth_session` + the lease; TaxPro service row `gps_payload_encryption false` |
| **New TaxPro API**: `asp-secret` header must be the ASP secret AES-encrypted with the key from a **GetKey** call (gsteinvoiceapi.cpp 287–293) | endpoint action `GETKEY` added to `gst_provider_endpoint` for CHARTERED; `GspClientService` fetches the key once per session and encrypts the secret — a row + one branch in the envelope, not a provider-specific class |
| GSTIN search on `commonapi/v1.1/search?Action=TP` | `GSTIN_GET` |
| Cancel reasons IRN / EWB: 1 Duplicate · 2 Order cancelled · 3 Data entry mistake · 4 Others | dropdown 60 |
| The register screen (`einvoicesystem`): period + company + customer + bill-no filters, counts Generated / Pending / Cancelled, **F-keys** Change period F5 · Print bill F6 · Generate IRN F7 · Get IRN F8 · Cancel IRN F3 · Generate EWB F9 · Cancel EWB F10 · Get EWB F11 · Print EWB F12, a progress bar for bulk, and a **JSON preview dialog** (`einvoicedetail`: "Generate JSON" → view → "Generate eInvoice") | registers 152 / 153 keep the same keys and counts; `/gst/einvoice/preview` + `/gst/ewaybill/preview` return the payload the worker WOULD send — the register's "Show JSON" and the first thing to look at on a FAILED row |
| `sales_tranlog` = one outbox row per bill with the party / ship / dispatch blocks, values, IRN + EWB state and cancel columns | that is `acc_voucher_doc_register` + `gde_` / `gdw_` — same idea, already live |
| `igston_intra` / `reverse_charge` flags in `TranDtls` | `gdr_is_reverse_charge` exists; **`gdr_igst_on_intra` added by 30 §5** (SEZ-in-state / specific cases) |

Not carried: the JSON built inside a SQL function (ours is a builder with a golden test), XML-in-a-parameter for the e-way detail update (flag 20), the OTP-based GSTR token path (`generateGSTToken`, hard-coded username) — the GSTR service is a later phase.

## 5. The call and the reply

1. Endpoint row → URL (`gpe_path_template` with `{gstin}` etc.), method,
   headers (auth token, session id, `gstin`, `user_name` from the
   credential), wrapper (`gpe_request_wrapper`, e.g. `{"Data": …}`),
   encryption of the body with the session key if the service says so.
2. Log the request with `gpe_redact_paths` applied (passwords, keys,
   tokens never land in `gal_request_payload`).
3. Send with `gpe_timeout_ms`; a network error is retryable.
4. Reply: `gpe_success_path` = `gpe_success_value` ⇒ success → RESPONSE
   field-map rows write into the `gde_` / `gdw_` columns
   (`gde_irn`, `gde_ack_no`, `gde_ack_on`, `gde_signed_invoice`,
   `gde_signed_qrcode`, `gde_qrcode`; `gdw_no`, `gdw_generated_on`,
   `gdw_valid_upto`), status GENERATED, `txn_status_log` IRN_GENERATED /
   EWB_GENERATED on the document.
5. Else `gpe_error_code_path` → `gst_provider_error_map`:
   * `gem_treat_as SUCCESS` — NIC 2150 "duplicate IRN" / EWB 312 "already
     generated": the issued number is in the error body at
     `gem_extract_path` → harvested into the canonical field, status
     GENERATED. This is the case every naïve integration gets wrong.
   * retryable → reschedule (with `gem_retry_after_seconds`);
   * re-auth → §3.5;
   * else FAILED: `gde_last_error_code / _last_message` (our code + the
     mapped message), the register screen shows it, the document is
     still POSTED (a GSP failure never un-posts).

---

## 6. The actions

| action | when | notes |
|---|---|---|
| GENERATE (IRN) | bill / credit note post, register action | idempotent on `gde_status GENERATED`; a re-run is a GET |
| CANCEL (IRN) | bill cancel / amend within `EINV_CANCEL_HOURS`, register action | reason codes NIC 1–4 (duplicate, data entry mistake, order cancelled, others) + remark; after the window: refused — raise a credit note. Cancelling the IRN cancels the EWB generated with it |
| GET (IRN) | register "fetch", retry after a timeout whose outcome is unknown | by IRN or by doc no + date + type |
| EWB_BY_IRN | IRN generated, transport block filled later | the e-invoice route |
| EWB_GENERATE | DC, DC return, bill without IRN (B2C over the limit), return with transport | the e-way route; needs `gdw_transaction_type` (Regular / Bill To-Ship To / Bill From-Dispatch From / Combination) derived from the four address blocks |
| EWB_VEHICLE (Part-B) | vehicle changed, DC left without one | new vehicle no / mode / reason; unlimited within validity |
| EWB_EXTEND | validity ending, goods in transit | allowed 8 h before to 8 h after expiry; needs from-place + from-state + from-PIN, remaining km, reason code + remark, consignment status M (moving) / T (in transit: transit type + address) |
| EWB_CANCEL | within `EWAY_CANCEL_HOURS`, not yet verified in transit | after that the RECIPIENT may reject within 72 h — a status we record (`gdw_status REJECTED`) from GET |
| EWB_GET | register fetch, status refresh | |
| GSTIN_GET | customer / supplier / ledger screens' "fetch" button | returns legal name, trade name, state, reg type, status → the screen fills the fields; nothing is stored here beyond the log |
| MANUAL_EWB | portal-generated EWB typed in (offline / provider down) | writes `gdw_no` / `gdw_valid_upto` with `gdw_status GENERATED` and `gal_our_code 'MANUAL'`; no call |

---

## 6a. Direction — which columns fill the register on an INWARD document

The EWB JSON rule above already says *"from/to blocks swap for an inward
document"*. This says which of OUR columns feed that swap, because getting it
wrong produces a payload that is accepted by the portal and wrong on the paper.

**The register no longer keeps ship / dispatch addresses (32 §2b).** It kept
three address blocks; only `gdr_party_*` was its own work, because GSTR-1 B2B
needs the recipient. Ship-to and dispatch-from appear in **neither GSTR-1 nor
GSTR-3B** — they exist only to build these two payloads, so they live with the
rest of the band. The builders read `ttd_from_*` / `ttd_to_*` for **every**
module, which is why `ck_ttd_doc_type` widens as each one arrives (33) and
already admits `STOCK_TRANSFER`.

**There is one source for all of it: `public.txn_transport_detail` (33).**

**Two sources, and the split is deliberate.** The **vehicle** (`*_vehicle_no`,
`*_vehicle_id`) and the crew stay on the DOCUMENT — they exist even when there
is no consignment — and everything else is `public.txn_transport_detail` (33).
The builder has the document loaded anyway, so Part-B's `vehicleNo` comes from
there and `transDocNo` / `transDocDate` / `transMode` / distance / the
addresses come from the band. Neither fact is stored twice. The builder reads the row for
`(ttd_doc_type, ttd_doc_id, ttd_acc_year)` and maps it straight across — and
because `ttd_direction` already encodes the swap, there is **no per-document
branch anywhere in the payload code**:

```
gdr_dispatch_*  <-  ttd_from_*      gdr_ship_*  <-  ttd_to_*
```

| document | `ttd_direction` / `gdr_doc_flow` | `ttd_from_*` (FROM) | `ttd_to_*` (TO) |
|---|---|---|---|
| sale bill | OUTWARD | our godown / branch | the customer's ship-to |
| delivery challan | OUTWARD | our godown / branch | the customer's ship-to |
| **sale return** | **INWARD** | **the customer's pickup** | **our receiving godown** |
| **DC return** | **INWARD** | **the customer's pickup** | **our receiving godown** |

So on a return the **pickup is the FROM**, which is the whole reason the band
keeps a full address block on both ends rather than reusing the bill-to
snapshot: a customer with several locations returns from one of them, and the
e-way bill has to name the place the goods actually started.

**`acc_voucher_doc_register` has NO transport columns, and must not gain any.**
Transport is an e-way concern — the e-invoice never carries it — so it would be
dead weight on every e-invoice-only document. It lands in
`acc_voucher_doc_ewaybill` instead, which reaches the addresses through
`gdw_gdr_id`. The register holds the two ADDRESS blocks only, because those are
the one part both calls need.

**And `gdr_dispatch_*` / `gdr_ship_*` duplicating `ttd_from_*` / `ttd_to_*` is
deliberate — do not "tidy" it away.** `ttd_` is the working copy and moves when
a document is amended; `gdr_` is frozen at post and must keep what was actually
DECLARED. The GSTR-1 builder reads `gdr_` and never touches the sales tables,
which is the whole reason it exists.

Two consequences the builder must honour:

* `supplyType` is **I** and `DocDtls.Typ` is **CRN** for the sale return (it is
  a credit note), **CHL** for the DC return — it moves goods and raises no
  debt.
* When the pickup block is empty the service falls back to the customer
  snapshot (`sr_cust_*`), which is right for the common case and wrong for a
  multi-location customer — so `/validate` warns
  (`GST_EWB_PICKUP_ASSUMED`, overridable) rather than filling it silently.

Part-B's `transDocNo` / `transDocDate` are the document's `*_lr_no` /
`*_lr_date`, and they are **stored on the e-way row** as
`gdw_transport_doc_no` / `gdw_transport_doc_date` (32) — not re-read from the
document, which may since have been amended.

---

## 6b. Locking — what freezes when, and why the band is a separate table

The band is NOT in its own table to save space or even only to avoid four
copies. **It is there because it has a different mutability lifecycle from the
document**, and that is the argument that actually decides it.

A Part-B vehicle update is a normal, expected act for the whole life of an
e-way bill: the lorry breaks down, the goods trans-ship, the transporter
changes. Meanwhile the invoice itself must not move at all once an IRN exists —
GST law gives you a 24-hour cancel and after that a credit note. If transport
were columns on `sale_bill`, "lock the bill" and "let them change the vehicle"
would be the same sentence, and one of the two rules would have to be broken.
Separate tables let both be true.

### Two lock points, and the second is absolute

| # | when | frozen |
|---|---|---|
| 1 | **POST** | the document's content — items, values, party. The band stays open: nothing has been declared yet |
| 2 | **IRN or EWB generated** | **everything, the band included. No amend — cancel only** |

Either generation trips lock 2: an IRN with no e-way bill freezes the band just
as an e-way bill does, because the IRN payload carries `ShipDtls` / `DispDtls`.
There is no partial state.

**A later vehicle change is a PORTAL operation, not an edit.** It goes through
`/gst/ewaybill/vehicle` and lands in `gdw_`; it never re-opens the band. So each
table has one job:

| | holds | after lock 2 |
|---|---|---|
| document + band | what we PREPARED to declare | frozen |
| `gdw_` | what we DECLARED, and every update since | the only thing that moves |

**Read rule that follows, and it is the easy one to get wrong: once an e-way
bill exists the CURRENT vehicle is `gdw_vehicle_no`, not the document's.** The
document keeps the original. Same for the transport doc no and the mode.

Derive both locks — lock 1 is `*_status = 'POSTED'`, lock 2 is
`gde_status = 'GENERATED' OR gdw_status = 'GENERATED'`. Never a stored flag.

### What the band can and cannot generate

It is the TRANSPORT source for both calls. It is not the whole source of
either, and it cannot be.

| block | IRN needs it? | EWB needs it? | comes from |
|---|---|---|---|
| seller / buyer | yes | yes | `gdr_` + company |
| **item lines** — HSN, qty, rate, four tax rates | **yes** | HSN summary | **`acc_voucher_doc_detail` (`vtx_`)** — 15 item columns |
| value totals | yes | yes | `gdr_` |
| `ShipDtls` / `DispDtls` | yes | — | **the band** |
| from / to, transporter, distance, mode, LR | — | yes | **the band** |
| `vehicleNo` | in `EwbDtls` only | yes | **the document** (see §6a) |
| supply type, doc type, applicability | yes | yes | `gdr_`, derived at post |

An IRN is mostly items and values, and the band supplies two of its six blocks.
An e-way bill is roughly half band. So: **build both payloads from `gdr_` +
`vtx_` + the band**, and let the band own transport only.

### Why `acc_voucher_doc_register` does not become redundant

If the document is frozen after posting, a snapshot of it looks pointless — and
that is the weaker of the two reasons `gdr_` exists. The stronger one:

1. **It spans eight modules** — `gdr_source_module` admits SALES, PURCHASE,
   INVENTORY, ACCOUNTS, SERVICE, JOBWORK, POS, OTHER. GSTR-1 and GSTR-3B read
   ONE table. Without it, every return would UNION the sales documents, the
   purchase documents and the job-work documents, and would break every time a
   new document type appears.
2. **It holds seven classifications that exist nowhere on the document** —
   `gdr_tran_nature`, `gdr_doc_flow`, `gdr_doc_sign`, `gdr_doc_type`,
   `gdr_supply_nature`, `gdr_is_einvoice_applicable`,
   `gdr_is_ewaybill_applicable`. Those are DECISIONS taken at post, not copies
   of anything. Delete the register and they have no home.

It is a normalising projection, not a duplicate. Its address blocks happen also
to be a frozen copy, which is useful, but that is a side effect rather than the
reason.

---

## 7. What the register row shows, step by step

`gde_status` / `gdw_status`: NA (not applicable) → PENDING (job queued)
→ GENERATED / FAILED (with code + message) → CANCELLED; EWB also
EXPIRED (by `gdw_valid_upto`, a sweep) and REJECTED. The document's
`/get` copies these into `posting.irn` / `posting.ewb`; the document
screens show badges; the register screens (152 / 153) list and act.

---

## 8. Sandbox → production

`gps_environment` on the service row; the credential row picks the
environment; the company setting `gst.environment` (COMPANY, default
SANDBOX) says which the workers use. Switching = one setting + a
PRODUCTION credential row; the seeded TaxPro SANDBOX rows stay for
tests. IRNs generated in SANDBOX are marked (`gal_environment`) and the
register screen labels them, so a sandbox IRN is never printed on a
real invoice: the print template reads `gde_irn` only when
`gal_environment = 'PRODUCTION'` for that row.

---

## 9. Decisions (defaults, 2026-09-20)

* **G1** payload in code, envelope in rows — the NIC schema is one
  schema; providers differ in transport, not in content.
* **G2** the job queue IS `gst_api_log` (a row with `gal_started_on
  NULL`) — one table, one partition rule, no second queue.
* **G3** one worker lane per credential — the auth lease is a property
  of the credential, and NIC's 5-per-15-min rule is per GSTIN.
* **G4** a GSP failure never fails a post; the document stays POSTED and
  the register shows FAILED with the mapped message.
* **G5** "duplicate" errors are successes (error map, not code).
* **G6** cancel-IRN inside the window is done SYNCHRONOUSLY by the
  document's cancel / amend (it must know the outcome before it
  proceeds); everything else is the worker's.
* **G7** manual EWB entry is a first-class action, not a back door.
* **G8** sandbox IRNs are never printed.
