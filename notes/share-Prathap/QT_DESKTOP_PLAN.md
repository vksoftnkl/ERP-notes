# Print designer canvas for the Qt desktop app — port plan

**Audience:** the C++/Qt developers building the ERP desktop client.
**Written:** 2026-09-29, against the web client at `dev` (commit `7b5e22d`).

This plan covers the **band canvas**: the visual editor for a template revision's
`ptvBody`. Read the master plan first:
[`features/printing/QT_DESKTOP_PLAN.md`](../printing/QT_DESKTOP_PLAN.md). It owns the
shared decisions (server-side rendering, Qt 6.8, async API client, parity vectors), the
REST contract, and the template designer screen that opens this canvas.

---

## 1. Scope

The canvas is a desktop-style report designer, deliberately modelled on QtRPT and Qt
Designer: a menu bar over tool strips, docked panels, band captions in a gutter beside the
page, a two-column property grid, and a status bar. Users arrive from the old Qt desktop
app. On the desktop this is native ground, so the Qt version should feel like Qt
Designer, not like a web page.

**Port:**
- the canvas shell, menus, tool strip, docks and status bar;
- the band and element model, GRAPHIC (millimetres) and GRID (character cells) modes;
- selection, marquee, drag, resize, snapping, guides, nudging, band height drag;
- undo and redo with coalescing;
- the report tree, the dataset tree with field drag, the property grid;
- the expression editor, the problem list, the preview dialog;
- the host contract that `features/printing` uses to open, save and preview a revision.

**Do not port** (dead in the web client):
- `list/page.tsx`, `new-page.tsx`, `designer-page.tsx`, `PaperSetupDialog`,
  `RevisionsDrawer`, `EditLayoutLink`, `createStarterDefinition`;
- the standalone (non-hosted) modes of `DesignerShell`, `useTemplateSave` and
  `useTemplateActions`;
- everything in `api/printTemplateApi.ts`. Every route in it is `/reports/*`, which does
  not exist on the server. Only its types are used;
- the menu items that depend on it: Publish as default, Duplicate template, Export JSON
  (server) and Version history. The printing module's template designer owns revisions
  and publishing.

---

## 2. Architecture

```
src/printdesigner/
  model/                     printdesigner_model (Qt6::Core only)
    Definition.{h,cpp}       JSON-backed Definition / Paper / Band / Element (section 3)
    Vocabulary.{h,cpp}       band types, element kinds, presets, transforms, fonts, zoom levels
    Units.{h,cpp}            port of lib/units.ts
    GridScale.{h,cpp}        port of lib/grid.ts (CanvasScale)
    Geometry.{h,cpp}         port of lib/geometry.ts
    Snap.{h,cpp}             port of lib/snap.ts
    Defaults.{h,cpp}         port of lib/defaults.ts
    Expression.{h,cpp}       port of lib/expression.ts (lint + completion; no evaluation)
    Validate.{h,cpp}         port of lib/validate.ts
    SampleData.{h,cpp}       port of lib/sample-data.ts
  document/
    DesignerDocument.{h,cpp} the definition + selection + QUndoStack; every edit goes here
    Commands.{h,cpp}         one QUndoCommand type, snapshot-based (section 5)
    Clipboard.{h,cpp}        in-process buffer, plus the system clipboard as JSON
  canvas/
    CanvasGeometry.{h,cpp}   band offsets, the one source of vertical layout
    CanvasScene.{h,cpp}      QGraphicsScene: page, band bodies, elements, overlay
    CanvasView.{h,cpp}       QGraphicsView: zoom, scroll, mouse state machine
    items/                   PageItem, BandBodyItem, ElementItem + one painter per kind,
                             SelectionOverlayItem, GuideItem, MarqueeItem
    HRuler.{h,cpp}  VRulerGutter.{h,cpp}
  panels/
    ReportTree.{h,cpp}  DatasetTree.{h,cpp}  ProblemsPanel.{h,cpp}
    PropertyGrid.{h,cpp}     the two-column grid widget
    sections/                PositionSection, KindSection, CrosstabSections, FontSection,
                             FormatSection, VisibilitySection, BandSection, PaperSection
  dialogs/
    ExpressionEditor.{h,cpp}  PreviewDialog.{h,cpp}  ShortcutSheet.{h,cpp}
  DesignerWindow.{h,cpp}     QMainWindow: menus, tool strip, docks, status bar
  CanvasHost.h               the host interface (section 9)
tests/printdesigner/
```

`model/` has no widget code and is where the parity tests aim. `document/` is the only
place that changes a definition. Widgets read the document and call its methods; they
never edit JSON themselves.

**Window.** Open the canvas as its own top-level `DesignerWindow` (a `QMainWindow`), one
per revision, like Qt Designer. The web version lives inside the app shell and refuses to
run below 1279 px wide. A separate window removes that limit and lets the operator keep
the template designer open beside it.

---

## 3. The model

### 3.1 JSON-backed, lossless

