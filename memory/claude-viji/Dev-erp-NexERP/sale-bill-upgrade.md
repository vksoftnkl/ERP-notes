---
name: sale-bill-upgrade
description: The Qt sale bill screen upgrade to the A3 backend — every section of plan §1 built as of 2026-09-23; the decisions that shaped each, and what waits on the server
metadata:
  node_type: memory
  type: project
---

Started 2026-09-22 evening once A3/A4/A5 were verified. `plan-qt-sales.md` §1
is the spec — the screen EXISTS (5,026 lines), so this is "changes only".

## Done and building clean

**1. `ApiEndpoints::Bills` extended to the full 14 verbs** (`src/core/api/api_endpoints.h`).
Built from the shipped controller + DTOs, not the spec.
**THE TRAP, encoded in query builders:** the four keys have TWO spellings —
document-addressed routes (`/get`, `/tender-context`) take **sb-prefixed**
(`sbId`…), the two context reads (`/open-sources`, `/party-context`) take
**BARE** (`companyId`, `partyId`…). Whitelist validators ⇒ a wrong name is a
400 "property should not exist", never a silent unfiltered read. Ids are
`ParseUUIDPipe({version:'7'})` — a uuid4 is refused.
`GetParams` replaced by `BillKey` (+ `toJson()` / `toQuery()`), with
`using GetParams = BillKey` so old call sites compile.

**2. `NexWarningStrip`** — `src/shared/ui/txn/nex_warning_strip.{h,cpp}`, in
CMake. THE shared validate/post band for all six posting screens. Reuses
`TxnBadgePalette` (include **`../theme.h`** for `toCss`, not the palette
header). Four rules built in, each a money-loss if a screen improvised:
refusals never tickable; checkbox only when `overridable` AND
`rights.override` (else a quiet "needs override right"); hides when empty;
**ticks do not survive a new answer**. `parse()` reads all four envelope
shapes (`data`, `details`, `error.details`, bare) — screens would hand-map one
and miss the 422. Carries the `statutory` block so a refusal reads
"269ST · 2,00,000 · from 2017-04-01".

**3. The verbs in `sale_bill_entry`** — Validate (Ctrl+F7), Post
(Ctrl+Shift+Enter), Post & Print (Ctrl+Enter), `cancelBill()`, `beginAmend()`,
`applyPostingState()`. Strip lives in **`root` row 4** — the one row the .ui
left free, between the item grid (3) and totals (5).
* **Post RE-VALIDATES** before posting — a stale "no warnings" is the
  difference between an operator who knowingly overrode a rate cap and one who
  never saw it.
* **A 422 on validate is the ANSWER, not a failure** — paint the strip, do not
  pop a dialog over a band that already explains itself.
* **Rights come from `/get`'s `rights` block, NOT NexDialog's `ctx`** — ctx has
  create/edit/delete/export/print and knows nothing of post/cancel/amend/
  override. Members `m_canPost/_canCancel/_canAmend/_canOverride`.
* `applyPostingState()` reads locks and names WHY a verb is off, in the order
  the operator must act: IRN first, then EWB, then returns/allocations/day-close.
  Lock 2 removes Amend from a user who HOLDS the right — `rights.amend` alone
  must never light the button.
* `ApiError` field is **`responseBody`**, not `payload`.

## §1.1 IS COMPLETE (2026-09-22 evening), builds clean

**4. The buttons + the rest of §1.1.**
* **Cancel / Amend on the IDENTITY STRIP, not the bottom bar** — that row's
  `btnCancel` closes the FORM, and two things called Cancel side by side on a
  money screen is how the wrong one gets pressed. Built in code into
  `ui->hboxHeader` (`m_btnCancelBill` / `m_btnAmendBill`).
* **`refreshVerbButtons()` is the ONE place** that paints them — an enabled
  state set in five places disagrees with itself in one. Called from
  `applyPostingState`, `beginAmend`, `resetScreenState`, the ctor.
* **HIDDEN vs GREYED is deliberate:** hidden = never applicable in this state
  (Cancel on a draft — a draft is DELETED, not cancelled); greyed = applicable
  but blocked, with the reason in the tooltip. Amend stays VISIBLE-but-greyed
  on a posted bill because "why can't I amend this" is what the tooltip is for.
