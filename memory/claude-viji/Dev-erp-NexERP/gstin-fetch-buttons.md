---
name: gstin-fetch-buttons
description: "GSTIN fetch buttons DONE 2026-10-02 on Company/Branch/Supplier/Ledger/Customer via shared/gst/gstin_lookup → GET /gst/search; live answers 503 until the server's GST_LOOKUP_* env is set (notes 79 §7)"
metadata:
  type: project
---

`src/shared/gst/gstin_lookup.h/.cpp`: `GstinLookup::install(form, fetchEdit, GstinFillMap)`.
- It validates the 15-character shape first, then confirms in edit mode, then calls `ApiEndpoints::Gst::gstinSearch` → `/gst/search`. The old constant pointed at `/gst/gstin/lookup`, which never shipped.
- It fills through `NexCrudForm::fillFromJson`, so each control kind is set natively. Number PINs (company/branch) need a JSON number (`pinIsNumber`). Reg type is coded on company/branch and text ("Regular") on supplier/customer/ledger.
- Names: trade name (falling back to legal) goes to the name; legal goes to compLegalName / ledMailingName / brMailingName. Branch never overwrites brName.
- A non-Active status (e.g. Cancelled) fills the form and then warns. 503/404/502/400 each get their own operator wording.

**The server contract** (nexerp-api-ro settings/gstinLookup) is GstinLookupPayload: gstin, legalName, tradeName, status, registrationType, gstRegType, stateCode, panNo, registeredOn, address{building, street, locality, city, district, state, pin}, raw. Since 2026-10-02 it reads env only (GST_LOOKUP_ENDPOINT / _ASP_ID / _ASP_PASSWORD) and answers 503 without them. The old fixed.gsp_* tables were dropped.

**Test:** `tests/gstin/run.sh` runs the real forms against `mock_gst_search.py` (28/28). Re-check live once Prathap sets the env. Ship-to has no fetch button yet. Related: [[gsp-masters-blocked]].