`ptvBody` is JSON text that the web client, the server's zod schema and older saves all
shape slightly differently. Crosstab `extraRowBys`, `extraColumnBys`, `extraMeasures` and
`measureLabel` are optional because older bodies lack them. `meta` is free-form. The
server may add fields before the desktop learns them.

So the model **wraps `QJsonObject` rather than copying into structs.** Each class keeps
its JSON and exposes typed accessors with the defaults the schema applies:

```cpp
class Element {                     // implicitly shared, cheap to copy
public:
    QString id() const;
    QString kind() const;           // TEXT FIELD LINE RECT IMAGE BARCODE QRCODE PAGEBREAK CROSSTAB
    double x() const;  double y() const;
    std::optional<double> w() const;  std::optional<double> h() const;
    std::optional<int> col() const;   std::optional<int> row() const;  std::optional<int> cols() const;
    int z() const;
    QString value() const;          // TEXT, FIELD, BARCODE, QRCODE
    // ...one accessor per field in types/template-definition.ts
    Element patched(const QJsonObject &patch) const;   // dotted keys: "font.size", "style.fill"
    const QJsonObject &json() const;
};
```

Unknown keys survive every edit because nothing ever rebuilds an object from scratch.
`Band` holds its own JSON plus a `QVector<Element>`. `Definition` holds `schemaVersion`,
`layoutMode`, `meta`, `paper`, `datasets` and `bands`.

The field list, per kind, is `types/template-definition.ts`. Treat that file as the spec.

### 3.2 Four contract facts the field names get wrong

1. **`x` is page-relative, `y` is band-relative.** The engine adds no margin to `x`. The
   shipped GST invoice has 8 mm margins and a frame at `x: 8, w: 194`. Draw every band at
   full page width and clamp `x` to the page, not the printable area.
2. **In GRID mode, `x`, `y`, `w` and `h` hold character cells**, mirroring `col`, `row`
   and `cols`. Treating them as millimetres collapses a 48-column receipt into 3 mm.
3. **Bands print in declaration order, not by type.** That is how one invoice carries two
   DETAIL bands: item lines, then the HSN tax summary. Never sort bands by type.
4. **Elements paint in ascending `z`, ties in array order.** The server's layout engine
   sorts each band's elements by `z` with a stable sort. The web canvas ignores `z` and
   paints in array order, which can disagree with the print. The Qt canvas follows the
   server.

### 3.3 Host-owned parts

The body supplies only **bands**. Paper comes from the version's `ptv*` columns and
datasets come from the version's dataset rows (`CanvasBridge::toTemplateDefinition` in
the printing plan). Paper and dataset edits made on the canvas are written into the body
but overwritten from the version on the next load.

Two consequences for the UI:

- Label the Paper section "From the Template tab". Either make it read-only in hosted mode
  or warn that changes there only last until reload. The recommendation is read-only.
- The dataset tree may bind a dataset the host never saves, and an element bound to it
  prints nothing. In hosted mode, do not offer "Bind a dataset"; list only the version's
  datasets.

### 3.4 Stored bodies that are not canvas bodies

A stored body such as `{"bands":[{"kind":"HEADER"}]}` is legal on the server and is not a
canvas definition. The web client crashed on it (`band.elements is not iterable`). Load
every band through a `normaliseBand` step: drop unknown band types, fill defaults, keep a
valid `printOn`, force `elements` to an array. Fall back to the empty definition
(PAGE_HEADER + PAGE_FOOTER) when nothing usable remains.

---

## 4. Canvas

### 4.1 Units and scale

| Constant | Value |
| --- | --- |
| screen density at 100% zoom | 96 px per inch, so 1 mm = 96 / 25.4 px and 1 pt = 96 / 72 px |
| zoom levels | 50, 75, 100, 125, 150, 200 % (default 100) |
| model rounding | 0.01 mm |

**Scene units are millimetres in both modes.** A `GridScale {x, y}` gives millimetres per
canvas unit: `{1, 1}` in GRAPHIC, and `{cellWidthMm, lineHeightMm}` in GRID. Convert at
exactly one place, when reading an element's rect for the scene and when writing a moved
rect back to the model.

- Cell width is `25.4 / cpi` when the paper preset has a CPI, otherwise printable width
  divided by columns.
- Line height is `25.4 / 6` (6 lines per inch).
- Columns come from `paper.columns`, else the preset's columns, else 80.

The view transform is `scale(zoom × 96 / 25.4)`. Offer "true size" later as an option,
using the screen's physical DPI.

### 4.2 Layout on screen

The web client puts the ruler, the caption gutter and the page into one CSS grid, one row
per band, so they cannot drift apart. The Qt equivalent is a single `CanvasGeometry`
object that computes each band's top and height in scene millimetres. The scene, the
vertical ruler and the caption gutter all read it.

