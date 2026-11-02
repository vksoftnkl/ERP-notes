---
name: erp-notes-80-txn-status-log
description: "Notes 80 (2026-10-02): txn_status_log actor/device/doc-type fixes built + verified, uncommitted; migration 20261002160000 already applied to dev DB"
metadata:
  node_type: memory
  type: project
  originSessionId: 4408a50b-2e08-41ab-86bf-1ec9b953f59e
  modified: 2026-10-02T11:43:45.959Z
---

Notes 80 (Qt History dialog, grid 126) asked for 3 fixes to what `public.txn_status_log` records. All three were built on 2026-10-02 and are uncommitted. An earlier session wrote most of the code and applied migration `20261002160000_txn_status_log_notes_80` to the dev DB at 17:00. A later session finished the specs and verified everything.

- **A (actor):** `statusActorOf()` in the helper. The real path was `bill.service.ts` `logStatusChange`, not `SalesDocStore.trail` as the notes guessed; SalesDocStore callers already passed uuids. The migration backfilled the DEFAULT_ACTOR rows: 0 left on dev.
- **B (device):** the accounts services pass `requestContext.getDeviceId()`, falling back to the header's device. Test: `test/txn-status-log-device-http.e2e-spec.ts`. `bootApp()` in `test/helpers/payment-e2e.ts` now takes claim overrides.
- **C:** OPENING_STOCK / PHYSICAL_STOCK were added to `ck_tsl_src_doc_type` and the enum. Trail rows of existing vouchers were re-filed. Rows of hard-deleted vouchers stay STOCK_ADJUSTMENT.

**Why:** the stock readers (list, get, the cancelled 409) and `nextSeqNo` look a trail up by doc type. Changing a screen's `statusDocType` without re-filing its old rows would orphan its history.

**How to apply:** `test/opening-cancel-http` "409 when DRAFT" fails, but it is stale: drafts became cancellable before notes 80. Related: [[erp-a2-posting-services-state]], [[erp-server-http-testing-without-credentials]].
