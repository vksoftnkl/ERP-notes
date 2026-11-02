---
name: receipt-screen-revision
description: The 11-item receipt (menu 99) revision of 2026-09-17 — what changed, plus the ui_tables index-shift trap
metadata:
  type: project
---

Eleven items reworked on the receipt screen on 2026-09-17. The ones with a lasting rule:

**Validation now carries WHERE, not just what.** `ReceiptEntry::Problem { message, widget,
grid, row, column }`; `focusProblem()` puts the cursor on the offending cell BEFORE the
message box, so the box is dismissed onto the field that has to change. Every tender-row
refusal names its row number.

**Adding a column to a receipt grid takes FOUR edits, not two.** The enum and the .ui are the
obvious pair; the two that bite are `BillCols::formats()` — `NexTable::setDisplayFormats()`
calls `setColumnCount(m_DisplayFormats.size())`, so a column with no format entry is silently
dropped from the grid at runtime — and the ui_tables registration below. Also: both column
enums end in `ColumnCount  = 18`, so an unscoped renumber hits `TenderCols` too and gives the
instruments grid a count its .ui does not have.

**Adding a column mid-grid needs the ui_tables layout re-registered.** `fixed.ui_table_columns`
is keyed by COLUMN NUMBER, so inserting "Paid" at index 6 silently applied every stored
width and header to the column next door — the new column simply did not appear and "Due"
vanished. Fixed by POSTing the full 19-column array to `/ui-table-masters/create` with
`uiTblId: 32`, reusing each existing `uiTblClmId` so widths survived, and paying for the new
6.2% out of Note's 15.8%. **Any future column on a registered grid needs the same step.**
See [[nextable-grid-traps]].

**Grid 108 has the same bare-params trap as grid 109** ([[cheques-screen-built]]): it names
`iavh_status` as a bare identifier, so a register that omits it is a 400 on the whole call.
Sent as `""` through `buildExtraParams`, NOT a Combo filter — a Combo's first option means
"no filter" and sends nothing, which is the one thing this grid cannot survive.

Also: menu 99 now opens the **register** (`TxnDocTypes::receipt()` in
`src/modules/accounts/common/`, grid 108, F3 = cancel not delete) or the entry screen per
`txnEntryFirst()`, like quotation and sale bill; mobile + address come from
`/account-ledger-masters/get?ledId=` because open-items' party object carries neither; the
TERMS field's one useful fact (NOT bill-by-bill) moved to the bills grid's own hint; posting
with money left over now confirms by name; the tender grid is pinned to two rows; the cheque
Bank column is dropdown 46 (`fixed.bank_master`, 166 banks) but stays EDITABLE because
`td_bank_name` is a varchar, not an FK; F9 was advertised in the footer and never bound.

**The 3.0 gap list is `/home/vk/Shared/Prathap/receipt/gap-vs-3.0.md`** — now at **REVISION 2
(2026-09-18)**, graded A (blocking) to D (deliberately dropped), plus a "where ours is AHEAD"
section so a later pass does not restore something removed on purpose.

Rev 2 re-verified all 14 of rev 1's items against the reworked screen and swept the 3.0
**delegates** — `customerreceiptdelegate.h` (TblAdj) and `Delegate/custrcttndrdelegate.h`
(TblMode) — which rev 1 never read and where **eight more items** were hiding:
per-row deposit-account override (`payAccChanged`, gated by the tender master's
`edit_ledger` — blocking with more than one bank account), a duplicate-receipt guard
(`checkDuplicateAmount`, flag 10), percentage entry in the discount cell ("2%"),
bill profit / net-profit columns (`sale_profit`, needs backend), balance-after-payment,
total-cheques-still-out, and the customer's own reference PER BILL (`user_refno`).
Two corrections: rev 1's **B5 is NOT a gap** (3.0's `pBtnAdvance` only fires on a CREDIT
row and opens a second dialog — our merged credits band does it inline), and the **edit
reason is already matched** (`editHeader()`'s required `editRemark` == 3.0's
`lELogRemarks`). Score: 10 of 14 still open, 1 downgraded, 8 new.
The biggest owed item is prev/next voucher (Ctrl+PgUp/PgDown), which needs a backend route.
3.0's real keys: F2 edit, F3 delete, F5 save&new, F6 save&print, F7 clear, F8 show list,
F1 toggle focus, F9 focus tender. **F8→list already matches**; the user chose 2026-09-17 to
keep our F1/F9 rather than re-map, and only make every key window-scoped.