| Part | Qt piece | Web size |
| --- | --- | --- |
| horizontal ruler | `HRuler` widget above the view, follows horizontal scroll and zoom | 18 px |
| vertical ruler + caption gutter | `VRulerGutter` widget left of the view, follows vertical scroll and zoom | 18 px + 148 px |
| page and band bodies | items in `CanvasScene` | page width × sum of band heights |

- **Page:** one white rectangle with a drop shadow behind all bands. Left and right
  margins are dashed guide lines. Top and bottom margins are not drawn.
- **Band body, GRAPHIC:** starts at the page's left edge, full page width, `heightMm`
  tall.
- **Band body, GRID:** inset by the left margin, `columns` cells wide, `heightRows` rows
  tall (at least 1).
- **Grid background:** dots every `gridMm` in GRAPHIC, cell lines in GRID.
- **Total line** after the last band: "X mm of Y mm" or "N rows of M", or "continuous".
  Show "bands exceed the page" in red and a dashed page outline when the heights overflow.
- **Collapse:** a caption's twisty collapses a band to a 4 px strip. This is view state
  and never enters the model.
- **autoGrow:** mark the band's bottom-right corner.

**Rulers.** Horizontal counts millimetres from the page edge, or columns from the margin
in GRID. Vertical restarts at 0 in every band. Minor ticks are 4 px long, major ticks
8 px every 10 units with a label. The minor step is 1 unit if ticks stay 4 px apart, else
5, else 10.

**Band caption.** Clicking selects the band. It shows the label, the size ("12mm" or
"3r") and chips for dataset, autoGrow, keepTogether, keepWithNext, keepWithLastDetail,
printOn (when not all pages) and "conditional".

### 4.3 Drawing elements

Each kind has a painter. The canvas is an **approximation, never a preview**. The real
render comes only from the server.

| Kind | Canvas drawing |
| --- | --- |
| TEXT, FIELD | text aligned per `align` and `vAlign`, padding, font, colour and fill unless they are expressions, wrap or single line |
| LINE | a stroke of `widthPt` (at least 1 px); horizontal when the x span is at least the y span. In GRID, `gridChar` repeated across the span |
| RECT | border, fill, corner radius (no radius in GRID) |
| IMAGE, BARCODE, QRCODE | dashed placeholder labelled "IMAGE contain", "CODE128 + text", "QR M" |
| PAGEBREAK | dashed top rule labelled "PAGE BREAK" or "BREAK IF" |
| CROSSTAB | a sketch: header rows per column level, label columns from `rowHeaderWidthMm`, three sample columns (four with totals), at most 12 rows |

- An element with `visible` set draws at 55% opacity.
- **Show expressions** (Ctrl+E, on by default) shows the raw `{{ … }}`. Off substitutes
  sample values from `SampleData`.
- **Text rendering:** do not draw text through the millimetre-scaled painter. Hinting and
  integer pixel sizes make it jump between zoom levels. Map the element rect to device
  coordinates, reset the painter transform, and draw with a pixel font size of
  `pt × 96 / 72 × zoom`.
- **Fonts:** bundle the server's faces (NotoSans, NotoSansMono, NotoSerif, NotoSansTamil
  from the server's `assets/fonts`) and load them with
  `QFontDatabase::addApplicationFont`, so the canvas measures text as the PDF does.
- **GRID text** uses the mono face at a size that fills one cell per character, one row
  per line.

### 4.4 Selection and editing

Do **not** use `QGraphicsItem::ItemIsMovable`. Every move must go through snapping and
end as one undo command. A `CanvasView` state machine owns the mouse, with the web's five
modes: IDLE, DRAGGING, RESIZING, MARQUEE, PLACING.

| Gesture | Behaviour |
| --- | --- |
| Click an element | Select it. Ctrl or Shift toggles it into the selection, same band only. Another band replaces the selection. An additive click never starts a drag |
| Drag an element | Moves the whole selection if it is part of it, else selects and moves it alone. Never crosses into another band. Live preview while dragging; one command on release. GRID moves in whole cells and does not snap |
| Drag a handle | Single selection only; 8 handles (7 × 7 px, constant screen size). Only the moving edges snap. Minimum 2 mm, holding the opposite edge. QRCODE stays square. LINE keeps its direction. GRID changes `col` and `cols` (see section 11) |
| Drag empty band space | Selects the band and starts a marquee. On release, select elements whose rects **intersect** it. Under 1 unit on both axes it is a click |
| Double-click empty band space | Paste |
| Placing mode | Armed from the tool strip or Insert menu. The next click in any band inserts the element with its top-left at the pointer (whole cells in GRID), selects it and disarms. Crosshair cursor |
| Drop a field | From the dataset tree onto a band: inserts a FIELD at the drop point |
| Band bottom edge (GRAPHIC) | A 5 px strip resizes the band height live, as one coalesced command |
| Ctrl + wheel | Steps zoom. Plain wheel scrolls |

**Clamping** (`Geometry::clampRectToBand`). Horizontal is a hard clamp to the page: `x`
between 0 and page width minus `w`, and `w` at most the page width. Crossing a margin is
allowed and only warns. Vertically only `y ≥ 0` is enforced.

