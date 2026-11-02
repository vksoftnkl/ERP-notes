# Receipt — what 3.0 has that menu 99 does not

**Revision 2, 2026-09-18.** Rev 1 (2026-09-17) listed 14 items read off
`3.0/pos/Accounts/customerreceipt.{cpp,ui}` (1,946 + 2,287 lines). This pass
re-verified every one against `src/modules/accounts/receipt/` as it stands
after the 11-item rework, and swept the 3.0 delegates
(`customerreceiptdelegate.h`, `Delegate/custrcttndrdelegate.h`) — which rev 1
did not read, and which is where eight further items were hiding.

Nothing here is guessed from the mockups: every line names the 3.0 widget,
slot or column constant it came from.

**Score: 10 of rev 1's 14 still open, 1 downgraded, 3 were always deliberate.
8 new.**

---

## A. Blocking — an operator does this every day and cannot

**A1. Previous / next voucher.** `tBtnLeft` / `tBtnRight`, Ctrl+PgUp and
Ctrl+PgDown, calling `DBFunctions::prevTransId` / `nextTransId`. Walking the
day's receipts one key at a time is how a collection run is checked. Every
reopen of ours is a round trip through F8 and a search. **Still open — needs a
`/receipts/adjacent` route, or an ordered id list the client can walk.**

**A2. Save & New (F5) and Save & Print (F6) as separate verbs.** The two
endings of a receipt are different: the counter takes the next customer, the
back office prints and files. We have one "Post & Print" and a "Save draft",
and after a post the screen stays on the document. **Still open.** Note F5 is
now taken on our screen (reload open items), so this needs a key decision, not
just a handler.

**A3. Clear (F7).** `btnClearClicked` empties the form without closing it.
Ours is F1 → New, which prompts on a dirty document. **Still open.**

**A4. Auto-adjust as a MODE, not a one-shot.** `chkAdjust` +
`comboAdjustMode` → `adjustBills()`: with the box ticked, EVERY change to the
received amount re-spreads the allocation down the bills, and unticking it
takes the allocation back off again. Ours is Alt+A, once, and a later change
to a tender amount leaves the old allocation standing. **Still open — the
single highest-value item on this list after A1.**

**A5. NEW — the deposit account is not choosable per row.**
`custrcttndrdelegate` carries `TblMode::LgrCode` / `LgrName` and emits
`payAccChanged(row, LgrCode, LgrName, CompId)`; the tender master's
`edit_ledger` flag says whether that row may override the ledger the money
lands in. Ours takes `tnd_ledger_id` from the master into hidden column
`TenderLedger` and never offers it. **With more than one bank account this is
blocking: an operator cannot say which bank a cheque is banked into.** The DTO
already carries the field — this is a column, a delegate case and the
`edit_ledger` flag on the tender master.

---

## B. Real gaps — worked around today, badly

**B6. Payment history for one bill.** `tBHistory` → `PaymentHistory` on the
selected bill: every receipt that has touched it, with dates and amounts. The
answer to "we paid that in March", and we have nowhere to look it up from the
receipt screen. **Still open.**

**B7. Card detail capture — wider than rev 1 said.** `setCardDetail` fills
FOUR columns (`CardBank`, `CardNo`, `CardDate`, `CardRef`) and `PayTypeChanged`
seeds `SPerc` from the tender master's `surcharge_perc`, computing `SAmt` from
it. Our tender grid has an Mdr column and the DTO carries `tdCardLast4` /
`tdAuthCode`, but nothing on the screen collects any of it and no surcharge is
ever seeded. A card receipt is keyed as an amount and a reference.
**Still open.**

**B8. WhatsApp the receipt.** `tBtnWhatsapp` → `SendWhatsApp(2, 8, …)`. Waits
on the printing module either way. **Still open.**

**B9. Audit log.** `tBtnLog` — who touched this voucher and when. The data
exists (`avh_created_by` / `avh_modified_by` / `txn_status_log`); no client
surface. **Still open.** Note this is NOT the same as the edit reason, which we
already have — see the closed list below.

**B10. NEW — duplicate-receipt guard.** `checkDuplicateAmount()` (flag 10)
fires on every tender amount edit: it asks the server whether this party
already has a receipt for this amount on this date, and if so asks the operator
to confirm. On a beat run, where the same round figure is collected from the
same customer twice, this is the only thing standing between a re-key and a
double receipt. **We have nothing equivalent.** Needs one server route; the
client side is one call on the Amount cell.

**B11. NEW — the discount cell takes a PERCENTAGE.** `cellChanged()` splits
the typed text on `'%'`, and "2%" becomes 2% of the bill amount. Ours takes
currency only, seeded from `ppdSuggested`. Small, and the single most-used
keystroke on the bills grid for anybody who settles on terms.

**B12. NEW — profit left after the settlement discount.** `TblAdj::BillPr` and
`TblAdj::NetPr` carry `sale_profit` and `sale_profit − discount`, recomputed on
every discount edit. It is informational in 3.0 (no warning fires), but it is
the figure that tells the operator whether the discount they are about to give
costs the margin. **Ours has no profit column and the open-items payload
carries no such field** — this one needs the backend before the client.

