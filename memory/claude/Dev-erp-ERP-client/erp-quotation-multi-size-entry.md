---
name: erp-quotation-multi-size-entry
description: "Size Entry dialog on Quotation + Sale Bill (2026-09-16/17) — one item in several sizes as several lines; shared generic module, no size master exists, F4 was taken, no migration needed"
metadata: 
  node_type: memory
  type: project
  originSessionId: a756e224-1e96-4bb6-95f6-738f98b7336b
  modified: 2026-10-12T10:22:01.843Z
---

Built 2026-09-16 on `features/sales/quotation`. **Picking an item in Description opens the
dialog by itself** (the Enter chain lands on Size, and that is the moment sizes get keyed);
double-click or **F3** on the row re-opens it. OK replaces that item's run of rows with one line
per size — same description on each.

**Reworked 2026-10-12 at the user's request:** the dialog now keys L/W/T/Pcs **+ Rate** per row
(read-only `= CFT` between them). Enter walks L→W→T→Pcs→Rate→next row's L, and **only Enter on
the LAST row's Rate adds a row** — Alt+R was removed (the "+ Add size" button stays, no hint). A
new row carries **only L and W** from the row above; T/Pcs blank, Rate = group base rate. That
row is `pristine` until keyed: `keyedSizeRows` drops it from OK/count/total (an L×W alone works
out to a CFT and would otherwise save as a bogus line), and Enter on a pristine last Rate
focuses OK instead of adding (so held Enter can't breed rows). Qty is still not keyed here.
As of that date only Sale Bill + Testbill mount the dialog; Quotation no longer imports it.

**Reworked again 2026-10-14 to match the user's paper sheet:** the dialog is now a SHEET of two
row kinds (`SizeSheetRow` in `quotation.sizes.ts`): an **L × W "face"** line with its own rate,
then **T – Pcs "cut"** lines under it (each a size = one bill line, saved as `L*W*T*Pcs`). Since 2026-10-14 (user: "hide the rate field") a
cut has NO rate at all — every cut sells at its face's rate (`rate`/`rateTouched` removed from
`SizeCutRow`). Faces are never persisted — on reopen `openSizeSheet` regroups consecutive lines
with equal L, W AND rate, so a line at another rate gets its own face line (never silently
re-rated). Enter flow: L→W→face Rate→T→Pcs→Rate→new cut's T; Enter on an EMPTY last
cut's T converts it to a new face; Enter on an empty last face's L → OK. A face keyed with no
cut under it is refused; a factor after a blank one is refused (joinSizeFactors drops blanks,
so `5**1*2` would save as `5*1*2`). The just-picked line's seed cut is `pristine` but holds
`source` and is never converted to a face (keeps line identity/srcDoc trail). The save path
is unchanged: `sizeSheetEntries` → plain `SizeEntryRow[]` → `applySizeEntry`. **`+`** (main row or keypad, no Ctrl/Alt) = new L × W — user's choice, replaced Alt+L; **Alt+T** = new T – Pcs (no button any more, user removed it). Both go to the end of the FOCUSED row's L × W group (`tr[data-size-sheet-row]`); the trailing empty Enter-opened cut is reused/replaced, not stacked. The "+ L × W - Press +" button appends at the sheet end. Enter on a cut's **Pcs** finishes the line (there is no cut Rate box) — face Rate stays in the walk. Finishing is GROUP-relative (`finishCut`): the last line of an L × W opens a cut under it even mid-sheet (walking on selected the next face's L and the next size overwrote it); Enter on that empty T mid-sheet drops it and moves to the next face. The face row's `= CFT`
cell shows its cuts' total (`sizeFaceTotals`, bold), left of the face Rate; cut rows show NO CFT
(user: totals only). The # column numbers FACE rows (1, 2, …); cuts are unnumbered. Pair columns are
`width: 1%` (shrink to the fixed-width boxes) so L × W and T – Pcs sit ~12px apart. Panel width
u(46) (user asked for a bigger popup the same day).

**Print side:** that draft became the company-scoped "Saleinvoice A5" (now rev 2, published 2026-10-15): lines group by item, then by FACE (`face_key` = L×W + rate), cuts print `t_pcs`. Columns `size_face`/`size_cut` are gone (`lxw`/`t_pcs`). See [[erp-saleinvoice-a5-print-template]].

Files: `quotation.sizes.ts` (pure: grouping, validation, `applySizeEntry`) +
`quotation.sizes.test.ts`, `components/size-entry-modal.tsx`, `lineSizesApplied` in
`store/slices/quotationSlice.ts`, `lineGroupKey` on `DraftLine`.

