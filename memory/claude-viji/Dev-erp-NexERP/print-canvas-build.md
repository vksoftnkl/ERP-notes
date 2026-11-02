---
name: print-canvas-build
description: Print designer CANVAS in NexERP — the user asked to "split the canvas better", so it follows share/printing/plan-qt-canvas-steps.md (13 small steps); steps 1–4 done 2026-09-29 (editing works); next are the rest of 5–6 (guides, band drag, clipboard, align)
metadata:
  type: project
---

On 2026-09-29 the user said **"split the canva better"**, meaning split the canvas build into smaller pieces.
The split is `share/printing/plan-qt-canvas-steps.md`. It overrides the C-plan's §14 phases: 13 steps, each
one checkable, and **editing + saving from step 4**.

**Key simplification:** the canvas never saves to the server. It edits the Designer's working body.
OK writes it back into the Layout tab, and the Designer's own Save posts it, with every existing rule.

Done (uncommitted), in `src/modules/admin/printing/canvas/`:
1. `canvas_model.*` (QtCore): Band and Element WRAP the QJsonObject (lossless), normaliseBand,
   presets, GridMetrics, rectMm (the one cells→mm point; a GRID line spans x1..x2 INCLUSIVE), paintOrder
   by z (stable), fromVersion/toBody. Test: test_canvas_model 35/35 on the 3 web fixtures (copied
   to tests/printing/fixtures).
2–3. `canvas_scene.*` + `canvas_window.*`: the viewer. The caption gutter is IN the scene at negative x;
   text is drawn in device px; placeholders; crosstab sketch; GRID mono; Ctrl+E samples; zoom +
   Ctrl+wheel. The Designer's Layout tab has `btnOpenCanvas`. Test: test_canvas_viewer 12/12, and
   PRINT_SNAP saves PNGs.

Qt trap: never name a method `slots()`, because it is a Qt macro.

Step 4 was DONE the same day, after the user said "can not design anything, it just opens" (a viewer
alone was not acceptable). Pieces of steps 5–8 were pulled forward:
- `canvas_document.*`: snapshot QUndoStack, selection, moves, rect, insert, delete, band edits; a 0.5 mm
  snap for MOUSE gestures only;
- CanvasView: a state machine (drag, resize with 8 handles, marquee, placing), drop of the
  `application/x-nexerp-canvas-kind` MIME;
- a PaletteButton INSERT column;
- `canvas_properties.*`;
- OK → applied(body) → the Designer's edBody, and the user presses Save.

Test: test_canvas_editing 25/25.

Traps found:
- the VIEWPORT needs setAcceptDrops(true), not just the view;
- a synthetic QDropEvent needs a DragEnter first;
- QTest widget helpers need -DQT_WIDGETS_LIB;
- the canvas opened scrolled to the middle of the page, so it now fits the width and scrolls to the top left;
- tests/printing/run.sh now runs ninja first.

Left: neighbour snap + guides, band height drag, rulers, clipboard/align/distribute/z-order, validator,
expression editor, tree panels, crosstab props. See [[print-designer-parity]], [[print-designer-port-analysis]].

**2026-09-29 (later): the user said "many features missed — can't choose datasets, icons not text on
the left, UI is completely different from the web; refer to the React front end".** Feedback: the Qt
canvas must LOOK and WORK like `nexerp-client/features/print-designer`, not a Qt-flavoured
redesign. Rebuilt to the web's DesignerShell layout:
- a menu bar (File / Edit / View / Insert / Arrange / Help);
- a top bar (◀ Close, name, host label, badges, Save);
- a tool strip of the web's GLYPHS (Insert T {} ─ ▭ 🖼 ||| ▦ ⤓ ⊞, align ⇤ ↔ ⇥ ⤒ ≡ ⤓, distribute ⇹ ⇳,
  ↑z ↓z, ↶ ↷, grid / snap / size / {{ }}, zoom);
