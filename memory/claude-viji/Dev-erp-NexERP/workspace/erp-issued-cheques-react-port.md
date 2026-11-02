---
name: erp-issued-cheques-react-port
description: "2026-10-03 Issued Cheques (menu 52) + Cheque Books (menu 263) ported from Qt to React features/accounts/cheques/{issued,books}; uncommitted; own screen, NOT the received vocabulary"
metadata:
  node_type: memory
  type: project
  originSessionId: e05d84f3-470d-4bf0-bac8-1b311496d7c1
  modified: 2026-10-03T06:06:56.516Z
---

Built 2026-10-03 in `/home/vk/Dev/erp/ERP client`, UNCOMMITTED, alongside [[erp-bill-wise-payment-react-port]]:
`features/accounts/cheques/issued/` (screen, specs, domain), `features/accounts/cheques/books/` (panel used by the
`/accounts/cheque-books` page AND a dialog from Issued Cheques), `store/api/issuedChequesApi.ts`, routes
`/accounts/issued-cheques` + `/accounts/cheque-books`, grid registry `issuedChequeList` 121 / `chequeBookList` 120.
The old `ISSUED_VOCABULARY` (Present as an F5 bulk deposit) was wrong and was removed.

Why a separate screen: `/issued-cheques` is its own server module — BARE keys `apdId·apdAccYear·companyId·branchId`
(received uses `apdCompanyId`), single-cheque verbs presented/returned/stop/void/replace, POSTs answer 200, history
OLDEST first with nullable changedBy/toStatus, no summary route (tiles summed client-side from grid 121 HELD,BOUNCED +
grid 120 ACTIVE). Grid 121 tokens are `icompany_id/ibranch_id/iacc_year` (+ the received six), all nine always sent.

Server gotchas: Replace on BOUNCED/CANCELLED needs `/get`'s `reversalVoucherId` (a leaf cancelled by the payment's own
cancel/amend has none → 409); `paymentVoucherId` comes with no year (use the row's apdAccYear); grid runner caches a
URL ~1s, so re-read 1.1s after an action; cheque-book rights are menu 263, not 52; verb rights are post/cancel/amend,
which client MenuPermissions lacks, so verbs aren't client-gated (server 403 shown). `TrimmedString` allows "" →
client requires reasons. Enter on a PmtV (menu 261) cheque opens its carrier voucher in /accounts/payment-voucher by exact keys (since [[erp-receipt-payment-voucher-port]]).

Cheque Books panel was reworked 2026-10-03 to match Qt's ChequeBookDialog layout (grid 120 configured columns,
"Book" group with leaves table, message line, New/Save/Close book…/Done, Enter = next field). On dev, menu 263 is
granted to only 3 users vs 6 for menus 51/52/99/100 — a login without it sees no Cheque Books menu item.

Verified without login: bodies vs compiled server DTOs, grids 121/120 run read-only with the screen's params on dev
(52 rows, 1 active book). Never clicked through logged-in.