**Snapping** (`Snap`, port exactly, including tie-breaks):
- threshold = 1.5 mm / max(zoom, 0.25);
- vertical targets: band x = 0, band width, width / 2, left margin, width minus right
  margin, and each neighbour's left, right and centre;
- horizontal targets: y = 0, band height, each neighbour's top, bottom and middle;
- test the moving rect's three lines against every target; the smallest correction wins;
  on an exact tie the earlier target wins, so band and margin targets beat neighbours;
- with no target in range, fall back to the grid: snap the left or top edge to a multiple
  of `gridMm`, only within the threshold, and draw no guide;
- draw guides as 1 px lines across the whole band; suppress them when clamping moved the
  snapped rect.

**Nudge.** Arrow keys move 0.5 mm, Shift + arrow 5 mm. In GRID an arrow moves one cell.
Held keys coalesce into one command.

**Selection overlay.** A 1 px union box around the selection. A single selection adds the
8 handles with matching resize cursors. Use the designer's maroon (`#8b1d1d`) and a 14%
fill, as on the web.

---

## 5. Document, commands and undo

`DesignerDocument` owns the `Definition`, the selection, view flags and a `QUndoStack`.

**One command class, snapshot based.** The web client stores immer patches. Qt does not
need them: the model is implicitly shared, so a snapshot of the whole definition costs
only the bands and elements that changed.

```cpp
class DefinitionEdit : public QUndoCommand {
    Definition m_before, m_after;
    QString    m_coalesceKey;       // empty = never merges
    qint64     m_lastMs;            // for the 400 ms window
    int id() const override;        // derived from the key; -1 when empty
    bool mergeWith(const QUndoCommand *next) override;  // same key and within 400 ms of m_lastMs
};
```

- `setUndoLimit(50)`.
- Coalescing is a sliding window: a merge refreshes `m_lastMs`.
- An edit that changes nothing is not pushed.
- Any edit sets dirty. Undo and redo set dirty too, as on the web; use
  `QUndoStack::cleanChanged` only if the owner wants "back to saved means clean".
- Only definition edits enter history. Selection, zoom, grid, snap, panel visibility and
  collapsed bands never do.
- Undo and redo prune the selection to elements that still exist.
- Menu and tool tips read "Undo Move element" from the command text.

Coalesce keys to keep: `nudge`, `band-height-<i>`, `margins`, per-band keys for value,
visible, source, groupBy, band visible, page-break when, and one per crosstab field.

**Edits** (one `DesignerDocument` method each, mirroring the web reducers):

| Group | Edits |
| --- | --- |
| Lifecycle | `seed(definition, datasets, dirty=false)`, `markSaved()` (clears dirty, keeps history), `replaceDefinition()` (clears history) |
| Geometry | `moveSelection(dx, dy)`, `resizeElement(handle, dx, dy)`, `setElementRect(ids, rect)` |
| Properties | `patchElements(ids, patch)`, `renameElement(id, newId)` (refuses empty or duplicate) |
| Create and remove | `addElement(band, kind, at)`, `dropField(band, dataset, field, at)`, `deleteSelection()`, `copy()`, `cut()`, `paste(band)`, `duplicate()` |
| Arrange | `align(mode)`, `distribute(axis)` (3 or more), `reorder(front / forward / backward / back)` |
| Bands | `patchBand`, `setBandHeight`, `addBand(type)` (appends), `removeBand` (refused for the last one), `moveBand(from, to)` |
| Paper and datasets | `applyPaperPreset` (also sets layout mode), `patchPaper`, `patchMargins`, `upsertDataset`, `removeDataset` (also clears bands bound to it) |

Rules worth copying exactly:

- **New ids** are `<kind>_<n>` with the smallest free `n`.
- **Paste and duplicate** regenerate ids and offset by 2 (mm, or cells in GRID).
- **Align:** one element aligns to the band box (0 to page width, not the margins);
  several align to their union box. Modes: left, centre, right, top, middle, bottom.
- **Distribute:** sort by position, then apply equal gaps.
- **Reorder:** front moves to the array end with `z = maxZ + 1` (capped at 1000); back
  moves to the start with `z = minZ − 1` (floored at 0); forward and backward change `z`
  by one and move to the end or start.
- **A new CROSSTAB** binds itself to the first dataset with cardinality "many". A crosstab
  left with an empty dataset makes the whole definition fail to parse on the server, and
  the template stops printing entirely.

**Clipboard.** Keep the web's in-process buffer, and also put the selection on the system
clipboard as JSON under a private MIME type (`application/x-vknex-print-elements`). That
lets an operator copy elements between two open designer windows.

---

## 6. Window, menus and panels

### 6.1 Menus

