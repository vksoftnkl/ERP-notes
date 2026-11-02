---
name: voucher-register-screen
description: Qt Voucher Register (menus 101-104,163,259-262) — coded 2026-09-26 in src/modules/accounts/register; file set, contract facts, what is tested and what is still owed
metadata:
  type: project
---

**Coded 2026-09-26** against the LIVE `/vouchers/*` (12 routes, deployed ~11:40 that day) from share/accounts/voucher_register.md §9. It builds with no warnings.

**Files**
- `src/modules/accounts/register/`: entity.h, ledger_picker, line_delegate (holds both `VoucherLineDelegate` and `VoucherBillDelegate`, on `ReceiptDelegateBase`), entry.{h,cpp,ui}.
- `ApiEndpoints::Vouchers` in `accounts_end_points.h`.
- `TxnDocTypes::voucher(code)` / `voucherRegister()` in `txn_doc_types_accounts`.
- Nine registrations in `form_accounts_menu.h`: 103 Jrl, 104 Con, 101 DrN, 102 CrN, 163 PurA, 259 SalA, 260 RcpV, 261 PmtV, 262 register.
- `tests/register/{run.sh,test_register.cpp}`.

**Contract facts (from vouchers-api.types.ts + Swagger)**
- Keys are bare camelCase: `companyId`, `branchId`, `accYear`, `voucherId`. NOT the receipt's avh-prefixed ones.
- `/validate` is ALWAYS a 200: `data.{ok, derived, refusals, warnings}`. `/post` answers 422 with every refusal.
- A leg settles bills on the OPPOSITE side, so `/open-bills` side = `opposite(leg side)`.
- POS: send `posStcd` only when the operator picked one. The server defaults it and echoes `gst.placeOfSupply`.
- A DRAFT's typed payload comes back in `data.draft`. Typed lines regain their names from `/validate` `legs[].lineRowNo`.
- Box ids: grid 117 (list, binds `icompany_id`/`ibranch_id`/`iacc_year`/`iuser_id`/`itype_code`/dates; `itype_code` always sent), grid 118 (exceptions, unused), ui_table 39 (legs, 12 cols), ui_table 40 (bill-wise, 9 cols), dropdown 21 (states).

