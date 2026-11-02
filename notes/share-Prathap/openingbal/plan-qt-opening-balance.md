# Opening Balances — Qt client plan (menu 55)

**For Viji (me). Written 2026-09-14, to be coded after Prathap runs
`plan-backend-opening-balance.md` §2 and the six routes answer.** Everything
here is read off the client as it is on `main` at `f76c055` and off the
mockup `opening_balance_ui_mockup.png`; nothing is assumed about the backend
beyond the plan's §4 payloads. **No code yet — this is the plan.**

**House rules that apply to every accounts screen (this one and the four
that follow):**

- **Its own `.ui` on `NexDialog`** — a Designer file per screen, the way
  `OpeningStockEntry` / `SaleBillEntry` are built. **Not** `NexCrudMasterPage`
  / `NexCrudForm` / the generic CRUD dialog — the user's instruction of
  2026-09-14 evening. The F8 list of a voucher screen is a page inside the
  screen's own `.ui` (a `QStackedWidget`: Entry / List) holding a
  `NexApiTableView` on the module's grid — not `TxnMainView`, which is a
  `NexCrudMasterPage`.
- Grids are `NexTable` + a `QStyledItemDelegate` subclass emitting
  `cellEdited(row, col)` from `setModelData()`; the screen connects to THAT,
  never to `itemChanged` ([[grid-delegate-edit-pattern]]). Layout by
  `loadLayout(<ui_table_master id>)`; the `.ui` still declares every column
  so the grid is usable when the server cannot answer.
- **WIDTH AND VISIBILITY BELONG TO `ui_tables`, NOT TO THE SCREEN** (added
  2026-09-15). `applyLayout()` writes visibility, header text AND width, so a
  screen that re-seeds its own widths on `layoutLoaded` puts hardcoded pixels
  and `QHeaderView::Fixed` back over the operator's saved percentages — the
  widths simply will not stick. Seed header text and the hidden flag ONCE,
  before the server answers; set no width and no resize mode; do NOT reconnect
  `layoutLoaded`. The only two things the screen may set are a
  `minimumSectionSize` floor and `stretchLastSection`, which decides where the
  rounding remainder goes rather than overriding a stored width.
  `NexTable::showEvent()` re-applies the percentages against the real width, so
  an offscreen test must `show()` the screen or every column measures the floor.
- **Every `ApiClient` callback must capture a `QPointer`, not `this`.**
  `NexDialog` sets `Qt::WA_DeleteOnClose`, so closing a screen DELETES it and
  an in-flight reply calls back into freed memory. Opening menu 55 and pressing
  Esc before the chart arrived segfaulted in `QTableWidget::setRowCount`.
- **Never gate an action on `ctx.canCreate` / `ctx.canEdit` directly.**
  `FormManager::reg<T>()` builds the dialog with `new T(ctx.parent)` and
  assigns `ctx.menuId` only AFTER the factory returns, so a constructor that
  calls `setup()` always runs with `menuId == 0`, never loads permissions, and
  leaves every flag false. An unloaded permission is not a denied one —
  `AppSession::canCreate()` itself defaults to TRUE for an unknown menu. Use
  the `ctx.menuId == 0` escape hatch in `applyPermissions()`.
- Widgets: `NexDropdownSingle` (dropdown ids), `FetchLineEdit` + popup grid
  (`NexTable::setPopupConfig`) for pickers, `NexDateEdit` (`dateEdited` only),
  `NexDoubleEdit` for money, `AppMessage` for every message, `ApiClient::
  instance().get/post(endpoint, onOk, onErr)` with `ApiClient::handleError`.
- Scope: `m_companyId / m_branchId / m_accYear` captured from `AppSession`
  on New and from the loaded document afterwards — never re-read mid-life
  (`TxnDocKey` reasoning).
- Ctrl+Enter is the primary verb from anywhere; plain Enter belongs to the
  grid; **no button is a default button**; F-keys as the mockup's footer.
- Money is kept as `double` members, never parsed back out of labels
  (`OpeningStockEntry::m_totalValue` comment).
