---
name: erp-txn-hold-contract
description: "/txn-holds (public.txn_hold) replaced /transaction-holds on 2026-08-31 — every column is txh_*, plus the create-required fields, the expiring lease and the device FK"
metadata:
  node_type: memory
  type: reference
---

The till-wide `public.txn_hold` (server module `modules/sales/txn-hold`, migration
`20260810160000_add_txn_hold`) **replaced** `transaction_hold`. The old routes are gone —
`POST /api/v1/transaction-holds/create` now 404s — and the client was ported on 2026-08-31
(Quotation Entry F9/F10; `features/sales/quotation/quotation.hold.ts` + `store/api/quotationApi.ts`).

What changed, beyond the `th_*` → `txh_*` rename:

- **Routes:** `/txn-holds/create` (create AND update, by `txhId`), `/list`, `/get`,
  `/delete`, and `POST /txn-holds/:id/{resume,release,force-release,convert}`.
- **Required on create** (service-enforced, so a miss is a 400 naming the field):
  `txhCompanyId`, `txhBranchId`, `txhAccYear`, `txhSrcModule`, `txhDocType`, `txhHoldNo`,
  `txhHoldSlno`, `txhDeviceId`, `txhHeldBy`, `txhPayload`. `QUOTATION` is a real doc type now.
- **`txhDeviceId` is a REAL FK** into `fixed.device_master` — send the login's `device_id`
  (`getUserInfo()?.deviceId`), never `getOrCreateClientDeviceId()`, which is a browser-local
  uuid matching no row. Same for the `X-Device-Id` header on every lease endpoint.
- **`txhAccYear` is `char(9)`** (`"2026-2027"`), not the old SMALLINT starting year, and it is
  half the primary key + the partition key: immutable, and worth sending on every `get`/`list`/
  lease call to prune to one partition.
- **`txhHoldSlno`** (int ≥ 1) is a per-device serial, unique per company/branch/year/docType/
  device — nothing generates it server-side. The client seeds it from the clock and counts up in
  `localStorage`; a fresh browser must NOT start at 1, because two browsers on one login share a
  `device_master` row.
- **`txh_payload`** (required, must be an object) replaced `th_ui_state`.
  `txhNetAmount` replaced `thTotalAmount`; `txhPartyName/Id/Type/Mobile` replaced `thCustomerName`;
  `txhHoldOn` replaced `thHoldDate`. `ck_txh_amounts`: item count, qty and net amount all ≥ 0.
- **The lock is a LEASE:** `txh_lock_expires_on` is written with every resume, so a lapsed lease
  is taken over by the next device with no force-release — treat a LOCKED row with a past expiry
  as FREE in the picker. `txhKind` (`HOLD`/`AUTOSAVE`/`TEMPLATE`) must be filtered in the list;
  the table is shared with the till.
- **`/convert`** takes no converted doc TYPE (the hold becomes what it was parked as) but
  REQUIRES `txhConvertedAccYear` — the document's own year, which may differ from the hold's.
- `txhSessionId` means the till's shift/day session; this client never sends one.

Verified end to end 2026-08-31 (curl + headless browser): create → resume → release → update →
convert → soft delete. Related: [[erp-quotation-entry-port]], [[erp-auth-endpoint-verification]],
[[erp-auth-session-userinfo]].