| Menu | Items |
| --- | --- |
| File | Save (Ctrl+S), Preview… (Ctrl+P, only when the host offers preview), Import bands from file…, Export bands to file… (both new, section 10), Close |
| Edit | Undo, Redo, Cut, Copy, Paste (needs a selected band), Duplicate (Ctrl+D), Delete, Select all in band (Ctrl+A) |
| View | Zoom in, Zoom out, Actual size, Grid (Ctrl+G), Snap to grid (Ctrl+;), Show expressions (Ctrl+E), Data panel, Properties panel |
| Insert | the nine element kinds (arm placing mode; disabled with no band selected), then "Band: <type>" for the nine band types |
| Format | Align left, centre, right, top, middle, bottom; Distribute horizontally, vertically; Bring to front, Bring forward, Send backward, Send to back |
| Help | Keyboard shortcuts (?) |

### 6.2 Tool strip

Insert (nine kinds; the armed one stays checked, clicking it again disarms), Align (six),
Distribute (two), z forward and backward, Undo, Redo, Grid, Snap, grid size (0.5, 1, 2,
5 mm), Show expressions, and a zoom combo. Build it from the same `QAction`s as the menus.

### 6.3 Top bar

Close, the template name (read-only when hosted: "The name lives on the Template tab"),
the host's label, badges ("read only" with the reason as tooltip, "modified"), Save
("Saving…" while busy; its tooltip explains the error count), and Preview.

### 6.4 Left dock: Report and Data tabs

**Report tree** (`QTreeView`):
- a root "Report" row showing the layout mode; clicking it selects the paper;
- one row per band **in array order**, with its dataset and height, collapsible;
- one row per element: kind icon, id, and "cond" when `visible` is set;
- click selects, Ctrl or Shift adds; the band expands when the selection moves into it;
- drag rows to reorder bands (new; the web only has Move up and Move down).

**Data tree:**
- one group per dataset: name, then "cardinality · N" or "provider missing";
- field rows with a label and a type mark (`abc`, `1.0`, `123`, `y/n`, `date`, `time`,
  `{ }`);
- a search box that matches name or label and expands groups with hits;
- field rows drag onto bands with MIME type `application/x-vknex-report-field`.

SQL datasets get their field names from the SELECT list. Provider datasets get none,
because there is no field catalogue endpoint.

A dropped field becomes a FIELD element:
- path `row.<f>` for a "many" dataset, `<dataset>.<f>` for a "one" dataset;
- wrapped as `{{ fmt(path,'#,##0.00') }}` for numbers or `{{ date(path,'dd-MM-yyyy') }}`
  for dates, unless the field declares its own format;
- numbers are right-aligned and 25 mm wide, booleans 12 mm, everything else 40 mm.

### 6.5 Right dock: property grid

Build one `PropertyGrid` widget: a `QTreeView` with collapsible section rows, a
44 / 56 name / value split, 20 px rows and 11 px text. The sections know nothing about
layout; they only describe fields. This is how the web code is split too
(`panels/controls.tsx` against the section files).

Shared behaviour:
- **Mixed values** across a multi-selection show an empty dashed editor with "Mixed". A
  mixed checkbox is tri-state; clicking it sets true.
- Number and text editors **commit on Enter or focus-out**. A blank or invalid number
  reverts. Default number step 0.5.
- Colour: a swatch (`#rrggbb`) plus a text box that also accepts an expression.
- Every expression field has an **fx** button that opens the one `ExpressionEditor`.
- **One edit on N selected elements is one undo command**, not N (section 11).

The grid shows the element sections for a selection, else the band sections for a
selected band, else the paper.

| Section | Fields |
| --- | --- |
| Position, GRAPHIC | X, Y, Width, Height (mm), Layer (z, 0 to 1000), Padding (mm), Element id (single selection). Footnote: "X from the sheet edge; Y from the band top" |
| Position, GRID | Column, Row, Span, Element id. Warn when past the paper's columns |
| Kind: TEXT, FIELD | Value (fx). FIELD only: Aggregate (none, sum, count, avg, min, max), Scope (report, page, group), Accumulate (raw expression, `aggregate.over`). Add an editor for `aggregate.dataset`, which the web lacks |
| Kind: LINE | Thickness (pt), Grid character, Colour. Consider adding endpoint inputs |
| Kind: RECT | Corner radius, Stroke (pt), Stroke colour, Fill |
| Kind: IMAGE | Source (fx), Fit (contain, cover, stretch) |
| Kind: BARCODE | Value (fx), Symbology (CODE128, EAN13, EAN8, UPCA, CODE39, ITF14), Print the digits |
| Kind: QRCODE | Value (fx), Size (mm, at least 2), Error correction (L, M, Q, H) |
| Kind: PAGEBREAK | Break when (fx; empty = always) |
| Crosstab | Dataset ("many" only); Rows + extra row levels (expression, heading, width); Columns + extra column levels; Measure, caption, aggregate, format, blank a zero; extra measures. Level and measure lists edit on a single selection only |
| Crosstab: columns and rows | Column order, Row order (label A to Z, Z to A, largest first, smallest first, dataset order), Max columns, overflow (fold or drop), Folded label, Row label width, Column width (0 = share), Header height, Row height |
| Crosstab: totals and look | Total each row, Total each column, Totals label, Corner caption (fx), Grid lines, Repeat header, Header fill, Font size (at least 4), Bold header |
| Font (TEXT and FIELD only) | Family (NotoSans, NotoSansMono, NotoSerif), Size (6 to 32), Bold, Italic, Underline (show the pressed state), Align, Vertical align, Colour, Fill, Wrap, Ellipsis, Blank when zero, Stroke width |
| Format (collapsed) | Number presets writing `fmt(…)`, date presets writing `date(…)`, a custom mask. It rewrites the value expression only when that is safe (one span, no literal text around it) |
| Visibility (collapsed) | Print when (fx; empty = always) |
| Band | Move up, Move down, Type, Height (mm or rows), Spacing after (rows, 0 to 20), Dataset (repeating bands list only "many" datasets), Group by (fx) and Group level (group bands), Print on, Grow to fit, Never split, Keep with next, Keep with last detail (SUMMARY only), Print when (fx), Remove this band (**confirm first**; the web does not) |
| Paper | Preset (also switches layout mode), Width, Height (0 = continuous), Orientation (a label only; it swaps nothing), Columns and Form length (GRID), four margins, and a footnote with the printable width or the cell size. Read-only when hosted (section 3.3) |