- Registration: one line in `src/core/navigation/form_accounts_menu.h`
  (`fm.reg<OpeningBalanceEntry>(55)`), the files added to `CMakeLists.txt`
  by hand (no GLOB), endpoints in a new `src/core/api/endpoints/
  accounts_end_points.h` under `ApiEndpoints::OpeningBalances` etc.
- Permissions: `NexDialog::applyPermissions()` reads `AppSession::canCreate/
  canEdit/canDelete/canPrint(menuId)`. **No approve right** — the approval
  step was withdrawn 2026-09-15 for every accounts screen, so
  `um_can_approve` / `AppSession::canApprove` are NOT being added.

---

## 1. File set

```
src/modules/accounts/opening/
  opening_balance_entry.h / .cpp / .ui     // NexDialog — the one page
  opening_balance_delegate.h / .cpp        // ledger grid: Opening (NexDoubleEdit), Dr/Cr (combo), Remarks
  opening_bill_delegate.h / .cpp           // bill panel: refno, dates, days, Dr/Cr, amount, narration
  opening_balance_entity.h                 // OpeningRow, OpeningBill structs + toJson/fromJson, column enums
src/core/api/endpoints/accounts_end_points.h   // ApiEndpoints::OpeningBalances::{List, Create, TrialBalance, Delete, CarryForward, Bills} + query builders
```

## 2. The `.ui` — one page, top to bottom (the mockup)

