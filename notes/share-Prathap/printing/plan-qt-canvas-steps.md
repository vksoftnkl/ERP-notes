# Print designer canvas in NexERP: the build, split into small steps

**Written:** 2026-09-29. **Overrides:** the phase table in
`nexerp-client/features/print-designer/QT_DESKTOP_PLAN.md` §14 (C0–C5). Everything else
in that plan still holds: the model, the rules, the web quirks to fix, and the keyboard map.

**Why split it again.** The C-plan's phases are 1–3 weeks each, and editing arrives only in C2,
while saving arrives only in C4. The steps below are sized so that each one ends with something you
can open and check, and **editing plus saving work from step 4**.

## The one simplification

The canvas does **not** save to the server itself. It edits the Designer's working draft:

- Opening the canvas hands it the Layout tab's body.
- **OK** writes the edited bands back into that body. The Designer's own **Save** then posts it, with
  every rule it already has (pointer echo, dataset ids, publish).

So there is **no second save path**, and the read-only rule is the Designer's too. A published
revision opens the canvas read-only, and the move to editing is the Designer's New draft.

## Steps

| # | Step | You can check it by | Status |
| --- | --- | --- | --- |
| 1 | **Model.** Band and element wrappers over QJsonObject (unknown keys survive), normaliseBand, paper presets, units and GRID cell scale, version → definition (paper and datasets come from the revision, bands from the body) and back | the three web fixtures + SaleInvoiceA5 load and save back semantically equal (test) | ✅ 2026-09-29, `canvas/canvas_model.*`, test_canvas_model 35/35 (bands save back identical, key for key) |
| 2 | **Viewer (graphic).** "Open canvas" on the Layout tab; a window with the page, bands in array order, a caption gutter, zoom; TEXT / FIELD / LINE / RECT drawn in z order | SaleInvoiceA5 opens and looks like the paper | ✅ 2026-09-29, `canvas/canvas_scene.*` + `canvas_window.*`, "Open the canvas…" on the Layout tab; captions sit in the scene's gutter, so zoom and scroll cannot misalign them |
| 3 | **Viewer (complete).** IMAGE / BARCODE / QRCODE / PAGEBREAK placeholders, the CROSSTAB sketch, GRID mode (T80 receipt), rulers, show expressions on/off | all three fixtures draw | ✅ except the rulers (moved to step 5): placeholders, crosstab sketch, GRID (T80), show expressions / samples (Ctrl+E), z order; test_canvas_viewer 12/12 |
| 4 | **Select, move, delete, undo — and OK back to the Layout tab** | move a field, OK, Save in the Designer, then print: the field moved | ✅ 2026-09-29: `canvas_document.*` (snapshot undo, 50 deep, nudges merge), click / Ctrl-click / marquee select, drag-move, Delete, Undo/Redo, OK → Layout tab. test_canvas_editing 25/25 with real mouse, key and drop events |
| 5 | Resize handles, snapping and guides, arrow-key nudge, band height drag | | ✅ 2026-09-29: 8 resize handles, `canvas_snap.*` (a port of snap.ts: band / margin / neighbour targets, the web's tie-break, grid fallback, guides hidden when clamped; resize snaps the moving edges only), guide lines while dragging, nudge, band bottom-edge drag (one undo step per drag, however slow; snaps to the grid), rulers across (mm, or columns) and down (0 per band). Tests: test_canvas_snap (the web's vectors) + test_canvas_editing |
| 6 | Insert tool (click to place each kind), copy / cut / paste / duplicate, align, distribute, z-order | | ✅ 2026-09-29 |
| 7 | Property panel I: position, value, font, colours, TEXT / FIELD / LINE / RECT | | ✅ 2026-09-29: the web's sections field for field (Position incl. padding + id rename; Font incl. ellipsis, blank when zero, stroke; Format presets + custom mask; Visibility) |
| 8 | Property panel II: band settings, the other kinds, paper shown read-only | | ✅ 2026-09-29: Field aggregate (+ its dataset), Line (+ end points), Rectangle, Image, Barcode, QR code, Page break; Band (order, type, spacing, group level, keep-with-last-detail); Page setup read-only |
| 9 | Report tree and data tree; drag a dataset field onto a band | | ✅ 2026-09-29 |
| 10 | Validator + problems list (the web's rules + the server's limits); errors block OK | | ✅ 2026-09-29: `canvas_validate.*` (validate.ts + the server's 60 / 500 / 20 / id / font / z limits + a \|coalesce+ warning); status-bar count, clickable list, errors block Save; the 3 web fixtures validate with 0 errors |
| 11 | Expression editor with completion and lint | | ✅ 2026-09-29: `canvas_expression.*` (lint, roots, completion; the SERVER's ctx / sys members and transform signatures) + `canvas_expression_editor.*` (fx dialog: highlighting, Tab completion, quick inserts, transform list, live lint); fx on every expression field |
| 12 | Crosstab properties | | ✅ 2026-09-29: Crosstab (levels, measures), Columns and rows, Totals and appearance |
| 13 | Preview from the canvas, import / export bands, remembered view settings, shortcut sheet | | ✅ 2026-09-29: Preview renders the canvas's unsaved bands through the Designer's preview dialog; File → Import / Export bands (checked, one undo step); QSettings printCanvas/*; ? sheet |

## Look and feel (user, 2026-09-29)

The canvas must look and work like the React designer (`nexerp-client/features/print-designer`,
DesignerShell): menu bar, top bar, an icon tool strip, a Report / Data dock, a sectioned property
panel, and a status bar, in the web's colours. Choosing datasets happens on the **Data tab** (bind a provider,
search fields, drag a field onto a band), as on the web. Done 2026-09-29, along with align, distribute,
z-order, clipboard and field drag (the rest of step 6, and step 9).

## Order rules

- Each step builds, has its own test in `tests/printing/`, and leaves the previous steps working.
- Step 1 is pure (QtCore only), so the rest can lean on it.
- Web bugs from the C-plan §11 and the check file §2c are fixed in the step that touches that code,
  never copied.
