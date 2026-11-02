---
name: erp-temp-credits-register
description: "Temp Credits (menu 257) rebuilt 2026-10-05 on CrudMasterPage at features/sales/temp-credit — F5 Receive hands over to the receipt by URL (?collect=1), F6 follow-up three-way promise date, Ctrl+H grid 132; shell gained hideWriteActions; older testbill/lists copy left in place"
metadata:
  node_type: memory
  type: project
  originSessionId: 59591584-91d1-44dd-ac7a-c056d6399d2d
  modified: 2026-10-05T14:10:03.508Z
---

Port of Qt `TxnDocTypes::tempCredits()` + `TempCreditFollowupDialog` (2026-10-05).
Route `/sales/temp-credits` now imports `features/sales/temp-credit/page.tsx`
(list on `CrudMasterPage` with `gridKey="tempCreditList"`, grid 114; Enter opens
the bill in salebill's `SaleBillEntryView` browse mode). The earlier copy in
`features/sales/testbill/lists/temp-credit-*.tsx` is unreferenced but NOT deleted.

**Receive (F5)** = `router.push(buildReceiptCollectHref(args))` →
`/accounts/receipt?collect=1&partyId&partyName&mobile&billId&ref&name`
(`features/accounts/receipt/domain/collect.ts`). The receipt page reads it with
`useSearchParams` (route wrapped in Suspense), strips the query via
`router.replace`, and `useReceiptDraft.collectTempCredit()` does Qt's
`applyOpenArgs`+`tickTempCreditBill`: NEW_DOCUMENT if dirty (confirm first),
PARTY_PICKED, `/receipts/open-items?mobile=` (new `mobile` arg on
`getReceiptOpenItems`; `BillRow.tempCredit` now carried through parse), ticks the
bill's Receive, puts the figure on the first instrument (held in
`pendingCollectAmount` ref until the tender-seeding effect adds the row). The
mobile filter persists across F5/date re-reads (`tempCreditMobile`,
`setTempCreditMobile("")` clears) and drops on `pickParty`.

**Why the URL hand-over:** two routes, and a link to "collect this one" the browser
can hold; Qt used `SessionEvents::requestOpenMenu(99, args)`.

**Gotchas hit:**
- grid 114 rows: `atc_party_id` null ⇒ Receive refuses (nothing to settle), as Qt.
- The shell asks `/configured-grid-sql/columns?grid_id=` (not `/grid-details/get`)
  for configured columns — fixture must answer an array of `grid_column_*` rows.
- `innerText` carries CSS text-transform: dialog titles read UPPERCASE in headless
  checks; compare case-insensitively.
- Shell auto-fit page size re-reads the list ~1-2s after first paint and clears the
  selection; click rows after it settles.
- `CrudMasterPage.hideWriteActions` (new) removes Add/Edit/Delete and zeroes the
  create/edit/delete rights so Alt+C/Alt+A/F1-F3 refuse too; `onViewAction` still
  reaches. The shell binds no Ctrl+H; the register binds F5/F6/Ctrl+H itself.
- Grid 132 `tempCreditHistory` has no ORDER BY — sort on `th_sort` client-side.

**Qt parity pass 2026-10-07:** the Qt `src/modules/sales/temp_credit/` folder
holds ONLY the follow-up dialog (fully ported); the register + history trail
live in `txn_doc_types_sales.cpp::tempCredits()` and `txn_history_dialog.cpp`.
Ported then: Receive/Follow-up grey out when balance ≤ 0 (Qt `enabledFor`) via
the shell's `toolbarActions` now accepting `(selectedRow) => ReactNode`; history
title "History · <bill>", subtitle, summary line (settled = SETTLE rows only,
never COUNTER), Event pills coloured by event TEXT (`history.ts` `eventTone`).

**menu_verbs (2026-10-07):** user asked for Create + Delete cells on 257 in User
Administration. Server migration `20261007100000_temp_credits_create_delete_verbs`
restores the full CRUD default and the seed's 257 narrowing block is retired (it
re-ran on every deploy).

**Add + Delete (2026-10-07, user's choice):** `hideWriteActions` replaced by the new
shell `hideEditAction`, so 257's Create/Delete grants now gate real buttons.
- **Add** (Alt+C) = a fresh `SaleBillEntryView` in entry mode ON THIS ROUTE
  (`page.tsx` view union); Qt had `allowAdd=false`. The bill's save then checks
  menu 257's Create (usePagePermissions resolves by route), not menu 12's.
- **Delete** = NEW server `POST /temp-credits/delete {atcId, atcAccYear, reason}`:
  soft delete (`atc_is_deleted`) of the register row ONLY — bill, tender, balance
  untouched, status column unchanged; trail step `TxnStatusEvent.DELETED`; audit
  `'cancel'`. Reason is typed in the shell's confirm (`deleteReasonPrompt`).
  Every atc reader (grid 114, bill read/cancel/retender, receipts, recompute)
  already filters `atc_is_deleted`, and `ux_atc_tender` is partial on it — so a
  later bill AMEND re-raises a fresh credit for a still-TEMP_CR tender.

Related: [[erp-receipt-entry-port]], [[erp-testbill-module]],
[[erp-crud-master-extensions]], [[erp-client-ui-browser-verification]].
