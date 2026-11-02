---
name: notes-143-opening-backend-review
description: "Notes 143 (2026-10-23) stock voucher hardening — row lock + in-tx preflight under item locks, base qty = qty × line factor (422), stock-voucher cancels never go negative, login is the actor, refno series skips taken numbers"
metadata:
  type: project
---

Built 2026-10-23 (uncommitted at hand-off), all in StockVoucherService / posting helper, so every
stock voucher type (opening, physical, adjustment family, transfers) gets it:
- #1 `loadForWrite` = raw `SELECT … FOR UPDATE` + company/branch check (404), taken in save() BEFORE
  the notes-89 `before` read; updateDraft no longer re-checks.
- #2 post(): row lock → assertDraft → `assertDocDateInYear` → `lockForPreflight` (exported
  `lockVoucherItems`, same company:branch:item advisory key the engine takes; skipped for transfer
  shapes) → assertPostable(tx). Save-and-post takes the item locks too. Proven live: two concurrent
  posts of one holding → one 422 "already has an opening".
- #3 assertPayloadRules: baseQty / freeBaseQty (0 if omitted) must equal qty × the LINE's factor at
  numeric(18,6) → 422 STOCK_LINE_(FREE_)BASE_QTY_MISMATCH (user chose refuse, not derive). Preflight
  also refuses baseUomId ≠ the base row the uom points to (iuc_base_unit_id) and stored mismatches.
- #4 user decision: a STOCK VOUCHER cancel never goes negative whatever stp_allow_negative →
  409 STOCK_CANCEL_WOULD_GO_NEGATIVE (`refuseNegative` passed only by StockVoucherService.cancel;
  sales shadow reversals keep the item policy).
- #5/#7 preflight: item other company / service (all), other branch + inactive item/godown only on
  SIMPLE docs (a transfer receipt of a branch-specific item must still land — the transfer e2e
  caught this); godown must be the doc's branch; btrim batch/serial; MRP ≤ 0 = no MRP.
- #6 docDate must fall in accYear (save 422 + post) STOCK_DOC_DATE_OUTSIDE_YEAR.
- #8 item-lookup alreadyOpened scoped to onDate's year + optional godownId/bucket params.
- #9 user decision "actor only": `actorOf` = login user over body userId. Company/branch NOT
  enforced — see [[token-scope-is-home-company-and-device-branch]].
- #10 nextAccountsRefno loops past refnos already on a live svh (cap 1000 → 409
  STOCK_REFNO_SERIES_EXHAUSTED); accounts-voucher duplicate printed numbers not addressed.

**Why:** Qt's notes 143 review; HIGH items were races and client-trusted quantities.
**How to apply:** `test/physical-save-and-post-http` fails on dev data (first count-sheet holding
has negative book qty → countedQty < 0 → 400), not code. No STOCK_* rows in Error_Messages.json
(none ever had them). Related: [[stock-ledger-cancel-leaves-forward-row]].