Paper presets: A4, A5, A6, LETTER, T58 (32 columns), T80 (48 columns), DM80, DM96,
DM132, DM137. Load the list from `Vocabulary`, not from literals in the UI.

### 6.6 Status bar

Selection ("TEXT · id", "N elements selected", "Band · label", "No selection");
geometry for a single selection (mm, or col, row, span); the **problems** button ("N
errors" red, "N warnings" amber, "No problems"); "B bands · E elements"; paper ("A4 ·
210×297 mm", "∞" when continuous, or "T80 · 48 cols"); "SNAP · GRID" (upper case when
on); zoom; save state ("Unsaved changes", "Not saved", "Saved <time>").

The problems button opens `ProblemsPanel`. Clicking a problem selects the element, band
or paper it names.

---

## 7. Expressions

Template text holds `{{ … }}` spans of jexl. **The desktop never evaluates them.** It
lints and completes; the server evaluates. Port `lib/expression.ts`:

- spans and `hasExpression`;
- `lintTemplateString`: unbalanced braces, empty span, unbalanced parentheses, odd quotes,
  unknown transform, unknown root identifier (after stripping string literals);
- roots `row`, `page`, `agg`, `ctx`, `sys`, `group`, plus dataset names;
- the 38 transforms with their signatures (`TRANSFORM_SIGNATURES`);
- completion and format-mask helpers (`readFormatMask`, `applyFormatMask`,
  `clearFormatMask`).

**ExpressionEditor** (`QDialog`, about 620 px wide):
- a `QPlainTextEdit` with syntax highlighting of spans, roots and transforms;
- a completion popup only inside an unclosed `{{`: transforms after `|`, fields after
  `root.`, roots otherwise; case-insensitive prefix, 40 items; arrow keys move, **Tab**
  accepts, Enter does not;
- `row` fields are the band dataset's fields plus `__index`; built-ins are `page` (number,
  total, isFirst, isLast), `ctx` (companyName, companyLogo, branchName, accYear, docId,
  userId), `sys` (now, renderedAt), `group` (key, count), `agg` (sum, count);
- quick-insert buttons: `{{ }}`, `row.`, `ctx.`, `page.number`, `row.__index`;
- a transform list that inserts `|name` at the caret;
- live lint lines in red;
- Apply (Ctrl+Enter), Cancel (Escape).

Two traps that silently print wrong numbers belong in the editor's help text and in
review:

- `|coalesce(0)` returns a **string**, so `a|coalesce(0) + b|coalesce(0)` concatenates.
  Use `|num`, and parenthesise before a transform:
  `{{ (a|num + b|num)|fmt('##,##,##0.00') }}`.
- NUMERIC columns arrive as strings, so `visible: "{{ header.cess_amt }}"` is true for
  `"0.00"`. Write `{{ header.cess_amt|num != 0 }}`.

A warning rule for both is a cheap, worthwhile addition to `Validate`.

---

## 8. Validation

Port `lib/validate.ts` exactly. Errors block Save; warnings do not. Each problem carries a
band index, an element id or a scope, so clicking it navigates.