* **Delete is a DRAFT verb only** — deleting a posted bill would leave a hole
  in the series where an auditor expects a document.
* **Save routes to `amendBill()` while `m_amending`** (no separate button —
  the gesture is the one they know), and the Save label becomes "Post
  amendment" BEFORE it is pressed. `/bills/amend` sends the whole payload +
  `baseRevision` + mandatory `editRemark`. `SALES_REVISION_STALE` = somebody
  else amended it while this screen was open.
* **`sales.auto_post`** — after `/create` succeeds it adopts the returned
  `sbId` then calls `postBill()`. The id must be adopted first or the post has
  nothing to address.
* **`resetScreenState()` clears the posting state** — otherwise a new bill
  inherits the last one's verbs, including a stale `baseRevision` that would
  amend a document the operator is no longer looking at.

## PERMISSIONS — the blocker, found and CLOSED same day

The verbs read `/bills/get`'s `rights`, which are `user_menus` flags. **All
1,111 rows had every posting flag false** and the admin screen had no columns
for them — `user_menu_widget.cpp` had six hard-coded columns and
`for (int c = 0; c <= 6; ...)` loops. So the screen correctly greyed every
verb and looked dead.

Two docs written, **Prathap ran BOTH and they are deployed + verified**:
* `USER_MENU_POSTING_RIGHTS.md` — the 5 fields in `SaveUserMenuDto` +
  `UserMenuPayloadDto` + the service. Live API serves 5/5 on both.
  **Why the client could not act alone: `main.ts:190` is
  `forbidNonWhitelisted: true`, so sending `umCanPost` would 400 the WHOLE
  user-admin save.** The client work sat behind `kServerHasPostingRights`.
* `PERMISSIONS_MODEL.md` — the answer to "too many permissions". **218 leaf
  menus x 11 flags = 2,398 checkboxes/user, ~870 meaningless; retender
  meaningless on 217 of 218.** Argued AGAINST the user's first idea (move
  retender to `user_master`) — it fixes one flag and leaves no rule for the
  next, and splits "why can't they post?" across two tables. **THE RULE: a
  verb exists on a menu if the menu can DO it.** `fixed.menu_master.menu_verbs
  text[]` now carries it; on the box: 205 masters (6 CRUD), 19 posting docs
  (+POST/CANCEL/AMEND/OVERRIDE), 2 read-only `{VIEW,PRINT}`, **RETENDER on
  menu 12 alone**. Roles proposed as phase 3, with the rule that a role is a
  TEMPLATE applied on assign, never a live indirection.

Client now reads `menuVerbs` from `/menu-masters/get` (both `get()` and
`getUserMenu()` select it) into `m_verbs`; the temporary hardcoded
`postingMenus()` set is **deleted**. A menu answering with no verbs falls back
to the six CRUD ones. **A cell whose menu lacks the verb gets NO checkbox** —
an unchecked box says "denied", which is a different statement from "this
screen has nothing to post". All/None skip those cells.

`tester1` is granted post/cancel/amend/override on menus 11, 12, 13, 182 and
retender on 12 — so the bill screen can now be driven for real.

## FOUR BUGS FOUND BY THE USER, ALL FIXED — see [[verify-against-artefact]]

1. `ctx.canPost` does not exist — rights come from `/bills/get`.
2. **`enterEditMode()` unlocked a POSTED bill without amending**, so Save hit
   `/bills/create` and the server said "use /bills/amend". **On a posted bill,
   Edit IS Amend** — it now routes to `beginAmend()`, one gesture for "change
   this document".
3. **Gated Amend on `locks.editable.document`** — the server computes it
   `status === 'DRAFT' && amendable !== false`, so it is **false on every
   posted bill by design; that IS lock 1**. Amend is the verb that reopens it.
   Now: POSTED + no IRN/EWB + `rights.amend`. `editable.document` is used for
   the READ-ONLY state instead, which is what it is for (§1.6a: the client
   never computes this).
4. Audited by grepping invented widget names; reported real features missing.

**Also fixed while tracing:** `/bills/post` takes only the KEYS, so validating
the screen and posting the stored draft were two different documents whenever
the operator edited after saving. **Post now SAVES FIRST** (`/bills/create`
updates in place when sbId is present), then validate, then post. Auto-post
skips the extra save — its callback IS the save.

