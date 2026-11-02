---
name: erp-received-cheques-port
description: "Received Cheques (menu 51) React port — where it lives, which plan §14 backend gaps were re-checked live on 2026-09-24 and what they answered, and the live test that pins them"
metadata:
  node_type: memory
  type: project
  originSessionId: 64f3a51e-151b-4a50-bf3b-08ef6455149e
  modified: 2026-09-24T11:28:23.672Z
---

Menu 51 was ported 2026-09-24 under `features/accounts/cheques/` (shared with the
future menu 52 Issued Cheques: `domain/` + `actions/` are screen-agnostic, the
screen takes a `ChequeVocabulary`), `store/api/chequesApi.ts`, route
`/accounts/received-cheques`. Registry keys: grid `receivedChequeList` (109),
dropdowns `bankLedger` (25) and `ledgerAll` (43).

Re-checked live (the plan left these open):
- **§14.1 second bounce** (deposit→bounce→re-present→bounce) — FIXED, 201.
- **§14.3 replace from HELD with no `reason`** — ACCEPTED; the server writes its own
  ("Replaced by cheque N dated D"). The plan guessed it would be refused and asked
  for a required field; it is offered but optional. The Qt screen was not broken.
- **§14.2 replacement row is a dead end** — STILL OPEN: `/return` on it is a 400,
  "has no tender row … written by hand or by an import" (misleading wording). The
  client deliberately does not predict it.
- After a re-present, `/cheques/get` `receiptVoucher` (and grid `receipt_refno`) is
  the RE-ISSUE voucher, not the original receipt — so Enter drills to that.

`cheques.integration.test.ts` pins all of the above (skipped without `CHEQUES_IT_*`
env). Each run posts a real receipt with five cheques and leaves them — nothing in
this module can be deleted. Only users ravi / VKPOS / VKPOS1 held a menu 51 grant
when the href was bound, so everyone else is now refused the route until granted.

Related: [[erp-receipt-entry-port]], [[erp-client-ui-browser-verification]],
[[erp-menu-permission-gating]], [[erp-configured-grid-registry]].