| Area | Rules |
| --- | --- |
| Paper | E: margins leave no printable width. E: GRID with no columns |
| Datasets | E: duplicate name |
| Band | E: unknown dataset. E: DETAIL or a group band without a dataset. E: repeating over a "one" dataset. E: group band without group-by. W: zero height with elements |
| Element | E: duplicate id anywhere. GRID: E missing col or row, E past the columns. GRAPHIC: E past the page width, W crosses the right margin, W left of the left margin, W more than 1 mm below a band that does not grow (crosstab exempt). E: empty value (TEXT, FIELD, BARCODE, QRCODE) or empty image source. FIELD aggregate: E unknown dataset, E group scope outside a group band |
| Crosstab | E: outside REPORT_HEADER, SUMMARY, REPORT_FOOTER, NO_DATA. E: in GRID mode. E: no or unknown dataset. E: row header at least as wide as the element. E: an empty pivot expression. W: several measures with blank captions. W: reads a `row.X` the dataset does not return. W: a fixed column width that fits no column |
| Expressions | every expression field is linted; a lint issue is an error. W: reads `row` in a band with no dataset (crosstabs and aggregates exempt) |
| Whole | E: no bands. E: more than one REPORT_HEADER, PAGE_HEADER, PAGE_FOOTER, SUMMARY, REPORT_FOOTER or NO_DATA |

Crosstabs live only in the four non-repeating bands because a repeating band would print
the whole table once per row, and page bands clip by primitive count.

---

## 9. The host contract

`features/printing` opens the canvas and owns saving and preview. On the web this is a
React context (`host/canvas-host.tsx`). In C++ it is an interface the template designer
implements:

```cpp
struct RenderRequest {
    Definition definition;           // the live canvas, sent only when previewsUnsaved
    QString accYear, outputMode;     // blank = omit
    int copies = 1;
    QVariantMap params;              // prompt answers; company_id is stripped by the host
};

struct PreviewHost {
    bool ready = false;
    QString notReadyReason;          // e.g. "Save the revision first"
    bool previewsUnsaved = false;    // true only on an editable DRAFT
    QString defaultAccYear;
    QVector<PromptParam> prompts;    // ptvParams minus server-owned names
    std::function<QFuture<RenderResult>(const RenderRequest &)> render;
};

class CanvasHost {
public:
    virtual ~CanvasHost() = default;
    virtual QString label() const = 0;            // top-bar meta line
    virtual bool readOnly() const = 0;            // a published or retired revision
    virtual QString readOnlyReason() const = 0;
    virtual QFuture<void> save(const Definition &) = 0;   // failure keeps the canvas dirty
    virtual void closeCanvas() = 0;
    virtual const PreviewHost *preview() const = 0;       // nullptr hides Preview entirely
};
```

`DesignerWindow` is constructed with the host, the seed definition, the dataset
descriptors and a `seedKey` (`ptlId:ptvId`). It re-seeds only when the key changes, and
**not** after a save, so the operator's selection and undo history survive saving.

On a read-only revision the canvas stays interactive (useful for inspecting), but Save is
refused with the host's reason.

The host side (`CanvasBridge`, the save payload with `includeDatasets = false` and the
preserved published pointer) is in the printing plan, section 8.2.

### 9.1 Preview dialog

Port `components/PreviewDialog.tsx` over `PreviewHost`:

- badges: pages, revision, output mode, size in KB, warnings;
- an output-mode combo (Automatic, PDF, ESCPOS, ESCP_DOTMATRIX), an accounting year
  seeded from the host, and one input per prompt by type (date, number, text; mark
  required ones);
- Render: once on open, then on demand; blank answers are omitted;
- notices: "a published revision previews its stored bands, not your unsaved changes",
  the host's not-ready reason, the error count;
- server refusals listed as `field — message`;
- PDF in a `QPdfView`; raw output through `EscPosText` from the printing module;
- add **Print** and **Save as…**, reusing the printing module's delivery classes.

On an editable draft, Preview sends the live body, so an unsaved broken element fails the
render even though nothing was saved. Say so in the error notice.

---

## 10. Desktop additions worth making

These are small in Qt and close real gaps on the web. None needs a server change.

1. **Import and export bands as a file.** The web has no import, and its export calls a
   dead route, so loading a layout today means writing `ptv_body` in SQL. Export writes
   the definition to a `.json` file; Import replaces the bands (one undoable command,
   validated first).
2. **Copy between windows** through the system clipboard (section 5).
3. **Separate designer windows**, one per revision (section 2).
4. **Remember view settings** (zoom, grid size, snap, show expressions, dock layout) in
   `QSettings` with `saveState` and `restoreState`.
5. **Confirm before closing** with unsaved changes, in `closeEvent`. This replaces the
   web's `UnsavedGuard`.

---

## 11. Web quirks: fix, do not copy