**Revised 2026-09-26 (user's five asks):**
- Ledger cell = NexTable popup grid **119** "POPUP - VOUCHER LEDGERS"; party = NexDropdownSingle **59** "VOUCHER PARTIES". I registered both.
  - Their SQL mirrors /vouchers/ledger-pick: share/grids/voucher_ledger_picker.sql.
  - Tokens `icompany_id`, `itype_code`, `iside` are always sent.
  - The route picker class was deleted.
- `/ledger-balance` shows under the grid for the row under the cursor, and as "balance" in the party facts line. Cached per ledger + date.
- To/By display comes from setting `accounts.drcr_display` (DR_CR | TO_BY; By = Dr, To = Cr).
  - `AppSession::showToBy()` reads it.
  - The definition row is share/41_drcr_display_setting.sql: RUN on the box 2026-09-26. The effective API returns it, still at the default DR_CR.
- The rule strip divides its width equally.
- A quick strip, the sale bill's idiom: prev/next/new/list/copy.
  - Prev/Next use `GET /vouchers/adjacent` (notes 52, live 2026-09-26).
    - `voucherId` is optional: prev with none gives the newest. `typeCode` is optional.
    - Rev is never a stop; `voucher: null` at either end.
    - The grid-117 walk was removed.
  - The deleted-bill-still-OPEN observation was test data; the user dropped it. Not an item.
  - Ctrl+PgUp / Ctrl+PgDn.

**Tested (live):**
- `run.sh smoke` 24/24; `run.sh features` 7/7.
- `run.sh live` 44/0/1: §11.2 all 15 checks, the PNG example posted and cancelled.
  - Uses Prathap's E2E ledgers: supplier `01a0dc5d-604f…` (194C), Housekeeping and Stationery, GST 18% / 12%.
  - The skip: the other-user variants of #1 need a second login.
- `run.sh cancel <refnos>` cancels test posts through the screen.
- Test posts pua00016–19, all cancelled, rev00049–52, net 0.
- Harness traps:
  - `askText` with presets opens on "Choose…", so pick index 1.
  - After the server-away case, wait until an API call answers before the next edit.
- VKPOS now HAS all 9 menus.

**Owed**
- Then the §11.2 live cases: these need ZT ledgers (expense, supplier with TDS 194C, GST ledger mapping).
- Registration nits: ui_table 39 visible widths sum to 108 and grid 117's to 123 (should be 100); every ui_table 39 focus flag is false.
- Not built yet: the GST card's ITC combo per rate row, and the exceptions grid 118.


**Revised again 2026-09-26 (user):**
- The quick strip and Copy are gone. Prev/Next are footer QPushButtons.
- frameTypes shows only on the register (262).
- frameLower is capped at 190 px.
- FIXED a dangling `TypedLine &` in `onLedgerPicked`: `ensureTrailingLine()` appended to `m_lines`, and the stale `l.drCr` read sent the cursor to Credit. Test: `run.sh pickcursor`.
- **Receipt/Payment Voucher → MANY (user chose both).**
  - Asked in share/accounts/notes-53: the type rows, a per-line DEMAND test, and per-party TDS on PmtV.
  - The client is ready. The bill-wise card follows the selected party line on any MANY type; draft allocations are restored per line; the TDS column shows on MANY+DEDUCT.
- The user had the .ui open in Designer (`voucher_register_entry.ui.autosave` exists). A save there would revert the layout.
- pua00020 ("AGL", 14:21) was the user's own manual post, not a test.
- Suites: smoke 24/24, features 7/7, pickcursor 3/3, live 45/0/1. Test posts pua00016-19 and 21 are all cancelled.

- **Notes 53 LIVE (2026-09-26, migration 20260926150000).** RcpV and PmtV are MANY.
  - DEMAND works per line.
  - On-account is still refused (strict), so it goes on a separate voucher.
  - PmtV TDS comes per supplier in `derived.tdsLines[]`; `derived.tds` is null on MANY.
  - The typed party line is the NET; the server's party leg is the gross.
  - Client: `DerivedVoucher.tdsLines`, `lblTds` in grpRule, a "net of TDS" chip + tooltip.
  - Test: `run.sh many` 18/18 (validate only, no post).

**2026-09-28 (user):**
- Lines: bare `+`/`-` (setLineKeysEnabled), Ctrl+Delete removed; refocus the grid after a popup pick; confirm before removing a filled line. See [[grid-line-keys-plus-minus]].
- **Party first (option A)** on MANY types with partySide DR/CR (RcpV, PmtV): `newLineSide()` opens a new line on the party side when balanced, else on the balancing side (Tally); the blank line re-sides unless `TypedLine::sideChosen`; a ledger picked on the balancing side is pre-filled with the difference. Test: `run.sh partyfirst` 5/5. Option B (header "Received in", Tally single-entry) was offered and not chosen.
- Grid 119 SQL now restricts by nature: receipt Dr / payment Cr = cash/bank only, the other side excludes them (share voucher_ledger_picker.sql). The 9 duplicate columns this left were deleted on 2026-09-28 with the user's go-ahead; grid 119 is back to its 9 original columns. /vouchers/validate does not enforce the rule yet: asked in share/accounts/notes-56 (nature rule in legalOnSide + ledgerPick, Bank OD on PmtV to decide).
- **Bill-wise POPUP (2026-09-28, user: "nobody finds the card")** — `voucher_billwise_dialog.{h,cpp}`, code-built, reuses ui_table 40 + VoucherBillDelegate. On MANY types (RcpV/PmtV/Jrl) typing a party amount → `requestBillwise` (loads bills, derives at once for the TDS gross) → opens filled oldest-first (due date, then bill date); remainder = "Advance (on account)" (DEMAND) / "On account" (Jrl), never an input. Enter on a party line's amount reopens; F2 oldest first, F3 all to remainder; OK has focus so one Enter accepts. Summary `3 bills · 55.00 adv` is a TOOLTIP on the Ledger cell only — the painted chip was removed 2026-09-28 (white text on the focused cell). Popup buttons: none default/autoDefault — plain QPushButton OK, NEVER a QDialogButtonBox (it re-defaults its first accept button on show); min size 860x420; no hint line; Enter = OK via keyPressEvent. The card (grpBillwise) shows on MANY types only read-only (posted). ONE-party types keep the card. Test `run.sh billwise` 12/12 on ZT-CUST-CASH (945 open over 3 bills); the modal-closer records the popup (`g_billwiseSeen`), sends Esc (`g_billwiseEscIgnored`), then ACCEPTS it. **No escape (user 2026-09-28):** no Cancel button, `reject()` is a no-op (Esc + window close), OK is the only way out; back out = OK then change the amount / remove the line.
- **Advance remainder is server-blocked: notes-57** (DEMAND refuses allocated < leg; ask = remainder raises one ADVANCE bill like /receipts R7). Client shows `ctx.remainderAccepted=false` warning until it lands — then flip it in openBillwise().
- After the popup: cheques 23/0/0 + paycheques 15/0/0 (2026-09-28; posted rcv00045-46, pmv00056-57, all cancelled). They set allocations directly, so the popup never opens in them. live 45/0/1 same day (pua00071 posted + cancelled by rev00105; the skip is the second-login rights case).
- **Decision 2026-09-28 (user): NO single/multiple party setting.** Register RcpV/PmtV stay MANY; one-party receipts/payments belong on menus 99/100 (Receipt/Payment screens). Rejected option: company setting `accounts.voucher_party_entry` SINGLE|MULTIPLE as a client-only layout over the MANY payload — it duplicated 99/100. Don't re-propose it.
- **Header + verbs (2026-09-28, user):** grpVoucher has no title; 4 QFormLayouts (formVoucher1-4: Party/GSTIN · Date/Collected by · Their ref/Bill date · Place of supply/Narration) in layVoucher (spacing 24, stretch 1,1,1,1), no spacer, lblFacts below and hidden while empty. Hide rows ONLY via `setRowShown(field, bool)` (QFormLayout::setRowVisible) — widget setVisible leaves a gap. Buttons: Post→"Save", Post & Print→"Save & Print", no key text on any caption; keys live in tooltips. Keys = house: F5 Save, F6 + Ctrl+Return/Ctrl+Enter Save & Print (falls back to Save while printing is disabled), F7 New, F8 List. `run.sh headshot` renders head_RcpV/PurA.png into $SHOT_DIR.
- **Enter navigation (2026-09-28):** the register header uses NexDialog's Enter-as-Tab (works in the MDI subwindow too — `run.sh entertrail` with TRAIL_MDI/TRAIL_TYPE/TRAIL_FROM_PARTY traces it). Explicit order in setupHeader: Party → Date → Their ref → Bill date → POS → Collected by → Narration → grid; txtGstin NoFocus. VoucherInstrumentDialog (plain QDialog) got its own Enter-as-Tab eventFilter (open combo list commits first; Enter on a button clicks it), plain OK/Cancel QPushButtons with no default (the QDialogButtonBox made OK default → Enter accepted a half-filled form), leaf NoFocus.
- **Drawee bank = dropdown 46 (2026-09-28, user):** VoucherInstrumentDialog `ddDraweeBank` (NexDropdownSingle, INDIAN BANKS LIST / fixed.bank_master). /vouchers has only `bankName` (no id), so the picked NAME is sent; a loaded cheque shows its saved name via setSelection("", name). Its Enter filter lets an open NexDropdown/QComboBox list take Enter first. Test closer fills it with setSelection. cheques 23/23 + paycheques 15/15 after (rcv00050-51, pmv00060-61 cancelled).
- Cheque-module dialogs + every other popup now use EnterNavigation (see [[popup-enter-navigation-rule]]); the instrument dialog dropped its own filter for it.
- **First focus (2026-09-28, user):** a new voucher (end of resetDocument, deferred) focuses Party on a ONE type, else tblLegs (0, Ledger) — `focusFirstField()`. Opened from the menu nothing had focus, so Enter went nowhere. Traces: `run.sh entertrail` (TRAIL_MDI=1 TRAIL_AS_OPENED=1 TRAIL_TYPE=…) and `run.sh chequetrail` (the popup via openInstrument, register as parent, ShortcutOverride+KeyPress) — never set focus by hand in a trace, that is what hid the bug.
- **Cheque popup Ctrl+Return / Ctrl+Enter = OK (2026-09-28, user)** — accept() still validates; tooltip on OK. Keypad-Enter fix applied to the register's own Enter tests (ledger cell, party amount) and the bill-wise popup too. The register test build now links Qt6Test (-DQT_GUI_LIB -DQT_WIDGETS_LIB) for QTest::keyClick on a window = the real key path.
- **Side lock (2026-09-28, user):** on RECEIPT/PAYMENT nature a line with its ledger picked is side-locked (`sideLocked()`, flag kSideLockedRole on the DrCr cell): Dr/Cr won't open, the other amount cell is greyed + refused, a figure typed there is carried to the right cell keystroke and all (eventFilter: setCurrentCell + edit() + insert — a re-sent QKeyEvent is NOT forwarded into the editor the view opens, the first digit was lost), onLineCellEdited ignores a wrong-side figure. To put bank first on RcpV set the blank line to By BEFORE picking. Test `run.sh sidelock` 5/5.
- **paintLegs keeps an open editor's text** (2026-09-28): a /validate or open-bills answer landing right after a pick repainted the grid and reset the editor, eating digits being typed. Saved/restored around the repaint.
- **Save explains instead of going dead (2026-09-28, user):** btnPost is disabled ONLY for hard blocks (no type / not open / no Post or Create right / no lines). A refusal, a difference, stale or in-flight figures leave it enabled (tooltip "Save and post (F5) — <why>"); post() re-validates, then refusals → the strip's confirmProceed popup (lists every reason, Back only), difference → showWarning with Dr/Cr/difference. Tests assert the popup via g_msgs ("Validation | … REFUSED"). live 47/0/1, many 20/20.
- **TDS column rule (asked 2026-09-28):** shown only where vchr_tds_mode = DEDUCT — PmtV always (MANY), PurA once the picked supplier's facts say tdsApplicable; RcpV/Jrl/Con/SalA/CrN/DrN never. `applyLegColumns()` owns the type-driven columns (GST/HSN/Instrument/Generated, TDS via paintLegs) and re-runs on NexTable::layoutLoaded — ui_table 39 marks every column visible, so a late layout used to re-show them.
- **2026-09-28 late:** notes 56+57 LIVE (migrations receipt_payment_money_side, voucher_allow_advance). 57 client done: DerivedVoucher.advances from derived.bills[isAdvance], popup warning removed, chip quotes server advance; `run.sh advance57` 7/7 (rcv00097 posted+cancelled; a cancelled voucher's bill = abl_is_deleted, figures kept). Menu 263 Cheque Books = ChequeBooksScreen (cheques/cheque_books_screen.*), notes-58 asks Prathap for the menu row + /cheque-books rights on 263; tests/issued `books` 4/4 (its modal-closer must skip ChequeBooksScreen). NexApiTableView fetch callbacks now QPointer-guarded (use-after-free).
- **Owed list CLOSED 2026-09-28:** (A) ITC combo per GST rate row (tblGst col 5, INPUT side only; `itcCombo()`, `effectiveItc()` = typed else DerivedLeg.itcEligibility) — `run.sh itc` 11/11. (B) exceptions = VoucherExceptionsDialog over grid 118 (btnExceptions on register + Jrl/DrN/CrN; Enter/double-click opens the voucher) — `run.sh exceptions` 7/7, posts+cancels one Jrl. (C) ui_table 39 already 100 / Ledger-Debit-Credit focus; grid 117 visible 97 — left. `run.sh money56` 2/2. 58 LIVE (menu 263 row + VKPOS rights; 52 kept).
- **Test trap:** since 57 a short allocation is LEGAL (advance), so a test that Saves a "short" voucher POSTS it — `many` did (rcv00098, cancelled via /vouchers/cancel). Tests assert refusals by /validate or strip, never by pressing Save on a now-legal voucher. The register harness's modal-closer now skips extra VoucherRegisterEntry windows (g_screen2 for a dialog a test drives).
- Full run 2026-09-28 evening: smoke 24, features 7, many 19, billwise 15, partyfirst 5, pickcursor 3, sidelock 5, itc 11, money56 2, cheques 23, live 47/0/1, paycheques SKIP (no book with 2 leaves — make one via menu 263).
