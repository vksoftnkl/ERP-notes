---
name: payment-screen-built
description: "Menu 100 Payment screen CODED 2026-09-29 in src/modules/accounts/payment — file set, shared-engine policy change, ui_tables 42/43, grid 123, live suite 44/0, VKPOS menu-100 grant"
metadata:
  node_type: memory
  type: project
  originSessionId: d1ee65aa-9018-42f1-a55a-36298f441368
  modified: 2026-09-29T05:23:31.335Z
---

**Built 2026-09-29, uncommitted** (user commits by hand). "Do as like receipt":
ReceiptEntry cloned, not the Voucher Register. Mockup = share/payment/payment_ui_mockup_rev2.png.

**Files** (`src/modules/accounts/payment/`): payment_entity.h (PaymentBillCols 20 cols,
PaymentTenderCols = receipt's 18 with 2 captions, PaymentRoles, PaymentParty +
computeTds, PaymentSummary/Totals, PaymentChequeBook), payment_{bill,tender}_delegate,
payment_cheque_dialog (book + auto leaf + favouring + A/c payee), payment_beneficiary_dialog,
payment_entry.{h,cpp,ui}. Registered `fm.reg(100, …TxnDocTypes::payment())` (grid 123);
`AppSession::paymentSalesmanMandatory()`; `ApiEndpoints::Payments`.

**Shared change:** `SettlementRoles` policy (isMirroredByBillColumn, isLegSplit) added to
receipt_entity.h; `ReceiptIdentity::compute` and `ReceiptAllocationEngine::allocate /
allocationAmountFor` take it, defaulting to `ReceiptRoles::policy()` — receipt unchanged.
TenderRow gained `beneficiary`, OtherLine `approvedBy` (sent only when set).

**Payment rules that differ:** bank charge is INSIDE tdAmount → BANK_CHARGES is an
ADDITION (no leg split); TDS seeded client-side with the server's exact gross-up
(net = Σamount − Σcharge; base = round2(net/(1−r)); threshold incl. tdsPaidThisYear),
never typed (server 409s disagreement); mirrors DISCOUNT_RECEIVED / WRITE_OFF / ROUND_OFF
never travel (400); role decides Dr/Cr + settles (cells closed); a cheque row never sends
tdRefNo (`tendersToSend()` strips the loaded leaf for amends); money ledger payee refused
on open-items `isMoneyLedger` (backstop only now). **Payee = dropdown 60 "PAYMENT PAYEES"**
(registered 2026-09-29): party ledgers only = group `acc_ledger_profile = 'Party'` OR
`led_ledger_type = 'PARTY'`, shared + company via `'icompany_id'` (always sent, text compare);
search flags on led_name / group_name / led_short (`dropdown_columns_filter` — without them
search returns 0). Dropdown 59 VOUCHER PARTIES can't be reused: it needs a register type.

**Box ids:** ui_table 42 "PAYMENT - BILLS", 43 "PAYMENT - TENDERS" (created via
/ui-table-masters/create 2026-09-29). Grid 123 = Prathap's.

**Tests:** tests/payment/run.sh — 70/0 on 2026-09-29. Posts one 1,020 NEFT (20 charge,
TDS 20.41) to E2E supplier 01a0e6f1-8fc0… and cancels it (pmt00260); saves/reopens/deletes
a cheque draft (no leaf taken). **VKPOS granted menu 100** (all but retender) that day via
full-replace /user-administration/create; before/after diff = only the menu-100 row.

**Prathap applied notes 61 + 62 on 2026-09-29 (same day).** Verified live: D2 rights
(PMT_RIGHT_VIEW), D1 renames reverted (creditsHeld / totalCredits — screen reads either),
A3 a draft's tender now returns cheque{chequeBookId, favouring, acPayee, bookNo, + receipt
nulls} — the screen keeps ONLY the three DTO keys (bookNo would 400 on re-save).
**/get has no isSeeded flag**: a draft's TDS_PAYABLE / BANK_CHARGES come back as plain lines,
so applyLoadedPayment marks them seeded (else the reseed doubles TDS → 409); a TDS line on a
party the screen does not compute TDS for stays typed. Loaded role lines carry ledgerId too —
cleared (role XOR ledger).
**A1 / A2 / A4 verified live 2026-09-29:** A2 — /payments/get|adjacent 404 on Rct/PmtV,
/receipts/get 404 on Pmt, /payments/create over a receipt draft 404 and leaves it Rct.
A1 — advance with TDS posts (pmt00366 API, pmt00370 via screen): onAccount = money + TDS
(1,000 + 20.41 → 1,020.41 advance), 26Q row 194C base 1,020.41; cancel reverses TDS + advance.
Client followed: ReceiptIdentity caps allocated deductions at the bills' room (overflow →
on account; shared with the receipt, whose server engine is the same). B1 — TDS base = tenders
less EVERY DR line (client too). B2 — typed TDS on an unflagged party is a 400, so
TDS_PAYABLE is no longer an offered role. A4 — code reads sup_cash_disc_perc within
sup_credit_days, but NOT provable live: the only supplier with terms (GOOGLE INDIA 2%/20d)
has no open bill. **E2 round-UP BUILT 2026-09-29:** Pay over a bill by < 1.00 (or a negative R/off, editor
min −0.99) → bill.roundOff NEGATIVE, After 0.00; seeded DR ROUND_OFF line (settles false,
an addition) TRAVELS; allocation amount = exact bill (+roundOff), roundoff 0 (server refuses
negative); draft memo sends the exact amount and a loaded DR ROUND_OFF stays manual.
paymentLinesThatTravel drops mirrors only when CR. **Trap:** ReceiptHeader::toDraftJson()
ran the receipt's linesThatTravel INSIDE it and dropped the DR ROUND_OFF → server "out by
0.40"; it now takes `linesFiltered` (payment passes true). Verified pmt00384 (70.40 on 70.00:
DR party 70, DR Round Off 0.40), cancelled. A posted round-up is not restored on amend
(posted /get has no otherLines) — the 0.40 shows as on account.