**Ported to Sale Bill 2026-09-17**, reusing the same module and dialog rather than copying
them: both are generic over the line type (`SizeGroupLine`), since `SaleBillDraftLine` IS the
quotation's `DraftLine` plus the bill's columns. Added: `lineSizesApplied` in
`store/slices/saleBillSlice.ts`, the wiring in `components/sale-bill-entry-view.tsx`, and
`salebill.sizes.test.ts` for the one rule that is only true there (below).

**The screens differ in exactly one thing: what a NEW row must shed.** `applySizeEntry` takes
the screen's own Alt+R copier for that — `duplicateDraftLine` on the quotation,
`duplicateBillDraftLine` on the bill, which also drops `sbiId` and the whole `srcDoc*` trail.
Without it, three sizes cut from an imported line would make one order line look billed three
times and hand the operator an uneditable `orderQtyLocked` qty cell. The row that came from the
document keeps itself, cap and trail included — it IS that order line, now sized.

Sale Bill specifics: the route opens on a browse LIST (+ Add enters the form), an unsaved draft
pops a "Recover the bill this counter was on?" prompt (Start fresh / Bring it back), a cash bill
opens "Settle Bill" (OK & Save) before it posts, and **table 22 configures no Size column at
all** — it is injected by `SALE_BILL_INJECTED_ITEM_COLUMNS`, which is what the auto-open guard
sees. `sale_bill_item` has only `pk_sale_bill_item` (sbi_id, sbi_acc_year): no line-number
uniqueness at all, so multiple lines per item are trivially legal.

Three things the original spec assumed that are not true of this codebase:

- **There is no size master**, client or server (`grep` finds no model/table). `sqi_size` is
  varchar(50) free text holding dimensions `L*W*T*Pcs`, and `cubicFeetFromSize()` turns them
  into the CFT that becomes the line's Bill Qty. So the dialog's Size column is the same four
  boxes the grid's Size cell uses, and Qty follows the CFT until keyed.
- **F4 was already the unit switcher** on this grid (the header hint says so). F3 was free.
- **`ux_sqi_quote_line` needed no migration.** Live definition:
  `UNIQUE (sqi_quote_id, sqi_acc_year, sqi_line_no) WHERE sqi_is_deleted = false`. It is not on
  the item or the size, so several lines of one item are already legal.

Two rules worth keeping if this is ported to Sale Order / Sale Bill:

- `applySizeEntry` **carries an existing row across by object**, so it keeps its `sqiId` (next
  save UPDATEs rather than inserting a twin) and every column the dialog does not show. Only
  genuinely new rows are cut from the group's first line. (Shrinking a group is fine too —
  dropped lines soft-delete normally, see [[erp-audit-delete-action-blocks-soft-delete]].)
- **Qty follows the size, and the dialog must not disagree with the grid cell it stands in
  for.** `item-grid.tsx` writes `billQty = cubicFeetFromSize(size)` on every keyed size, so the
  dialog does too: editing a row's dimensions always re-derives its Qty. `qtyTouched` only
  protects a stored quantity at the moment the dialog OPENS (`openSizeEntry` sets it from
  `billQty > 0`) — a just-picked line has nothing to protect and must follow its size, or the
  dialog blocks its own save on a row it just filled.

Since 2026-10-14 **OK** sends the cursor to the Description (`itemLookupFieldKey`) of the row AFTER the sized group (key noted in the screens' `saveSizeEntry` BEFORE the dispatch — group rows may change keys), so the operator picks the next item; **Cancel** keeps the old rule — closing must `focusCell(ITEM_GRID_NAME, rowKey, "itemSize")` on an rAF: its own
autofocus is abandoned on close and focus falls to `<body>`, which leaves nowhere to key a size
by hand and stops F3 (read off the focused cell) re-opening it at all.

`styles.buttonPrimary` is a **modifier, not a button**: used alone it gives colour only, so the
control keeps the browser's default font size and no padding. Always `cx(styles.button,
styles.buttonPrimary)` — every other primary button on these screens does.

`formatQty`/`formatCurrency` blank a zero by design (a grid of "0.00" is unreadable), so a lone
footer total needs its own zero case or it renders as "total  CFT" with a hole in it.

Auto-open is gated on the Size column being visible in the resolved layout — a grid configured
without one is quoting something with no dimensions.

The modal snapshots `lines` in a `useState` initialiser and is mounted only while open — re-reading
them would let any dispatch wipe what the operator keyed. Ctrl+Enter is a **document** capture
listener, not a React `onKeyDown`: deleting a row unmounts the button holding focus, which falls
to `<body>` and a handler on the dialog's markup never sees the key.

Verified 2026-09-16 against the live stack, quotation `Q0043`: two sizes save as two
`sale_quotation_item` rows; reload → Edit → double-click re-groups both from persisted data
(`lineGroupKey` null, item-id fallback); editing the group re-saves with the SAME `sqi_id`s.

Related: [[erp-quotation-entry-port]], [[erp-grid-disabled-input-swallows-mouse]].