## 2026-09-18 run-through — four fixes (commit 52232db)

**A saved DRAFT could not be deleted from anywhere.** `/receipts/cancel` refuses one and
the register greyed F3 with "there is no route for it" — true when written. **`/receipts/delete`
exists now** (`ApiEndpoints::Receipts::Delete`, same four keys). Entry screen has a
`btnDeleteDraft` beside Cancel; the two are NEVER both live. `receiptRowPolicy` no longer
blocks DRAFT, and `c.onDelete` routes by `key.status` — DRAFT → `deleteDraftReceipt`,
POSTED → `cancelReceipt`. Gating trap: `setEntryEnabled()` is what enables the button, so
`saveDraft()`'s success handler must call it — `setStatusPill()` only paints the pill.

**`/receipts/post` answers `data.HEADER.avhVoucherRefno`, not `data.avhVoucherRefno`.**
`sendPost()` read one level too shallow, so a posted receipt showed a BLANK number and said
"Receipt  posted." — indistinguishable from a failed post, which is what "cant post
directly" turned out to mean. `saveDraft()` had already learnt this for `avhVoucherId`;
**assume every /receipts/* route uses the {data:{header,…}} envelope.**

**The list is now a PICKER**, like quotation and the sale bill:
`TxnMainView::openPicker(TxnDocTypes::receipt(), this)` + `picked` → `loadReceipt`. The
second QStackedWidget page, its grid, paging buttons, `setupListPage()` and
`applyListParams()` are all gone (~250 lines). `stkPages` is kept with one page so the
`setCurrentWidget(pageEntry)` call sites stay meaningful.

**The yellow banner moved** from a full-width strip above the party frame to a small chip
on the title row; `showBanner(shortText, detail)` puts the sentence on the tooltip.

**Tests:** `tests/receipt/run-post-new.sh` drives both reports through the screen's own
slots (`QMetaObject::invokeMethod` on the private slots) and posts/deletes real documents.
The modal-closer must ACCEPT, not reject — Post asks "keep it on account?" first.

**BACKEND DEFECT found while cleaning up — see notes (26):** cancelling a receipt that
carried a post-dated cheque reverses ONLY the receipt; the PDC child voucher stays POSTED
with live legs, and `/receipts/cancel` on the child refuses ("cancel the receipt it belongs
to and both are reversed together" — which is what did not happen), so it is unreachable.
9 orphaned vouchers holding 5,400 on the box, 5 predating my testing.


## 2026-09-18, second round — the five unblocked features (commit adeaaa3)

`/receipts/adjacent` → **Ctrl+PgUp / Ctrl+PgDown** (`stepVoucher()`); the server
orders it, the end of the walk is an AppMessage not a dead key, and the whole
VoucherKey comes back so another branch's receipt opens.
`/receipts/duplicate-check` → the guard on the tender Amount cell, asked on the
TOTAL received (a split tender would otherwise cry wolf), raised as the chip not
a modal. `party-context.summary` → **TOTAL BALANCE / CHEQUES OUT / AFTER THIS
RECEIPT** on the party band. Bills grid **19 → 24 columns**: Their ref
(`usrRefno`), TCS due, Profit, Net profit, + hidden TcsAmount; ui_table 32
re-registered.

**Three traps worth keeping:**
1. **A banner raised from a paint function gets re-raised on every repaint.**
   `paintPartySummary()` now runs from `refreshIdentity()` (AFTER THIS RECEIPT
   must move with the cells), so the not-bill-by-bill notice it used to raise
   stamped over the duplicate warning every keystroke. A one-shot notice belongs
   where the data LOADS, not where it paints.
2. **A "don't ask twice" memo must be cleared on every new/loaded document** —
   `m_lastDuplicateAsked` was not, so the second receipt for the same figure was
   silently skipped. That is the exact case the guard exists for.
3. **`/ui-table-masters/create` column keys are `uiTblClmColumnVisibility` /
   `ColumnFocus` / `ColumnNecessity`** (not `uiTblClmVisibility`/`Focus`/
   `Necessary`), and `uiTblClmNo` is a STRING. Wrong names = a 400 per column.

**Every test runner now runs `ninja` first** (`tests/{receipt,ledger,cheques}/*.sh`).
This cost two debugging cycles: they link ninja's own `.o` files, so an
uncompiled edit silently tests the PREVIOUS binary and the failure reads as a bug
in the fix. See [[qt-offscreen-test-harness]].

`tests/receipt/run-features.sh` — 23 live checks, posts and cancels its own
receipts.


## Draft allocations — the ask, notes (30), 2026-09-18

R10 (a DRAFT keeps no allocations) means a reopened draft has to be re-settled
bill by bill. The user asked whether that is the only way; it is today —
`/receipts/create` takes header + tenders + otherLines and no allocations at
all. **Alt+A is the practical answer** for the common case: it spreads the
tender total down the bills in the server's order.

**The proper fix is small and is now asked for.** `avh_draft_lines` (jsonb) is
already live on 342 headers carrying `{"cheques": {}, "otherLines": []}`, and
`/receipts/get` ALREADY returns `allocations[]` / `creditsApplied[]` keys —
they are just always empty for a draft. So the ask is only "put the allocation
in the same box and hand it back".

**The constraint that must survive:** nothing about it may write
`acc_bill_adjustment` or move `abl_pending_amount`. R10's reason stands — a
draft locks nothing, so two operators can hold drafts against the same party.
The allocation is REMEMBERED, not applied.

**And the client owns the staleness**, deliberately: what comes back is a
suggestion, because somebody may have settled the same bill meanwhile. Prathap
was asked NOT to validate it on the way in or out — the screen re-reads
`/open-items` on reopen and will clamp each remembered figure to what the bill
can still take. Server-side validation would turn a recoverable load into a
refused one. **LANDED AND WIRED 2026-09-18 (commit 48b6a11).** Verified before wiring: two allocations
round-trip through `avh_draft_lines`, bills unchanged, zero `acc_bill_adjustment` rows.

Two things the client side needed that are easy to get wrong:
- **the draft memo sends the RAW `b.receive`, not the post-shaped allocation total.**
  `buildPostPayload()` sends receive + settling deductions + credits because the posting
  engine needs that; a memo must round-trip to the cell it came out of, or reopening
  inflates Receive by a TDS line the operator never typed.
- **the clamp** (`mergeDraftAllocations()`): each remembered figure is capped at what the
  bill can still take, MONEY FIRST — a discount is a decision about a balance that may no
  longer exist — and a bill that has since closed is dropped. Both cases are said in the
  chip; silently keying a smaller number than the operator left is the worst option.


## Full-coverage pass before Payment, 2026-09-18 (commit cec2442)

`tests/receipt/run-full.sh` — 40 checks over what the other three suites never
reached, asserting against the **DATABASE** rather than the reply that wrote it.
Four suites now: `run` (58), `run-post-new` (21), `run-features` (37+), `run-full`.

**IT FOUND A REAL ONE: any receipt with a settlement discount was unpostable.**
`DISCOUNT_ALLOWED` / `WRITE_OFF` / `ROUND_OFF` exist in memory because the
IDENTITY needs them (they are deductions), but they MIRROR the bills grid's own
columns, which already travel inside `allocations[].discount` / `.writeoff`.
Sending them as other-lines asks for the same money twice — and the server
refuses two different ways depending on `settlesBill`, so flipping the flag just
swaps one 400 for another. **The fix is not to send them at all**
(`ReceiptEntity::linesThatTravel()`), verified live first: with the discount only
on the allocation, the post writes the `DR Discount Allowed` leg itself.
`otherLineBills` pins are numbered against the FILTERED list — `lineNo` is a
position in the array the create sent.

**Harness lessons worth keeping:**
- a required-reason prompt (`AppMessage::askText`) must be **answered**, not just
  accepted — an empty reason silently aborts cancel and amend. The modal closer
  fills any empty QLineEdit before accepting.
- the amend's reason PROMPT contains "restated in place"; the RESULT says "was
  restated". Waiting on the wrong one ran the DB assertions before the amend was
  sent, and the failures looked like the amend not working.
- a test must set `AppSession::setAllowPostedAmend(true)` — nothing in a test run
  loads the settings the app loads at login, so Correct Receipt is invisible.
- `apd_voucher_id` points at the PDC **child** voucher, not the parent receipt.
- the register's newest row is often an **unnumbered DRAFT**, so "did Prev load
  something" cannot be asserted on the refno box; use the discard button coming
  alive (it needs a saved voucher id).