| Web behaviour | Qt decision |
| --- | --- |
| The canvas paints in array order and ignores `z` | Paint by `z`, then array order, as the server prints |
| In GRID mode, paste, duplicate, align, distribute and the Position inputs update `x`/`y` or `col`/`row` but not both; the canvas and the validator can disagree | `col`, `row` and `cols` are authoritative in GRID. Every write goes through one `syncGridGeometry` that also sets `x`, `y`, `w` and `h` (`h` = 1 when missing) |
| GRID resize from the west handle changes the span but not the column | Move `col` and adjust `cols` together |
| A new TEXT or FIELD in GRID keeps `h = 5`, i.e. five rows | Default `h` to 1 row in GRID |
| Style, font and Position edits on N elements create N undo entries | One command per user action |
| Resize has no live preview; the geometry jumps on release | Show the live rect while resizing |
| B, I and U toggles never show a pressed state | Show it (checked when all selected elements agree) |
| Remove this band has no confirmation | Confirm |
| Global keys keep firing while a dialog is open | Modal dialogs block them in Qt by default; scope canvas keys (arrows, Delete) to the view with `Qt::WidgetWithChildrenShortcut` |
| Expression lint issues are errors and block Save, while code comments say lint must never block | Match the web until the owner decides (section 13) |

---

## 12. Keyboard

| Keys | Action | Qt shortcut context |
| --- | --- | --- |
| Ctrl+S | Save (works while typing in a field) | window |
| Ctrl+P | Preview (only with a host preview) | window |
| Ctrl+Z, Ctrl+Shift+Z, Ctrl+Y | Undo, Redo | window |
| Ctrl+C, Ctrl+X, Ctrl+V | Copy, Cut, Paste (paste needs a selected band) | canvas |
| Ctrl+D | Duplicate | canvas |
| Delete, Backspace | Delete the selection | canvas |
| Arrows, Shift+Arrows | Nudge 0.5 mm or 5 mm; one cell in GRID | canvas |
| Ctrl+A | Select all in the band | canvas |
| Escape | Cancel the current drag, resize, marquee or placing; else clear the selection | canvas |
| Ctrl+G, Ctrl+;, Ctrl+E | Grid, Snap, Show expressions | window |
| Ctrl+Wheel | Zoom | canvas |
| ? | Keyboard shortcut sheet | window |

Canvas-context shortcuts must not fire while a property editor has focus. That falls out
of the widget shortcut context; do not reimplement the web's "ignore when an input is
focused" check.

---

## 13. Testing

- **Golden bodies.** The three fixtures in `lib/__fixtures__/` (GST A4 invoice: 6 bands,
  101 elements; quotation rate matrix A4 with a crosstab: 6 bands, 85 elements; T80
  thermal receipt in GRID: 3 bands, 38 elements) must load, validate with the same
  problems as the web, and save back **semantically equal** to the input.
- **Parity vectors**, generated from the web's vitest files as in the printing plan,
  section 9.1: `units`, `grid`, `geometry`, `snap`, `defaults`, `validate`, `expression`,
  and the reducer tests in `store/designer.test.ts` (as sequences of document edits with
  the expected definition after each).
- **Command tests:** every edit is undoable and redoable to the identical definition;
  coalescing merges inside 400 ms and not after; the stack stops at 50.
- **Interaction tests** with `QTest` mouse events on a `CanvasView`: drag with snap,
  marquee, resize with the minimum size, placing mode, field drop.
- **Round trip against the server:** save through the host, reload, compare; then preview
  and check `X-Print-Pages`.

---

## 14. Phases

Rough sizes for one developer experienced in Qt. The canvas starts after phase 0 of the
printing plan and can run in parallel with its phases 1 to 4.

| Phase | Scope | Done when | Rough size |
| --- | --- | --- | --- |
| C0. Model | JSON-backed model, `normaliseBand`, vocabulary, units, grid scale, geometry, snap, defaults, validate, expression; parity vectors | All three fixtures round-trip; every model vector passes | 2 weeks |
| C1. Read-only canvas | `DesignerWindow` frame, scene, band bodies, element painters, rulers, caption gutter, zoom, GRAPHIC and GRID | The three fixtures draw correctly beside the web canvas at 100% and 200% | 2 weeks |
| C2. Editing | selection, marquee, drag, resize, snapping, guides, nudge, band height, commands and undo, clipboard, align, distribute, z-order, placing mode | All interaction and command tests pass | 3 weeks |
| C3. Panels | report tree, data tree with field drag, property grid and every section including crosstab | Every field on the web's property panel is editable | 3 weeks |
| C4. Host and preview | `CanvasHost` wired to the template designer, save, read-only mode, expression editor, problems panel, preview dialog | Edit, save and preview a draft end to end; a published revision refuses Save | 2 weeks |
| C5. Polish | shortcut sheet, settings persistence, import and export, close guard | — | 1 week |

---

## 15. Open questions

1. **Paper on the canvas.** Make the Paper section read-only when hosted (recommended),
   or keep the web's editable-but-overwritten behaviour?
2. **Lint blocks Save.** Keep the web's behaviour, where any expression lint issue is an
   error, or downgrade lint to warnings as the code comments intend?
3. **"Back to saved" is clean?** The web marks the canvas dirty after undoing back to the
   saved state. `QUndoStack`'s clean state could clear it. Either is fine; pick one.
4. **Legacy QtRPT designs.** If the old desktop app's QtRPT XML reports must come across,
   an importer to `JSON_BANDS` belongs here as a later phase. None exists today.