1. **Scope strip:** company NAME · year (the name is not in the login payload
   — the token carries only `company_id` — so it is fetched once from
   `/company-masters/get` and cached on `AppSession::setCompanyName()` for the
   four accounts screens still to come; the uuid is the tooltip), `cmbScope`
   = "This branch" / "All branches" (a plain `QComboBox`; All sends
   `branch_id` absent and **closes the bill panel** — `abl_branch_id` is NOT
   NULL), `btnCarryForward` (visible only when the prior year exists and is
   CLOSED — from `/list`'s `priorClosing*` presence), `btnReload`.
2. **Trial-balance strip:** Σ Dr · Σ Cr · Difference (`lblTbDr / lblTbCr /
   lblTbDiff`, red when ≠ 0) · the plug line ("OPENING_DIFFERENCE will take
   …" — informational; the server writes it) · staleness banner (`opIsStale`
   rows count, `opStaleSince`).
3. **Ledger grid** `tblLedgers` (`NexTable`, **ui_table 30** "OPENING
   BALANCE - LEDGERS"): Ledger · Group · Nature · Bill-wise (chip) · Prior
   closing · Prior side · **Opening** · **Dr/Cr** · Source (chip: MIGRATION /
   CARRY FORWARD / MANUAL) · Stale · Bills · Remarks · hidden `opId`, `ledId`,
   `ledIsBillByBill`.

   **REVISED 2026-09-15 — rows are the openings that EXIST, plus what you
   pick.** This plan originally said "EVERY balance-sheet ledger the server
   lists, no Add row", on the argument that a partial chart is a wrong trial
   balance. **That argument is false and the server settles it:** asked for the
   same year with `includeZero` true and false it answers 31 rows and 5 rows
   and THE SAME TRIAL BALANCE, because it totals the whole chart server-side
   whatever it puts in `rows[]`. A ledger with no opening contributes zero to
   both sides.

   So the grid loads with **`includeZero=false`** — every ledger that already
   has an opening, plus every one carrying a prior-year closing (the
   carry-forward candidates) — and everything else is PICKED, the way a
   quotation picks an item. A medium company keeps 1000+ debtors; painting 970
   rows of `0.00` is not a review surface, it is a haystack with the trial
   balance hidden in it.

   What is lost — seeing at a glance that a ledger was forgotten entirely — is
   covered by the strip, which is permanent, comes from the server over the
   FULL chart, and shows an omission as a difference.

   **THE INVARIANT THAT MAKES IT SAFE:** `POST /create` is sent with
   `replace:true` and the server SOFT DELETES every opening absent from the
   payload. That is what lets a row be cleared by removing it, and it is only
   safe while the grid holds EVERY row that has an opening. `includeZero=false`
   guarantees exactly that and the picker only ever ADDS on top. **So never
   page, filter or trim the loaded set** — a search box over these rows, added
   later in all innocence, would make Save delete every opening that scrolled
   out of view.

   **The picker** is `NexTable::setPopupConfig` on the Ledger cell, grid
   **107 "POPUP - LEDGERS"** (registered 2026-09-15). Its SQL restricts to
   `acc_group_nature IN ('Assets','Liabilities')`, so a P&L ledger the server
   would reject cannot even be offered, and it is scoped by `iled_company_id`
   so one company is never shown another's ledgers. Note the parameter
   convention: the configured-grid runner does **textual substitution of bare
   `i`-prefixed tokens** (`NULLIF('iled_company_id','')::uuid`), NOT `:name`
   binding — `:name` is not supported by the deployed runner; see notes.txt (14).

   **There are no Add / Remove buttons** (revised 2026-09-15 evening). BOTH
   grids always end in ONE blank row, which IS the picker — adding a ledger is
   typing into the last line, and picking into it grows the next blank row
   behind it. Removing uses `NexTable`'s own line keys (`setLineKeysEnabled`,
   `lineCreateRequested` / `lineRemoveRequested`), the same `+` / `-` the
   quotation and sale bill use; NexTable refuses `-` on the last row by itself,
   which is exactly right when the last row is the blank one.

   Every removal CONFIRMS first — `-` is one keystroke and the row may be a
   figure somebody spent the morning agreeing — and a row with a SAVED opening
   says so in stronger words, because dropping it and saving DELETES it
   server-side. Removing a bill-wise party that still has bills is refused
   outright: they own its figure and the server retains the row anyway.
   A duplicate pick is refused by name: `ux_op_scope` is one row per ledger
   per year.

   The trailing blank row is furniture, not data: it does NOT mark the screen
   dirty, `buildLedgerPayload()` drops it, and validation says nothing about
   it. Same for the bill grid's.

   **And nothing on it opens until a ledger is picked into it.** The delegate
   refuses Opening / Dr-Cr / Remarks on any row with no `ledId` and paints them
   grey — an amount with no account is not an opening balance, it is a number
   floating in a grid, and the payload would drop it anyway. The Ledger cell is
   left to `NexTable`'s own picker. Same rule `OpeningStockDelegate` applies to
   a line with no item. A bill-wise
   row's Opening cell is **disabled** (it is the NET of its bills, not the sum
   — 5,000 Dr and 3,000 Cr open the party at 2,000 D) and painted grey by the
   delegate.

   **A SINGLE CLICK opens its bills below** (revised 2026-09-15), through
   `cellClicked` and `currentCellChanged` so the mouse and the arrow keys
   behave alike. Double-click is deliberately NOT wired: it is how a cell is
   opened for editing, and hijacking it made an ordinary ledger answer "Not a
   bill-wise party" instead of letting a figure be typed. Selecting an ordinary
   ledger CLOSES and CLEARS the panel; switching party clears the grid BEFORE
   the fetch, so one party's invoices never sit under another's name.
   Editing a CARRY FORWARD row repaints its Source chip MANUAL on the client
   at once (the server flips it anyway — §5.5 rule 1) so the operator sees
   what they did.
4. **Unclassified banner:** `unclassified[]` from `/list` — "3 ledgers
   have no group nature and are not listed: …" with a button to open the
   Account Group master (menu 54 via `FormManager`).
5. **Bill-wise panel** `grpBills` (collapsible, under the grid, never a
   modal): party name · tie strip (Σ bills vs the party's opening — always
   equal by construction after save; shows the difference while editing) ·
   `tblBills` (`NexTable`, layout id to issue — "OPENING BALANCE - BILLS"):
   Doc ref · Doc date · Due date · Credit days · Grace · Dr/Cr · Amount ·
   Narration · Allocated (read-only) · Status chip · hidden `ablId`,
   `ablAccYear`. A receipted bill (Allocated > 0) has only the date / days /
   narration cells enabled (§5.5 rule 3) and the delegate paints the frozen
   cells grey with the reason in the tooltip; the hint bar names the two
   honest fixes.