---

## C. Present in 3.0's party band and totals, missing from ours

**C13. Place.** 3.0's `lEPlace`, from `ledger_place`. We show Beat, mobile and
address (added 2026-09-17); Place is still absent though `led_city` is already
in the payload `loadPartyContact()` fetches. **Still open — cheapest item here.**

**C14. Total Balance.** `lETotalBalance` — one figure for the party across
everything, beside the bill-wise outstanding. Ours shows OUTSTANDING / OVERDUE
/ CREDITS HELD, which is the bill-wise view only. **Still open.**

**C15. NEW — Balance after this payment.** `lEBalanceAfterPay`, computed in
`calcTotal()` as `totalBalance − amount − discount`. It answers the question
the customer actually asks at the counter — "so what do I still owe?" — and
our identity strip does not: the strip is about whether the RECEIPT balances,
not about what the PARTY is left owing.

**C16. NEW — Total of cheques still out.** `lETotalChqBalance`, summed in
`comboCustomerActivated` from each bill's `cheque_amount`. We show the
pending-cheques panel row by row with no total under it, and the bills grid's
PdcHeld column per row with no sum.

**C17. NEW — the customer's own reference, per bill.** `TblAdj::UserRef` from
`user_refno`, displayed on the bill row and sent back on the allocation
(`bill.usrRef`). It is how an operator matches a customer's remittance advice
to our bills. Our `OpenBill` has `docRefno` (OUR reference) only; `usrRefno`
exists on the receipt HEADER but not per bill.

---

## D. Closed, or deliberately not carried over

**D18. Edit reason — ALREADY MATCHED, was never on rev 1.** 3.0's
`lELogRemarks` forces a typed reason when `global.editingLog` is on and stores
it as `ilog_remarks`. `ReceiptEntry::editHeader()` does exactly this: a
required `AppMessage::askText` reason, sent as `editRemark`. Recorded so the
next pass does not raise it.

**D19. DOWNGRADED — apply an existing advance (rev 1's B5).** Rev 1 read
`pBtnAdvance` as "put the cursor on a bill and spend the party's advance on
it". It is not: `btnAdvanceClicked` returns immediately unless the row's
balance is `<= 0` — it works on the ADVANCE row, and opens a separate
`BillwiseAdvanceAdjust` dialog. Our rev-4 merge does the same job inline
(credits sit in the bills grid, Receive means APPLY on a CR row), with no
second dialog. **Not a gap; ours is the better shape.**

**D20. `chkAdjust`'s "adjust Advance" sub-mode** — superseded by the credits
band, which lists what the party holds explicitly instead of hiding it behind
a checkbox.

**D21. The collection queue** (`bindCollectionNos` / `processCollection`) —
opens a list of receipt numbers one after another for a beat run. Belongs to
the Dairy/milk-collection flow; revisit with that module.

**D22. `pBtnDelete`** — 3.0 deletes a receipt. Ours cancels it, which is the
correct accounting behaviour.

**D23. The multi-company loop** and its validation family (`compList` /
`advcompList`, "TDS not allowed with multiple companies", "ledger company and
invoice company differ", "advance across companies") — dropped by design. Most
of 3.0's 240-line `checkinputValues` is this and does not apply.

---

## Where ours is AHEAD of 3.0 — for the record

So the next pass does not "restore" something on purpose removed:

- **Bill grid.** Ours carries Bill amount, Paid, PdcHeld, After and a type chip
  that 3.0 splits across hidden columns or does not have at all. 3.0 has no
  credits in the grid — it needs `pBtnAdvance` and a second dialog.
- **Validation.** Every refusal names the row and puts the cursor on the
  offending cell before the message box (`Problem` + `focusProblem`). 3.0
  warns and focuses the table.
- **Post-dated cheques** are their own voucher with a maturity path. 3.0 has a
  `cheque_amount` column and no lifecycle.
- **The identity strip** (RECEIVED + DEDUCTIONS + CREDITS = ALLOCATED + ON
  ACCOUNT + ADDITIONS) has no 3.0 equivalent; 3.0 has Pending / Adjusted /
  Balance, which is the allocation only.
- **Roles** (TDS, bank charges, write-off, TCS…) are first-class rows with
  `av_role`. 3.0 has one hard-coded TDS box.
- **Cancel is a reversal**, not a delete.

---

## Shortcuts — 3.0's actual set, against ours

    3.0                                 ours
    F1  toggle focus bills <-> customer F1  New
    F2  Edit                            —   (Edit header on a posted receipt)
    F3  Delete                          F3  cancel, on the register
    F5  Save & New                      F5  reload open items
    F6  Save & Print                    —
    F7  Clear                           —
    F8  Show List                       F8  Show List          <-- matches
    F9  focus the tender table          F9  Ledger statement
    Ctrl+PgUp / PgDn  prev / next       —
    —                                   Ctrl+Enter  Post
    —                                   Alt+A  auto-allocate, Alt+T  add tender
    —                                   F4  cheque extras

Decided 2026-09-17 to keep ours rather than re-map, and only make every key
window-scoped. A2/A3 therefore need free keys, not 3.0's.