**Issued Cheques (52) Enter** routes by /vouchers/get rules: inRegister → VoucherRegisterEntry,
typeCode 'Pmt' → PaymentEntry::openFor, else a message. Received Cheques (51) already routed
RcpV → register, else Receipt. See [[payment-design]].

**C1 verified live 2026-09-29:** amend now CANCELs the old HELD leaf (not soft-delete),
reason "Amended into revision N", still listed on /cheque-books/get. Test: pmt00398 (cheque
94314094 → amended to cash). **NEW DEFECT from that fix:** assertIssuedChequesStillHeld
(payment-unwind.guards.ts:26-51) refuses any apd row not HELD, so the leaf the amend
CANCELLED blocks /payments/cancel ("Cheque 94314094 is CANCELLED … unwind on menu 52") and a
second amend (payment-amend.service.ts:125) forever. pmt00398 is stuck POSTED on .106 (70.00
cash to E2E Chq Supplier Z). Written up as share/accounts/notes-63-amended-payment-cannot-cancel.md.
ZT52-31409 is FINISHED; test book **ZT-PAY-01** (Kvb Current A/c, leaves 94320000-94320049)
created 2026-09-29 via /cheque-books/create for tests/payment. The round-up test (B3c) now pays
whatever bill Chq Supplier Z still owes (+0.40) — pmt00398 took its 70.00 bill.

**notes 63 FIXED + verified 2026-09-29:** pmt00492 (cheque 94320000 → amend → amend → cancel)
all 200, leaf stays CANCELLED "Amended into revision 1". Stop still blocks cancel (pmt00496,
leaf 94320002 STOPPED → chqbnc00078; cancel refused — by design, it stays POSTED, netted).
Prathap cancelled pmt00398 himself. **NEW defect, not yet written up:** /issued-cheques/stop
on a cheque whose payment went ON ACCOUNT (advance, no bill) is refused by the books check —
"open bills −760 vs ledger −770" (pmt00494, 10.00): the stop reverses one side of the ADVANCE
but not the other. Bill-allocated stop is fine. Re-confirmed on pmt00497 (0 abj rows carry the leaf; unwind() only reverses abj_cheque_id adjustments, never the PAYMENT_ADVANCE bill). **Written up as share/accounts/notes-64-stop-cheque-on-account.md**; /returned shares unwind(). Both test payments cancelled, party back to -770.
(The POSTED Pmt numbers right after each cancelled test payment are the cancels' reversal
vouchers, same series.)

**notes 64 FIXED + verified live 2026-09-29** (by API, scratch script; the rsync of his source
needs the user's password, so the fix was not read). Stop, all on account: pmt00584 → chqbnc00097,
ADVANCE closed. Returned, mixed (70 to a bill + 5 on account): pmt00585 → chqbnc00098, the bill
reopens and the ADVANCE closes. Advance SPENT (pmt00586 used by pmt00587): stop is a 409 that
names both documents. After the spender was cancelled, the stop goes through (chqbnc00099).
Supplier Z reconciles at −770 after every step. The three payments stay POSTED, netted by the ChqBnc.
Not run: /returned on a spent advance (it shares unwind()).

**Full retest 2026-09-29 (after notes 64):** payment suite 71/0; API route matrix 33/0 (update-header is PUT;
/adjacent must OMIT status, not send it empty); issued 14/0/1; receipt suites fail ONLY on "Deepan" having
no open bills (data, not code). Code review found 7 high client bugs — **ALL FIXED 2026-09-29**, suite now 92/0 (B3d-B3f live cases added):
(1) approver = the POSTING USER: OpenBill::toAllocationJson(amount, approver) sends writeoffApprovedBy
(receipt too), paymentLinesThatTravel(lines, approver) stamps BALANCES_WRITTEN_BACK; the narration banner
is gone. Server default limit 0 = every write-off needs one; client's own 0 used to mean "never".
(2) TDS base subtracts the bills' round-up. (3) NEW ReceiptAllocationEngine::allocationAmounts() — the
wire per bill, unpinned deduction capped at each SENT bill's room (pins first), largest-remainder split
like the server's distributeProRata; identity room counts only bills with settled > 0 (receipt too).
(4/8) captureTenderRow changes a line's role only when a PaymentRoles::all() entry is picked; the tender
delegate's combo shows an unknown name as itself. A fresh round-up replaces a loaded DR ROUND_OFF.
(5) loaded BANK_CHARGES seeded only if it equals chargeTotal(). (6/11) applyLoadedX resets
m_amending/baseline/btnPost text + m_summary, party context asked for posted docs too (both screens).
(7) ReceiptHeader::fromJson reads avhEmployeeId; restoreSalesman() asks /employee-masters/get?empId= for
the name (both screens); startNewDocument clears ddSalesman. Test harness must setUserId from login usrId.
Review 9 + 10 also FIXED same day (suite 100/0): canEditDocument() = (isEditable || m_amending) && !writeDenied
gates add/remove/default instrument + the F4 dialog (both screens); a typed negative R/off is refused unless
Pay is over the bill, and snapped to exactly the overpayment. Test runners now need -lQt6PdfWidgets -lQt6Pdf
(another session added QtPdf printing preview) — added to payment/receipt/issued run*.sh only.
Receipt live suites still can't exercise bills: "Deepan" has 0 DR bills (only a 100 advance, rct00287).
Stray un-deleted DRAFT 01a0ebaf-132b… (70.40, Supplier Z) from an 11:11 run.