6. **Footer** (revised 2026-09-15 evening), TWO bands stacked:
   - `frameTotals` — Σ Dr · Σ Cr · Difference · plug · server figure, in a
     bordered card of its own. It moved down from the top of the screen so the
     number that decides whether saving is sensible sits beside the button that
     saves, but it keeps its own band: figures crammed into the same line as
     the hint text and the buttons read as a caption rather than as a total.
   - hint bar · spacer · `btnSave` · `btnClose`. Save reads just "Save" — the
     Ctrl+Enter chord still works and is named in the hint bar, not on the
     button.

## 3. Behaviour

- **Load** (`GET /opening-balances/list?company_id&acc_year[&branch_id]`):
  paint the grid, keep `m_rows` (`QList<OpeningRow>`) as the model of
  record with each row's loaded `opModifiedAt` (for the optional staleness
  guard); compute the trial balance on the client from `m_rows` and show
  the server's `/trial-balance` figure beside it only on demand (they must
  agree — a mismatch is a bug, shown loudly).
- **Edit** a cell → `cellEdited` → validate (amount ≥ 0; side D/C; a
  bill-wise row refuses) → recompute the strip → `m_dirty = true`.
- **Bills** for the selected party (`GET /opening-balances/bills?party_id
  &company_id&acc_year&branch_id`): paint `tblBills`; edits recompute the
  tie; the party's Opening cell mirrors Σ bills live (read-only).
- **Save** (Ctrl+Enter): TWO calls in order — `POST /create` with EVERY
  ledger row (`replace: true`; a bill-wise row echoes Σ bills), then `POST
  /bills` once per party panel touched (`replace: true`, branch REQUIRED).
  Refuse locally before the round trip when: the set is unbalanced beyond
  the plug's tolerance? No — the server plugs; send it. Refuse when a
  bill-wise party's panel is dirty and its bills do not sum to a positive
  figure, or a frozen bill's amount/side/presence changed. After both
  answer: reload (`/list`) — the server owns ids and sources.
- **Carry forward** (`POST /carry-forward`): confirm with the prior year
  named and the MANUAL-rows-spared rule quoted; then reload.
- **Delete a row** = amount 0 and save (there is no row delete; the server
  soft-deletes absent / zero rows) — the delegate accepts 0.
- **All branches** switches `cmbScope`: confirm if dirty, reload with
  `branch_id` absent, hide `grpBills`.
- **History** (Ctrl+H): `AuditHistoryDialog::fetchData(screenId, …)` once
  the backend issues an `audit_screen` id for this table (plan §5.5 note);
  the button is hidden until it does.
- **Close**: `closeEvent` confirms when dirty (the `OpeningStockEntry`
  pattern).

## 4. Endpoint calls (all in `ApiEndpoints::OpeningBalances`)

| call | when |
|---|---|
| `GET /opening-balances/list` | open, reload, after save, scope change |
| `GET /opening-balances/trial-balance` | on demand (`btnCheckServer`) |
| `POST /opening-balances/create` | save, step 1 |
| `GET` / `POST /opening-balances/bills` | party selected / save step 2 |
| `POST /opening-balances/carry-forward` | the button |
| `DELETE /opening-balances/delete` | NOT used by this screen (amount 0 + save instead) — keep out of the endpoints header so nobody calls it |

## 5. Backend facts the client depends on (read before coding)

DECISION 1 (`led_ob_*` vs `acc_opening_balance`) must be closed; DECISION 10
(ledger visibility) decides the grid's row set; `abl_branch_id` NOT NULL
decides the All-branches rule; `opSource` echo; `replace: true` semantics;
`priorClosing*` presence gates Carry forward; `unclassified[]` present.

## 6. Tests (offscreen harness, [[qt-offscreen-test-harness]])

1. Load paints N rows, strip = Σ; 2. edit a CARRY FORWARD amount → chip
MANUAL, dirty; 3. bill-wise row Opening cell refuses focus; 4. panel edit
mirrors Σ into the party row; 5. frozen bill: amount cell refuses, due date
accepts; 6. save sends `/create` then `/bills` in order with `replace: true`;
7. All branches hides the panel and drops `branch_id`; 8. close while dirty
asks; 9. locale with comma decimals builds the payload from doubles, not
labels.
