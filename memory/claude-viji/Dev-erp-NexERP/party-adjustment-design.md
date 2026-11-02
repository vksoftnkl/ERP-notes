---
name: party-adjustment-design
description: "Open-credit adjustment (advance + credit note) on the sale bill — one shared NexTable panel, in tender and outside it; adjustment is never a tender"
metadata: 
  node_type: memory
  type: project
  originSessionId: 870ac0e3-9eb8-4823-b68f-324bdbf3317a
  modified: 2026-08-15T05:42:51.710Z
---

Built 2026-08-15. `src/shared/adjustments/` — `PartyAdjustCols` entity,
`PartyAdjustDelegate`, `PartyAdjustGridController`, `PartyAdjustPanel`,
`PartyAdjustDialog`. An order **advance** and a sale-return **credit note** are
one instrument (a CR row on `accounts.acc_bill_balance`), so one grid handles
both.

Locked decisions:

- **An adjustment is NOT a tender.** The ADJUST row (tender type 7, renamed
  `RRN → ADJUST`; `isAdjustTender()` accepts both names) is a read-only MIRROR
  of the panel total. It is skipped in `recalcTotals()` and produces no
  `SaleTenderLine` in `collectAndEmit()`. It posts through the payload's
  `adjustments` array to `acc_bill_adjustment` only — emitting a tender line as
  well would write the same money to `acc_tender_detail` too.
- **One panel instance, owned by `SaleBillEntry`**, lent to `SaleBillTender`
  (`setAdjustPanel`) and to the modal on **F4**. Two entry points cannot
  double-count because there is only one set of amounts. F4 is the only free
  function key (F7 is `btnClear`, not free). Both hosts release the panel in
  their DESTRUCTOR, so the screen re-adopts it on `destroyed()`, never on
  `finished()` — the other order leaves the panel parentless.
- **Endpoint** `GET /transactions/party-balance?partyId&companyId&type` —
  `type=CR` = the party's open credits, advances + credit notes merged (the
  sale bill's list); `type=DR` = their open invoices (a future sale return's
  list). The values are the ledger's own vocabulary, `abl_dr_cr`. No branch and
  no accYear params by design. `partyId` is the CUSTOMER id; the server
  resolves the ledger, which closes the `:iparty_id` gap. The constant is a
  bare path — `AppConfig::baseUrl` already ends in `/api/v1`. Retired
  `SaleReturns::OpenList`, whose branch/year params encoded the wrong model.
- **`ui_table_master` grid id 25** (user-assigned) — its column rows still need
  seeding backend-side. The controller applies a LOCAL column seed right after
  `loadLayout()` because `loadLayout()` hides every column synchronously and
  only unhides what the server names back.
- **Absent ≠ empty on re-save.** `m_adjustmentsAuthoritative` is false when a
  loaded bill had `sbAdvanceAmt > 0` but the GET returned no `adjustments`;
  `buildSavePayload()` then OMITS the key rather than sending `[]`, which a
  server may read as "reverse everything".
- `sbAdvanceAmt` stays the ADVANCE-typed roll-up only (it feeds the order's
  advance ledger); credit notes land in `sbPaidAmt`/`sbBalanceAmt`.
- Customer change with keyed adjustments asks first and RESTORES the previous
  customer on "no" — safe because `NexDropdownSingle::setSelection()`/`clear()`
  do not emit `selected` (only `handleRowChosen`/`onBackspace` do).

Bug fixed on the way in: type 7 had `needsRef = true` in `kTypeRules`, so
`checkInputValues()` refused every settlement that adjusted anything — the row
it demanded a reference for is the one row nobody can type into.

**Open with the backend:** whether `sbAdvanceAmt` or `adjustments` is
authoritative (both describe the ADVANCE settlement — if the server posts both,
the advance posts twice); and whether `sale_bill` has a column for the
credit-note half. See also [[sales-backend-schema-design]],
[[grid-delegate-edit-pattern]], [[widget-signal-wiring-rule]].