## VERIFIED against a live `/bills/get` (2026-09-22)

Every field the new code reads exists; `canCancel`/`canAmend` both compute
true for a posted bill. The header comes back **FLAT** (sbStatus,
sbRevisionNo, sbBillRefno top-level) — there is no `header` sub-object; a dead
fallback to one was removed. **`rights` has only four keys — no `retender`.**

## THE AUDIT — `share/sales/AUDIT_sale_bill_2026-09-22.md`

§1.1 done; **nine of twelve sections not started**; 4 of 11 endpoints wired.
Two corrections after reading the real `.ui`: the **BILL-TO fields already
exist** (`txtCustName/Addr/Place/Phone/Gstin` — missing only PIN and the "Edit
for this bill" lock), and the **transport CREW fields already exist**
(`ddDriver/ddSupervisor/ddLoadman/txtVehicleNo`) — it is the consignment half
that is absent. Only 3 of 18 quick-strip buttons are connected.

## Then §1.3–§1.12

Challan import (Ctrl+F5, grid 114), append-capable order import, charges
"from order" column, party panel, transport band, delivery status, re-tender
dialog (grid 133 + `/tender-context`).

**Build:** `PATH=/home/vk/Qt/Tools/CMake/bin:/home/vk/Qt/Tools/Ninja:$PATH ninja`
in `build/Desktop_Qt_6_10_2-Debug`. Takes ~4 min cold; run it in background.

## §1.5 PARTY PANEL — DONE 2026-09-23, builds clean, create route dry-run live

**Two calls still, not one.** `/bills/party-context`'s `party` block has NO
addr/place/pin/phone (verified live; `bill-read.service.ts partyContext()`
does not SELECT them), so it replaced `/master-lookups/party-credit` only.
`fetchCustomerDetail` stays for the BILL-TO band + area/freight/allow_* flags.
Asked as `share/sales/notes-40-party-context-address.md`.

* `fetchPartyContext()` — BARE keys + `billDate` (ISO); the stale guard is
  `customerId != ui->ddCustomer->selectedId()` because the combo is the one
  thing every route (pick / seed / load / import / copy) sets synchronously —
  `m_voucherParam.customerId` is NOT set yet on the default-customer seed.
* `applyPartyCredit()` reads the NEW keys (used/openBills/limitAmount/
  limitBills/creditDays/oldestOpenDays/amt|bill|daysExceeded/mode). **A limit
  of 0 means NO limit** — the save gate only treats "nothing left" as a breach
  when a limit exists. `mode` OFF ⇒ checkEnabled false. grpCredit rows
  relabelled: "Overdue"→**Credit Days**, "Overdue By"→**Oldest Open** (the
  endpoint ages the oldest bill; it has no overdue-amount sum).
* `lblPartyFacts` (grpCredit row 9, rich text): PAN ✓ / Form 60 / "No PAN /
  Form 60" (amber — the 269ST fact), advances, credit notes "— F4 adjusts",
  temp credit open, cash today, ship-to count, open DC · orders.
* Loyalty points now come from `loyalty.balance` (null ⇒ 0), NOT
  `cust_points` (which was null anyway); tooltip carries card / redeemable /
  max on this bill. The tender Context still reads `edtLoyaltyPoints`.
* **BILL-TO lock:** `applyBillToLock()` uses **setReadOnly**, never
  setEnabled — `applyCustomerLock()` (import lock) owns setEnabled on the
  same six fields and two locks on one property undo each other. Listed
  customer ⇒ locked until `btnEditBillTo` (checkable, in `hboxCustomerPick`
  beside the combo); walk-in (`party.isWalkIn`, session default until it
  lands) ⇒ open. The toggle is the ONE writer of `m_billToUnlocked`; resets go
  through `setChecked(false)`.
* `txtCustPin` (in `hboxPlacePin`, 6-digit validator) is what `sbCustPin`
  is built from now; order import paints `soCustPin`, a quotation has none.
* **Quick-add** = `src/modules/sales/common/customer_quick_add_dialog.{h,cpp}`
  (code-built QDialog, in CMake). Sends cusName/Phone1/GstNo/GstType/AreaId/
  GroupId/StateCode/StateName/PriceLevelId(sales default)/CompanyId/BranchId/
  IsActive. Dropdowns 13 area, 33 group, 21 state. SEZ/Overseas NOT offered
  (ledger reg-type CHECK 400s them). **Dry-run live 2026-09-23:** 201, data
  has cusId + cusName, ledger row group = area, reg type UNREGISTERED; deleted
  after. On success it goes through `confirmCustomerChange()` + both fetches
  exactly like an operator pick.

## 2026-09-23 EVENING — THE WHOLE OF PLAN §1 IS BUILT (builds clean, live routes dry-run)

**The verb bar is a STATE TABLE now** (`refreshVerbButtons`, the user's "too many
buttons" question): hidden, never greyed, when a verb cannot apply; ONE primary F5
whose label says what it does (Post / Save draft / Post amendment; Tender on the
tender route); a separate **Post** button ONLY when `sales.auto_post` is off;
**Edit and Amend are ONE button** (`btnEdit` reads "Amend - F2" on a posted bill,
`m_btnAmendBill` is gone); bottom-bar Cancel renamed **"Close - Esc"** so the only
Cancel is "Cancel bill" on the identity strip; Hold/Pick Held/Clear/Delete only
while keying a draft; Validate has no button (Ctrl+F7 stays).
`updateTenderButtons()` now only decides `m_tenderRoute` and defers painting.

**§1.6** `TxnTransportBand` (`shared/ui/txn/`, direction-aware, prefix-parameterised
`readFlat/writeFlat/toNested`) in root row 4 above the strip. Flat `sbShip*` on
/create; `PUT /bills/transport` (own Save button) once POSTED and
`locks.editable.transportBand`; lock 2 → `setDeclared`. Ship-to combo is fed from
party-context `shipTo[]` (no grid 110 read). Vehicle is a HEADER field: the band's
picker (dd 56) writes `txtVehicleNo`/`vehicleId`, the header edit mirrors back.
`SALES_EWAY_TRANSPORT_MISSING` from the strip's new `notesChanged` opens the band.
Dropdowns **55/56 registered** (masters are empty on the box).

**§1.3** `OpenSourcesDialog` over `/bills/open-sources` (QTreeWidget, doc → lines,
qty ≤ open, red past window). Ctrl+F5 challan; Ctrl+F4 on a bill WITH lines
appends (`sales.multi_order_bill`), on an empty bill keeps the whole-order import.
**Per-line trail is REAL now:** ui_table 22 gained hidden cols 94–98
(`SrcDocType/SrcLineId/SrcDocYear/LotId/Bucket`, `ColumnCount = 99`);
`SrcDocId` holds the DOCUMENT (it used to hold the soi line id and was sent as
`sbiSrcDocId` — a bug: `sbiSrcItemId` was never sent, so charge carry never
proposed). `updateSourceChip` enumerates every source; `m_sourceRefnos` maps ids.

**§1.4** `ChargeGridController` carry: facts on the ChargeName cell's item data
(`kCarryRole`), `markRowsAsCarriedFrom(year)` BEFORE `clearRowIds()` on order
import, `applyCarryProposals(proposals.charges)` at both validate sites, basis via
right-click (PRORATA/FULL/MANUAL/NONE), typing the amount ⇒ MANUAL; payload
`cdSrcCdId/cdSrcAccYear/cdCarryBasis`.

**§1.2** `m_billMode` WHOLESALE → `sbBillMode`; loaded bills keep theirs.

**§1.10** `WeightBarcode` (`sales.weight_barcode` JSON) — the delegate's
`automode(row)` on an empty row now RESOLVES (`resolveScannedBarcode` →
`/master-lookups/item-by-barcode`); nothing resolved a scan on the bill before.
The value waits in `m_pendingScans` until `fetchItemPrice` lands
(`applyPendingScan`): WEIGHT → BillQty, PRICE → Rate.

**§1.7 A–E** in `SaleBillTender`: cash-today line + limit from the pre-validate
note, PAN/Form 60 box (`identityCaptured` → `sbCustPan/sbForm60Ref`), loyalty from
the SCHEME (party-context) with points↔amount both ways + cap + multiple, daily
limit in the hint, TEMP_CR panel (`SaleTenderLine::TempCredit` → `tempCredit` on
the tender row; `/get` echoes it). **openTenderDialog now VALIDATES FIRST** and
opens `showTenderDialog(notes)`; refusals stop before the money is counted.

**§1.7b** `SaleBillRetender` (own dialog: void ticks + reason, new rows from the
masters, Voided vs New footer) over `/tender-context` + `/retender`; picker =
`TxnDocTypes::retenderPicker()` grid **115** (device + today); Ctrl+F6 opens it
regardless and explains that `rights.retender` is not served yet.

**§1.8** badges (`m_lblGstBadges`) from `posting`; e-Inv/e-Way/Part-B call
`ApiEndpoints::Gst::*` (HANDOVER paths, NOT on the box — they 404 today);
Shift+F2/F3 → `SessionEvents::requestOpenMenu(152/153)` (new signal, shell
listens). Edit remarks button (posted) → `/bills/update-remarks`.

**§1.11** `TxnDocTypes::billDelivery()` menu 226, grid **113**, row actions
Verify/Packed/Dispatched/Delivered → `/bills/delivery-status` (dry-run OK).
**§1.12** `tempCredits()` menu 257 (row missing on the box), grid **114**,
Receive → menu 99 + mobile on clipboard, Follow-up dialog → `/temp-credits/follow-up`.
`TxnDocKey` now carries `row` (the raw list row) for such actions.

Read [[grid-runner-binding-rules]] before writing any grid SQL again.

## THE .ui REBUILD (2026-09-23 afternoon/evening) — how the screen is put together now

`sale_bill_entry.ui` root grid `root` (inside `gridLayoutWidget_2`, `setLayout(ui->root)`):
row 0 `frameHeader` identity strip (title, status, audit; badges / Edit remarks /
Cancel bill are code) · 1 `frameQuick` icons-only 44×40, keys in tooltips,
`btnQuickImportChallan` + `btnCopyAsNew` (QToolButton, Alt+Y) live here · 2
`frameHdrFields` Bill No · Date · Type(comboTerm) · Their Ref(txtUsrRefno) ·
Ref Date(dtUsrRefdate) · Price Level · Salesman · Due(lblDueValue) · Category ·
3 `frame` = `grpCustomer` PARTY (`partyGrid`: customer + Quick add, lblFacts1/2,
POS state, beat, option ticks) | `wRight` = `grpSources` (lblSourceList) over
`grpCredit` (`creditGrid` fact card) · 4 `tblBillItems` (`loadLayout(id,false)`,
NOT fixed height) · 5 `frameLower` = tblCharges | grpRemarks · 6 the warning
strip (code, `root->addWidget(m_warnings, 6, 0)`) · 7 `frame_2` `footerGrid`
facts (ITEMS TAXABLE GST CHARGES(edtCharges) BILL AMOUNT ADV SET OFF BALANCE;
the rest hidden on rows 2/3) · 8 `hboxBottomBar` (Hold · Pick Held · Tender/
Post/Save draft F5 · Post · Post & Print F6 · Bill List · Clear · Last Bill ·
Close) · 9 `frameShipping` HIDDEN = grpBillTo + grpDelivery + `vboxTransportSlot`
+ grpTerms, which `buildShippingDialog()` moves into the modal popup with the
band (`TxnTransportBand`, ship-to combo item 0 = bill-to via `setBillToDefault`).
Members that grew up in code (`m_txtUsrRefno`, `m_lblFacts1/2`, `m_lblSourceList`,
`m_btnPostFoot`…) are bound to `ui->` widgets in the ctor.
**Regeneration recipe:** python over the OLD .ui — `nodes[name]` deep-copied
verbatim for every leaf widget, new containers/layouts built around them,
customwidgets/resources/connections copied; assert no leaf is missing. A
converted QPushButton→QToolButton must drop `autoDefault`.
**Stylesheet rule:** anything the code repaints with `setStyleSheet()` (the
credit figures) must be styled from the DIALOG sheet by selector, not on the
widget. `QGroupBox { padding:5px }` in the dialog sheet is why titled groups
(Delivery crew / Terms) overlap their first row in the popup — untouched.

## THE DESIGN PASS (2026-09-23 evening) — after the user said the UI was ugly

The arrangement was the mockup's; the LOOK was still the legacy form. Fixed by
moving the whole visual layer into ONE stylesheet on the dialog:

* deleted 27 `<font><bold>` label properties and ~25 per-widget stylesheets;
* the sheet styles by object name and by DYNAMIC PROPERTY —
  `caption="true"` (small grey label over a figure), `fieldLabel="true"`
  (label beside an input), `heading="true"` (band title),
  `figure="true|big"` + `tone="accent|alert|good"` (a read-only number).
  **A widget the code repaints can never lose its look this way** — that is
  why the credit figures moved here (`applyPartyCredit` clears widget sheets);
* band 1 captions now sit ABOVE the fields, as the mockup draws them
  (`hboxHdrFields` became a 2-row `hdrGrid`, one column per pair);
* `lblStatus` is a PILL — amber DRAFT / green POSTED / grey CANCELLED via
  `pill` dynamic property (set it from code when the status changes);
* every input 26px, one border `#D7DCE3`, focus `#7B1113`, read-only tinted;
* figures render as text (transparent, no border); Bill Amount maroon,
  Balance red, Adv set off green;
* grid header/gridlines restyled; Post / Post & Print / Tender maroon.

**Three layout traps found by rendering, in order:**
1. a card stretched to match a taller sibling SPREADS its rows — park the
   slack in one expanding bottom row (`spacerPartyFill` + `rowstretch`),
   never leave the grid to distribute it;
2. `alignment="AlignTop"` on the grid item looks like the fix and is not —
   it gives the widget its sizeHint and CLIPPED the credit card's last rows;
3. `tblCharges` shows 3 of its 33 columns, so the card was bare table behind
   the last one — `horizontalHeader()->setStretchLastSection(true)` after the
   ChargeGridController is built.

The party card carries white space at the bottom: that is the room BILL-TO
vacated when it moved to the shipping popup. Deliberate, not a bug.

## BACK TO THE HOUSE STYLE (2026-09-23, late) — the design pass was REJECTED

The user: *"no i m not satisfied shall we go with like older one refer
quotation entry"*. Rendering `QuotationEntry` beside it settled it — the
quotation screen is FLAT: no cards, no grey page, bold labels LEFT of every
field, groups with empty titles, three columns of form rows, then the grid,
then charges | terms | totals, then the verb bar. My cards + caption
typography were a one-off that matched nothing else in the app.

`sale_bill_entry.ui` was regenerated AGAIN, this time from the ORIGINAL
(`scratchpad/sale_bill_entry.ui.bak`, 12:32), keeping its stylesheet verbatim,
with only what the code needs folded in:

* **grpCustomer** — customer + `+` + Edit-for-this-bill, name, address,
  place+PIN, phone, GSTIN, POS state, beat  (BILL-TO is back on the MAIN
  screen — the user picked that over the popup when asked);
* **grpBillInfo** — bill no, date, term, **Their Ref**, **Ref Date**, **Due**,
  category, price level, salesman, agent;
* **grpCredit** — the original credit form, untouched;
* **grpSources** — "Lines drawn from" + `lblSourceList` (4th group);
* under them: the option ticks, then `lblFacts1` / `lblFacts2`;
* bottom row: charges | **grpRemarks** (remarks only) | grpTotals
  (+ **Charges** and **Bill Amount**);
* verb bar gains **Post** / **Post & Print**, loses Copy (it is a quick icon);
* quick strip: icons only, 44×38, name + key in the tooltip;
* `frameShipping` (hidden, root row 7) = crew + transport slot + terms → the
  popup. `grpBillTo` and `frameHdrFields` are GONE; the read-only switch and
  `m_warnings`' row (back to 4) were updated with them.

**Traps this round:** copying a layout AND a widget it contains duplicates the
widget (`chkQuickDelivery` appeared twice — check with a Counter over
`iter('widget')` after every regeneration); regenerating from the original
also restores its "coming soon" `enabled=false` flags, so `btnQuickShipping`,
`btnQuickEInvoice`, `btnQuickEWay` had to be re-enabled; a titled QGroupBox
needs `topMargin >= 16` or app.qss's 5px padding lets the title sit on the
first row.

## THE THREE-COLUMN FORM (2026-09-23, the user's column spec)

Top row, four groups — the user dictated columns 1-3:

* **col 1 grpCustomer** — Beat · Customer (+ Edit for this bill) · Name ·
  Address · Place + PIN · Phone
* **col 2 grpDelivery** — Salesman · Loadman · Driver · Vehicle No ·
  Packed By · Supervisor · Agent   (the crew came BACK from the popup)
* **col 3 grpBillInfo** — Bill No · Bill Date · Term · Their Ref · Ref Date ·
  Price Level · Due
* **col 4 grpCredit** — unchanged
* then the option ticks (incl. Quick Del), then lblFacts1 / lblFacts2

`lblSourceList` now sits in the identity strip, right after the title.
`btnQuickAddCust` is an icon on the quick strip (person_add).
**GSTIN, POS State and Category left the form** — parked hidden inside
`frameShipping` so the code still writes and reads the printed snapshot; the
GSTIN NUMBER was added to facts line 1 so it stays readable. A walk-in cannot
type a GSTIN any more — raise it if that matters.
The popup is now transport band + terms only.

**Traps, again:** appending a widget element to a second parent in ElementTree
leaves it in the first — always re-parse and Counter the names; rebuilding a
form layout from an explicit row list silently DROPS anything not listed
(`chkQuickDelivery` vanished and broke the build); and a layout's `<property>`
children come first, so inserting "after the first item" by element index puts
the widget among the properties (the source list rendered before the title).

## VERB BAR TRIM (2026-09-25, user: "too many buttons")

* **Tender route ⇒ the bar shows Tender only** — the tender dialog has its own
  Save / Save & Print, so the bill bar repeats neither (F6 still opens tender
  with print). Plain route: Save/Save draft F5 + one print button (Save & Print,
  or Post & Print when auto_post is off).
* **`btnPost` and `btnSaveAndPrint` are DELETED from the .ui** — posting without
  printing is Ctrl+Shift+Enter only. Amend on the tender route keeps
  Tender + "Save changes" (an edit that doesn't touch payment).
* **Cancel bill is `ui->btnCancelBill` in `hboxBottomBar`** (after Delete), not
  the identity strip. **Edit remarks button + `editRemarks()` removed** —
  remarks change through Edit/amend; `/bills/update-remarks` is unused now.
* F6 with auto_post routes to `saveBill(true)` (check-first `postNewBill`),
  and `postBill` no longer refuses a NEW bill for `!m_canPost` (rights arrive
  at save) — that refusal made the enabled print button dead on a new bill.

## AMEND 500 (2026-09-25) — two client bugs fixed, one server suspect (notes 45)

* **Posted set-offs reloaded SHORT:** the adjust panel capped a restored credit
  at the advance's LIVE pending, which on a posted bill already excludes what
  this bill holds (300 held → 200 sent). Now `PartyAdjustPanel::setHeldByDocument()`
  (loadBill passes the adjustments when sbStatus POSTED, reset/copy pass {})
  adds them back and rebuilds credits the bill used up entirely.
* **sbPaidAmt counted set-offs twice:** `m_voucherParam.paidAmount` is COUNTER
  money only (onTenderValues); buildSavePayload adds adjustAmt. The load now
  subtracts sbAdvanceAmt + sbNoteAdjAmt from sbPaidAmt.
* Either made an amend payload inconsistent → server 500 (bil00686).
* **Still open, server:** bil00687 (220 loyalty pts, consistent payload) 500'd;
  every no-points bill amends fine. Suspect: re-earn writes EARN row 1 over the
  live original → ux_lld_src_row. `share/sales/notes-45-amend-500-loyalty.md`.
* Tools: `run-full.sh amend <billno> [dry]` (dry = dump payload to
  /tmp/claude-1000/amend-payload.json, nothing sent) and `amendmatrix [case]`
  (posts fresh ZT bills and amends each). Both run BEFORE cleanupEarlierRuns.

## BILL CHEQUE NOT REGISTERED (2026-09-25) — server gap, notes 46

A CHEQUE (type 5) tender on a sale bill posts DR Cheques In Hand / CR party but
writes NO acc_pdc_register row (plan-backend-sales.md:401 treats it like cash;
only the sale ORDER's order-pdc-posting.helper.ts registers cheques). So menu 51
and the receipt's pending-cheques panel never see it, and a bounce could not
re-open the bill (C4 finds bills via abj_tender_id; a bill's own tender writes
none). Client already sends ref/bank/date/tdIsPdc — no client change.

## LOYALTY REDEEM refused (2026-09-25) — client fix

SALES_LOYALTY_CAP "redeems points but names no loyalty member": the validate
guard (bill-lifecycle step 11) needs `sbLoyaltyMemberId` on the payload for
any LOYALTY tender, although the post itself would resolveMember() from the
party. The client read party-context `loyalty` but dropped `memberId`. It is
now `m_party.loyaltyMemberId`, sent as sbLoyaltyMemberId (null when the
customer is not a member). No member creation needed: a customer with points IS a member.
**Notes 46 DEPLOYED and verified the same day:** backfill, post (HELD), amend
(old row CANCELLED + new row HELD), cancel, and deposit → bounce re-opening the
bill all work; C2 ledger = register. Harness modes: `chequecheck`,
`chequebounce` in tests/salebill/run-full.sh (post fresh ZT bills).

## 2026-09-25 — prompts + Clear
* Every text prompt on the bill is `AppMessage::askText` (cancel reason with
  presets Keyed wrong / Wrong customer / Duplicate bill / Customer cancelled;
  amend remark required; order-line cancel; Part-B vehicle + reason). The two
  % prompts (QInputDialog::getDouble) remain — askText is text-only.
* **Clear - F7 is visible in EVERY state** and means "start a new bill": an
  edit being made is discarded after a confirm, and a loaded bill clears with no
  prompt. "Discard changes" (reload the same bill) is gone.
* The test modal-closer fills askText prompts: it picks preset index 1, or types
  into an enabled empty QLineEdit.

## TENDER DIALOG REWORK (2026-09-25, user's list)
* **Fixed width** (868): the detail column is reserved even when blank
  (`m_blankPanel` + spacer kPanelWidth 560 + lockWidth()). showPanel no longer
  resizes. The old narrow/wide resize made the window jump on every row.
* **Enter on a row goes to its right-hand panel** (eventFilter on tableTender →
  focusDetail): UPI → reference, card/cheque → bank, TEMP_CR → name, RRN →
  the credits' Adjust cell (PartyAdjustPanel::focusEntry), LOYALTY → redeem
  amount. Keying an amount + Enter also moves on. **RRN/LOYALTY cells are not
  editable**; typing on them jumps to the panel and the keystroke follows
  (postEvent). TEMP_CR no longer steals focus on mere selection.
* UPI UTR optional (placeholder "optional"); loyalty fields 220px min.
* The "not on loyalty scheme" hint was wrong for a MEMBER whose customer master
  has allow_loyalty off → it now says so.
* **Cheque details** (drawer default bill-to name, branch, IFSC, MICR) live in
  the panel under Cheque No → SaleTenderLine::Cheque → `cheque` on the row,
  sent since notes 48 deployed 2026-09-25 (`kServerHasChequeDetails = true`); verified post / reload / amend / cancel (bil00723, after the amend-path fix).
* The adjust tender type is named **RRN** in the masters, so the user calls
  the credits list the "RRN table".
* Harness: `run-full.sh tendershot <dir>` renders one PNG per row and reports
  where Enter lands. That is how to see the dialog (Wayland blocks capture).
* Re-tender still has no button (Ctrl+F6 only); the user did not pick one.

## HOLD / AUTOSAVE restore fixed (2026-09-25)
Restores painted lines through fillLoadedItemRow, which reads the /bills/get
DISPLAY keys (sbiItemName, sbiUnitName, sbiGodownName, sbiGroupId). The local
snapshot was the bare save payload (ids only), so restored lines had a blank
item name, UOM and godown and no item group. `buildLocalSnapshot()` (used by
hold AND autosave) now adds them per line, matched by sbiLineNo = row + 1.
Holds saved BEFORE the fix still restore blank names. Hold / Pick Held moved to
just before Close in hboxBottomBar.
Harness `holdcheck` diffs every grid cell before and after both paths; 0 real
differences now. NOTE: the harness closer answers No to "Recover Bill", and a
normal close DELETES the autosave file, so the test copies the file and calls
restoreFromSnapshot directly.