- a left dock with Report / Data tabs (`canvas_panels.*`: CanvasReportTree, CanvasDataTree with "Bind a
  dataset…" over /print-render/providers, a field search, and SELECT-list fields that drag onto bands);
- a sectioned property panel ("TEXT · id" header, ▾ sections);
- a status bar cells row;
- the web's colours as a stylesheet.

A dataset bound on the canvas becomes a PROVIDER row on the Designer's Data tab when the canvas saves.
The document gained align / distribute / reorder / copy / cut / paste / duplicate / insertField /
bindDataset. Stylesheet trap: a generic `QDialog#x QWidget {color}` beats `QPushButton#id {color}`,
so the colour goes on the dialog itself.

Step 5 was DONE the same day. `canvas_snap.*` is the pure port of snap.ts; the vectors are in
test_canvas_snap. Guides are drawn as z 6500 lines. The band-edge drag uses a "gesture:" coalesce
key, which merges regardless of the 400 ms window. CanvasRuler sits in a grid around the view.

Float trap: 0.4 mm "ties" are not ties in doubles (|18−18.4| < |8−8.4|). JS behaves the same,
so test ties with 0.5 mm offsets.

Left: the fx expression editor, the problems list / validator, the Format section, crosstab / barcode /
QR / image detail sections, and Preview from the canvas.

Steps 10–11 were DONE the same day:
- `canvas_expression.*` + `canvas_expression_editor.*`: fx on Value / Source / Break when / Print when /
  Group by / band Print when. fx commits through the DOCUMENT by key, not through the line edit, because
  the panel may rebuild while the modal is open;
- `canvas_validate.*`: a problems button in the status bar; errors block canvas Save.

Tests: test_canvas_expression (pure; the 3 fixtures have 0 errors) and test_canvas_editing (fx + problems).

Traps:
- a private `accept(Completion)` HIDES QDialog::accept();
- a canvas test must load the fixture WITH its datasets, or every root is "unknown".

Left: the Format section, crosstab / barcode / QR / image detail sections, Preview from the canvas,
import/export, remembered view settings.

ALL 13 STEPS DONE 2026-09-29, after the user said "complete, don't skip".
- `canvas_properties.cpp` was rewritten to the web's sections field for field, including the whole crosstab, field
  aggregates, format presets and line end points.
- The Designer's `openPreview(layout, unsaved, over)` serves both its own Preview and the canvas's
  previewRequested.
- File → Import / Export bands.
- QSettings group `printCanvas`.
- The `?` sheet.

Tests: 8 printing suites all green. The fx closures COPY their commit functor before exec(), because the panel
can be rebuilt during the modal.

**2026-09-29 (late) — web-parity pass, after "the menu is in the right side corner; do a double against the web".**
- ROOT CAUSE of the corner menu: a Qt stylesheet cascades into child top-level windows, so the shell's
  menu_window.ui QMenuBar (64 px white items) and dark QMenu reached the canvas. Items taller than the bar
  are all folded into the » extension at the far right. kStyle now resets every property the shell sets
  (QMenuBar/QMenu), plus setNativeMenuBar(false). Any new child window of a screen needs the same reset.
- A 44-item audit (web vs Qt) was closed item for item:
  - one app-level key filter mirrors DesignerShell handleKeyDown; menu actions only SHOW their keys
    (WidgetShortcut);
  - canvas Save = the Designer saves the revision and the canvas STAYS OPEN (saveRequested → save(Draft, done)
    → saveFinished). applyCanvasBody also writes the paper to the Template tab;
  - read-only canvases are interactive; only Save is refused;
  - the property panel was rewritten as the web grid: 44/56 rows, wide expression textareas that commit live
    (m_selfEdit stops the rebuild), Mixed values, and editable Paper/Margins;
  - captions moved OUT of the scene into CanvasGutter (148 px, sticky); bands have no gap; ▾ collapses a band
    to a 4 px strip; the foot row sits on the sheet;
  - also: the web's sample resolver, crosstab sketch, 3 crosstab validate warnings, and a one-window
    PreviewDialog that auto-renders.
- Qt trap again: `slots` is a macro, so never use it as a local name.
- Tests: test_canvas_editing uses fieldFor() over propRow / propName; all 8 printing suites are green.
