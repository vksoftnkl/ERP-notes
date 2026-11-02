# Received Cheques — Qt client plan (menu 51)

**For Viji. Written 2026-09-14; code after `plan-backend-received-cheques.md`
§2 is run and the eight `/cheques/*` routes answer.** House rules in
`../openingbal/plan-qt-opening-balance.md` §0 (own `.ui` on `NexDialog`, no
`NexCrud*`, no `TxnMainView`). The screen is `received_cheques_ui_mockup.png`
band for band: ONE page — the register with the action for the selected
row(s) as a panel under it.

## 1. File set

```
src/modules/accounts/cheques/
  received_cheques.h / .cpp / .ui        // NexDialog: filters, summary strip, register, action bar, the panel stack
  cheque_action_panels.h / .cpp          // the six QWidget panels (Deposit, Clear, Bounce, Re-present, Replace, Return) — one class each, in one file pair
  cheque_register_entity.h               // ChequeRow (from the grid's row JSON), DueBucket enum, status → allowed-actions map
src/core/api/endpoints/accounts_end_points.h   // ApiEndpoints::Cheques::{List, Get, History, Deposit, Clear, Bounce, RePresent, Replace, Return, DepositSlip}
```

Shared with `../payment/plan-qt-issued-cheques.md`: `cheque_register_entity.h`
(the same row shape, `apd_tra_type` decides the labels) and the History
dialog — put both in `src/modules/accounts/cheques/` and let the issued
screen include them.

## 2. The `.ui`

1. **Title strip:** company · branch · year · today; `lblPostingMode` (the
   setting, display only).
2. **Filters:** status check boxes (IN HAND / WITH THE BANK / CLEARED /
   BOUNCED / RETURNED-CANCELLED / REPLACED — `apd_status IN (…)`), instrument
   date range (`NexDateEdit` ×2), bank (`NexDropdownSingle` id 25 BANK
   LEDGERS), party (dropdown 43), search (cheque no / drawer / receipt no).
   Every filter is a grid param (`istatus`, `ifrom`, `ito`,
   `ibank_ledger_id`, `iparty_id`, `search`) on `NexApiTableView::
   setGridParam` + `reload()`; a 300 ms debounce on the search box.
3. **Summary strip** (`QLabel`s): IN HAND · WITH THE BANK · DUE THIS WEEK ·
   MATURED, NOT DEPOSITED · STALE · BOUNCED, OPEN · CLEARED THIS MONTH · the
   tie "Cheques In Hand = IN HAND + WITH THE BANK". Computed from a second
   call to the same grid with no status filter? No — one `GET
   /cheques/summary` is cheaper; ask Prathap for it (a one-SELECT endpoint,
   the strip's eight figures) — noted as a small addition to his plan. Until
   it exists, the strip sums the loaded page only and says "(this page)".
4. **The register** `tblRegister` = `NexApiTableView::setup(<grid id>)`,
   multi-select (`checkShortcutEnabled` — Ctrl+Space / Space toggles), row
   badges for status and due bucket painted by a small delegate reusing
   `TxnBadgePalette::styleFor()` (the palette is standalone — no
   `TxnMainView` needed). Columns come from the grid; the key columns
   (`apd_id, apd_acc_year, apd_company_id, apd_branch_id`) are read from
   `rowObject()`; Enter / double-click on the receipt-no cell opens the
   receipt (`FormManager` → menu 99 `loadReceipt(key)` — expose a static
   `ReceiptEntry::openFor(key)` for cross-screen drill-through).
5. **Action bar:** Deposit F5 · Clear F7 · Bounce F8 · Re-present · Replace
   · Return / Cancel F3 · Print slip F6 · History Ctrl+H — **enabled by the
   checked / selected rows' status** through `allowedActions(status)` in
   the entity header (the `ck_apd_*` map on the mockup: HELD → Deposit /
   Return / Cancel / Replace; DEPOSITED → Clear / Bounce; BOUNCED →
   Re-present / Replace; CLEARED / RETURNED / CANCELLED / REPLACED → History
   only). Mixed selections enable only what every row allows.
6. **The panel stack** `stkAction` (`QStackedWidget`) under the bar — one
   panel per action, shown when its button is pressed, with its own OK /
   Cancel:
   - **Deposit** (bulk): bank (`NexDropdownSingle` 25), deposit date, slip
     no, remarks; the list of checked cheques with Σ; OK → `POST
     /cheques/deposit` with `cheques[]`; then offer the slip print.
   - **Clear:** clear date, bank date, remarks; ON_CLEARING rows show an
     allocation grid (the receipt's bills band reused as a widget —
     extract `ReceiptBillsBand` from the receipt screen when this lands;
     until then the server's auto FIFO with an empty `allocations[]`).
   - **Bounce:** bounce date, reason (combo from `accounts.bounce_reasons`
     + free text), bank charge, party charge (pre-filled from
     `accounts.bounce_charge_to_party`); a preview box listing the legs and
     the bills that will reopen (client-side from the row's allocations —
     `GET /cheques/get` on selection); OK → `POST /cheques/bounce`.
   - **Re-present:** bank, deposit date, slip no → `POST /cheques/re-present`.
   - **Replace:** the new cheque's fields (no, date, amount, bank, branch,
     IFSC, MICR, drawer) → `POST /cheques/replace`.
   - **Return / Cancel:** action radio, reason, remarks → `POST /cheques/return`.
7. **Footer:** hint bar (Space selects · F5/F7/F8/F3/F6 · Ctrl+H · Enter
   opens the receipt · no button takes focus) · `btnClose`.

## 3. Behaviour

- Open → load the grid with the defaults (IN HAND + WITH THE BANK ticked,
  date range = the year); selection change → `GET /cheques/get` for the
  row (its party, tender row, vouchers, settled bills, replacement chain)
  into a read-only detail strip above the panel; the panel buttons
  re-enable per status.
- Every action's OK: confirm with the rows named and the money named
  ("Bounce 221870 — 50,000: two bills reopen, the party is charged 300"),
  call, then `hardReload()` the grid and reselect the row; show the
  server's message on 409 (`ux_avh_src` "already cleared on <date>", etc.).
- History: `ChequeHistoryDialog` — a plain `QDialog` listing `GET
  /cheques/history` (`txn_status_log`, newest first). Reused by menu 52.
- Print slip (F6): `GET /cheques/deposit-slip` → the print runner
  (`../payment/plan-qt-payment.md` §7); until it exists, a plain table
  print through `QTextDocument` is acceptable for a slip.

## 4. Endpoint table — all in `ApiEndpoints::Cheques`

`List` (grid), `Get`, `History`, `Deposit`, `Clear`, `Bounce`, `RePresent`,
`Replace`, `Return`, `DepositSlip`; plus the summary endpoint if Prathap
adds it.

## 5. Tests

1. Default filters load HELD + DEPOSITED; 2. status → enabled buttons per
the map, mixed selection narrows; 3. Deposit payload carries every checked
key with its acc year; 4. Bounce preview lists the bills from `/get`; 5.
after any action the grid reloads and the row stays selected; 6. Enter on
a row opens the receipt with the row's own scope; 7. the tie label equals
IN HAND + WITH THE BANK; 8. search debounced, one request per pause.
