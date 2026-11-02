---
name: receipt-design
description: "Receipt voucher (menu 99) design rev 3, 2026-09-14: share/receipt/ — five live-vs-spec divergences, what a receipt writes, the rev-2/3 additions (credits in-receipt, PDC own voucher, other-ledgers band + av_role, approval, PPD slabs, beat/customer), R1-R18 all taken, one phase, notes (7)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 152e1bb6-b02d-4445-8ee2-a03b779d2771
  modified: 2026-09-14T05:24:51.351Z
---

**Folder:** `/home/vk/Shared/Prathap/receipt/` — README, `receipt_flow.md`
(spec), `plan-backend-receipt.md`, `receipt_ui_mockup.py/.png`, all at
**REVISION 3** (2026-09-14). In `notes.txt` as **(7)**, superseding (6).
Follows [[opening-balance-concept]] — a receipt is what an OPENING bill exists
for. No Qt code; user said "do not write code" and it still applies.

**How the user works on designs (worth remembering):** wants the PNG FIRST,
decides on the picture, THEN the docs ("write png first ill decide"); asks
for comparisons against references (3.0 screen, Tally, GoFrugal) and folds
the answers in; prefers ONE phase over sequencing ("dont post any phase
2,3"); answers decision lists tersely ("skip 3, 4 … all other defaults").

**Rev 2 (3.0 + Tally) and rev 3 (GoFrugal) added:** credits the party holds
applied INSIDE the receipt (R4 reversed → ADVANCE_ADJUST/NOTE_ADJUST pairs);
a post-dated cheque is ITS OWN voucher dated the cheque, `avh_against_voucher_id`
→ the receipt, bills settle on maturity via `abj_is_post_dated` exclusion in
`fn_abl_recompute` + daily `fn_abl_regularise_pdc` (Tally Ctrl+T; 3.0's
"Cheque" column = `pdcHeld`); the OTHER LEDGERS band — TDS_RECEIVABLE,
BANK_CHARGES, DISCOUNT_ALLOWED, WRITE_OFF, CLAIMS_ALLOWED, TCS_PAYABLE,
SURCHARGE_RECOVERED, INTEREST_INCOME — one `acc_vouchers` leg each with NEW
column `av_role`; identity RECEIVED + DEDUCTIONS + CREDITS = ALLOCATED + ON
ACCOUNT + ADDITIONS, **bank charges are a leg split, outside it**; due-date
sort; header-only edit on POSTED; **NO approval** — withdrawn 2026-09-15,
DRAFT → POSTED → CANCELLED, Post runs straight off the draft
([[api-house-conventions]] rule 4); PPD
slabs `accounts.ppd_slabs` SEED the discount (server returns `ppdSuggested`,
never re-seeds at post); `accounts.tcs_basis` = RECEIPT; party picked as a
CUSTOMER (dropdown 39 + new optional `iarea_id`, Beat = dropdown 13 =
sales.area_master); the picked customer id IS the ledger id — no bridge (R9
taken); last-receipts + pending-cheques panels; `avh_draft_lines jsonb` for a
draft's other-lines; settings keys lowercase dotted (`accounts.*`).

**ONE phase by decision.** Out: Received Cheques screen (51) + ON_CLEARING,
Payment (100), Collection (187/188), batch post (dropped), bank
reconciliation (skipped), day-close (skipped).

**Five live-vs-spec divergences (verified 2026-09-14, live wins):**
1. `acc_bill_balance` IS partitioned by `abl_acc_year` (09 says "NOT
   partitioned, deliberately") — every bill ref travels with its year
   (`abj_bill_acc_year` NOT NULL live, `av_doc_acc_year`).
2. `acc_pdc_register` IS partitioned (vouchers.sql says not).
3. `fn_abl_recompute` / `tr_abj_refresh_balance` from 09 NOT deployed —
   `abl_alloc_amount` never moves; no trigger/function on ANY accounts table.
4. TWO cheque tables: `acc_voucher_cheques` (03, Postgres enums, no FK to
   header live) vs `acc_pdc_register` (lifecycle + `apd_posting_mode`);
   `fk_abj_cheque` points at the OLD one. DECISION R1 → use pdc_register,
   repoint the FK, loosen `ck_apd_dates` (refuses a cheque dated before receipt).
5. `acc_ledger_map` has ZERO rows — DISCOUNT_ALLOWED / WRITE_OFF unmapped.
Also: no Receipt voucher type (only Bil=3, ARc=5 affect accounts); `acc_vouchers`
has 0 rows ever — the receipt is the FIRST writer of legs.

**What a receipt writes (flow §4):** header (Rct type, DRAFT unnumbered →
POSTED numbered via acc_voucher_seq) · legs: one DR per instrument + ONE CR
party (+ DR Discount Allowed / CR party) · `acc_tender_detail` rows with the
SAME td* keys the sale bill sends, `td_src_doc_type='RECEIPT'`, `td_dr_cr='DR'`
· `acc_bill_adjustment` ALLOCATION rows, `abj_dr_cr='CR'`, instant money one
MIXED row per bill, each cheque its own rows (bounce reverses exactly that),
DISCOUNT/WRITEOFF rows · `acc_pdc_register` row per cheque (every cheque, not
just post-dated) · unallocated remainder = ONE `acc_bill_balance` ADVANCE CR
bill, src RECEIPT (the sale bill's adjust panel can spend it).

**Rules:** RECEIVED = ALLOCATED + ON ACCOUNT (discount is not money); per-bill
alloc ≤ pending under `FOR UPDATE`; POSTED never edited — cancel = reversal
voucher + negative abj rows; DRAFT keeps NO allocations (R10); header totals
DERIVED by a new trigger `tr_av_refresh_totals` (fixes notes (4)) and POSTED
must be set LAST in the transaction or `ck_avh_balanced` passes vacuously.

**Decisions R1-R18, ALL TAKEN** (flow §10 / plan §11): R1 pdc_register only;
R2 PDC own voucher + `accounts.pdc_posting_mode` default ON_RECEIPT (ON_CLEARING
not in this plan); R3 header-only edit on POSTED; R4 credits applied in-receipt
(YES); R5 customer picker, any ledger with empty panel; R6 one salesman,
mandatory by setting; R7 remainder = ADVANCE bill; R8 other-ledgers band;
R9 `cus_id` IS `led_id` (no bridge — [[api-house-conventions]]) (taken); R10 draft allocations nowhere, other-lines in
`avh_draft_lines`; R11 no unreferenced on-account; R12 due-date sort;
R13 collection = own screen; R14 approval RE-TAKEN = none; R15 tcs_basis RECEIPT; R16 ppd
slabs seed; R17 beat + customer + bridge; R18 one phase.

**Legacy references:** the REAL predecessor is
`/home/vk/Projects/NextLvl/3.0/pos/Accounts/customerreceipt.cpp` ("Bill wise
Receipt": tender grid, beat combo, TDS field, adjust-Advance, last-receipts
grid 813, pending-cheques grid 563, Ledger Statement, line-collection import,
edit with log, multi-company loop); EPOSGo `creditreceipt.cpp` is the older
one-tender version. Kept the gestures, dropped hard-coded ledgers
(100093/100002/100151/100162), the multi-company loop and the popup cheque
dialog. Tally F6 and GoFrugal receivables pages were the other two references.

**REV 5 (2026-09-16 evening) — the head of the screen, and the credit row.**
Mockups `receipt_ui_mockup_rev4.png` and `_rev5.png` in the share (rev 3 kept).
· TOP FRAME is now three panels — **PARTY DETAIL | LAST RECEIPTS | CHEQUES
  STILL OUT** (3.0's FCustomer). PARTY DETAIL holds Beat + Party and six facts
  (group / terms / TDS-TCS, outstanding / overdue / credits held); the DOCUMENT
  fields sit in a slim strip ABOVE it, because they are not the party's.
· **CREDITS ARE LISTED FIRST**, green-tinted across the whole row, and chipped
  **CREDIT / ADVANCE / CR NOTE** — never the raw billType. This mattered:
  Deepan's credit is `abl_bill_type='OPENING'`, the SAME word his nine DR bills
  carry, so both sides chipped "OPENING" and nothing said which was owed and
  which was held. Green = held, blue = owed, in the chip palette.
· **Bill amount** column added beside Pending / Held (both bills and credits —
  the server was already sending `billAmount` on both). "7,490 pending" means
  one thing on a 7,500 bill and another on a 75,000 one.
· The cheque dialog is **optional, on F4** — `apd_instrument_no` and
  `apd_instrument_date` are the only NOT NULLs and `SaveReceiptChequeDto`
  requires nothing, so the row's Ref / Instr. date / Bank are the mandatory
  fields and the popup carries extras only.
· The F8 list borrows the quotation list's own `TxnRowDelegate` (status badges
  + cancelled rows struck through), plus server-side search, prev/next paging
  and "showing X to Y of Z".
· Row routing flipped with the credits: `isBillRow`, `billIndex`,
  `creditIndex`. `billRowForRef()` returns a MODEL index into m_bills — it
  stopped being the same number as the grid row.

**CLIENT BUGS FOUND AND FIXED 2026-09-16** (all were live-server shape
mismatches, none caught by compiling):
1. **dropdown 54** — `buildFilterParamJson()` sends `{}` when no filter param
   is set, so `iarea_id` was never substituted and Postgres got the literal
   word ("invalid input syntax for type uuid"). Choosing no beat returned NO
   customers. Always send it, empty, BEFORE `setDropdownId()`.
2. **`/receipts/create` answers `data.HEADER.avhVoucherId`**, not
   `data.avhVoucherId` — same envelope as `/get`. Read one level too shallow,
   the id came back empty and `/post` then reported "avhVoucherId must be a
   valid UUID", pointing at Post while the fault was in the create. `sendPost()`
   now refuses without an id rather than letting the server say it.
3. **`otherLineBills[]`** sent `lineIndex` and no amount;
   `PostReceiptOtherLinePinDto` requires `lineNo` (1-based) + `amount`.
4. `avhDeviceType` must be one of PC/WEB/MOBILE/POS/DESKTOP — AppConfig sends
   PC, so fine, but "Desktop" is rejected.

**REV 4 (2026-09-16 pm) — TWO GRIDS, not four.** After seeing rev 3 run the
user called the UI bad and sent it back to the 3.0 reference
(`pos/Accounts/customerreceipt.ui`). Three merges, all coded and green:
· CREDITS fold into the BILLS grid — acc_bill_balance already holds both
  (`abl_dr_cr` DR/CR), so one list, a type chip, and on a CR row Receive means
  APPLY with Disc/W-off/R-off greyed;
· OTHER LEDGERS fold into the INSTRUMENTS grid — a row is an INSTRUMENT or a
  ROLE, hidden `Kind` column T/R, the Type combo lists tenders then roles, and
  picking the other sort CONVERTS the row (it moves between m_tenders and
  m_otherLines);
· 3.0's `+ / Default / −` buttons kept beside the tender table.
**Presentation only** — the payload still splits into tenders[] and
otherLines[], both tables and their NOT NULLs untouched, nothing to re-plan
with Prathap. Mockup: `receipt_ui_mockup_rev4.py/.png` (rev 3 kept beside it).
Delegates went 4 → 2. Alt+L is gone (a role row is '+' then a pick).
Known gap: a FREE ledger line (role blank) is not offerable in the merged grid
— it needs a picker column; 3.0 had none at all.

**A NexTable trap worth remembering:** `DisplayFormat::Currency` formats
unconditionally, so an empty QVariant paints as "0.00" (only `Number` blanks a
zero). Forty rows of 0.00 hide the three figures that matter. The receipt
writes an empty `QTableWidgetItem` directly for a zero money cell rather than
changing shared behaviour every grid depends on.

**Dropdown 54 bug, fixed:** `buildFilterParamJson()` returns `{}` when no
filter param is set, so the runner never substituted `iarea_id` and Postgres
got the literal word — *"invalid input syntax for type uuid: iarea_id"*, i.e.
choosing no beat returned NO customers. The client now always sends
`iarea_id`, empty when no beat, set BEFORE `setDropdownId()`. Verified live.
The user asked for `:area_id` naming instead; the real defect is that the
runner leaves UNSUPPLIED placeholders in the SQL — a server-side fix that would
touch every dropdown and grid. Prathap's call, not needed for this screen.

**BUILD STATUS (2026-09-16 — THE SCREEN IS CODED).** Design done; backend
LIVE (all 8 `/receipts/*` routes answer; `rct00001` posted). Client, built and
tested offscreen against 192.168.0.106:
`receipt_entry.h/.cpp/.ui` (NexDialog, QStackedWidget Entry/List), the four
delegates on a shared `receipt_delegate_base`, `receipt_cheque_dialog`,
menu 99 registered in `form_accounts_menu.h`, `tests/receipt/run.sh` (37
checks, all green). `TenderMasterRow` moved to
`src/shared/txn/tender_master_row.h` (+ namespace `TenderMasters`), sale bill
forwards to it. Five `accounts.*` settings added to `AppSession`.

**ui_tables 32 + 33 REGISTERED 2026-09-16** via
`POST /ui-table-masters/create` (`uiTblColumns` + `replaceColumns:true`; pass
`uiTblId` to update): **32 RECEIPT - BILLS** (17 cols, 13-16 hidden, 5 focus
stops) and **33 RECEIPT - TENDERS** (18 cols, 11-17 hidden, 9 focus stops),
widths as percentages summing to 100 taken from the rev 4 mockup's pixel
proportions. 34/35 were never used — the merge left only two grids.

**THE IDS WERE ALREADY ISSUED** (this was recorded as owed and was wrong):
grid **108** MAIN LIST - RECEIPTS, dropdown **54** CUSTOMERS BY AREA (param
`iarea_id`), menu **99** Receipt. Only ui_tables **32-35** (RECEIPT - BILLS /
CREDITS / TENDERS / OTHER LINES) are unregistered — the grids seed their own
headers so they work without them; widths just do not persist.

**THREE ARITHMETIC BUGS fixed in the 09-15 foundation files** (they made any
receipt with TDS, a discount or TCS unpostable):
1. engine docked the CASH by the settling deductions — the TDS was placed
   nowhere, so the bill was short AND the strip showed the TDS as surplus.
   Now: a DEDUCTION takes ROOM on the bills, an ADDITION takes CASH.
2. `ReceiptIdentity::compute` counted a deduction on the left only. It is on
   BOTH sides — flow §4.5 and the worked example (70,000 + 1,400 + 4,000 =
   70,750 + 4,650, the 1,400 inside the 70,750).
3. DISCOUNT/WRITE_OFF/ROUND_OFF are MIRRORS of bill columns and were being
   reserved twice. One predicate now: `ReceiptRoles::isMirroredByBillColumn`
   in receipt_entity.h, used by both engine and identity.

**Known gaps in the coded screen:** no TCS / TCS-pending columns on the bills
grid (the column enum has none — adding them is an enum + ui_table change);
print after post waits on the shared `PrintRunner`; ledger-statement button
hidden until that screen exists.

**Two findings open with Prathap:** (a) `avh_total_debit/credit` stamped 0 →
`ck_avh_balanced` vacuous ([[voucher-balanced-check-vacuous]]), fix = the
derive-from-legs trigger §2.4; (b) the service recomputes alloc from
adjustment rows, so every OPENING bill is restated on its first receipt.

**Still owed after this:** Received Cheques (menu 51) + ON_CLEARING,
Payment (100), Collection (187/188), Journal (103), ledger statement view
(DECISION 8), the receipt list grid id, dropdown id for CUSTOMERS BY AREA,
and the Qt screen itself. Next design the user is likely to ask for: Received
Cheques or Payment.
