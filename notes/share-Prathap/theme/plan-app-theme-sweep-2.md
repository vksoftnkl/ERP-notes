# Theme sweep 2 — nothing that sets a look left in code or .ui (2026-10-01)

**The user's rule:** "do not keep any stylesheet in code/ui in any form". Sweep 1 removed every
`styleSheet` property and `setStyleSheet` call (template v1–v6). This sweep removes the other ways
code still sets a look. The only things code may keep are:
- what a widget IS (its objectName, or `nexRole` / `nexTone`);
- painting, with every colour taken from `ThemeManager::color(token)`.

Audit on 2026-10-01 (src/, built files):

| Form | Count | Goes to |
|---|---|---|
| per-screen button rules (template) | 187 rules, ~41 looks | **step 1 — DONE, v8** |
| `<font>` blocks in `.ui` | 256 in 10 files | template rules by id |
| inline HTML colour in label text | 31 in 10 files | `nexTone` + template |
| `setPalette` / `QPalette::` | 57 in 14 files (mostly grid cell editors) | template (`QTableView QLineEdit…`) |
| `setFont` on widgets | 96 in 30 files (part is painting) | template where it is a widget |
| painted colour literals (`QColor(…)`, `Qt::red`…) | 113 + 20 in 28 files | `ThemeManager::color(token)` |

**Exempt (not app chrome):**
- the print canvas's paper and elements — they are document content;
- the theme master's swatches — they show the colour being edited.

## Step 1 — buttons (DONE, template v8 live 2026-10-01)

There is now one button system in Part A, built from theme colours:
- **standard:** every QPushButton.
- **primary:** chosen by name — `btnSave`, `btnPost`, `pBtnSave`, `pBSave`, `btnOk`, `btnPanelOk`,
  `btnPick`, `btnBookSave`, `btnBelowCostSave`, `btnFilterApply`, `settingsSave`, `btnDeposit`,
  `btnReplace`. A button made in code without one of these names uses `nexRole="primary"`.
- **danger:** `btnDelete*`, `btnCancelVoucher`, `btnCancelBill`, `btnCancelDoc`, `btnUnmap`,
  `btnBounce`, `btnReturned`.
- **success:** the cheque registers' Clear / Presented.
- **checked:** any checkable button that is on.

107 per-screen button selectors were removed. The `bar` / `kind` / `verb` / `primary` dynamic
properties came out of 4 `.ui` files, and `barPrimary` / `barButton` / `templateAction*` out of the code.

**Kept on purpose:**
- the canvas designer, which copies the web designer's look;
- the master-list toolbar (QToolButton, coloured per action);
- chips, the pager, AppMessage, the login window, `btnUsedBy` / `btnTemplateFilter` (filter pills)
  and `settingReset` (a link).

**Naming rule from now on:** a form's main action is named `btnSave` or `btnPost`, and a
destructive one `btnDelete…` or `btnCancel…`. A new name needs a line in the v8 block, not a style.

## Step 2 — `.ui` fonts
Move each `<font>` (pointsize / bold) into Part D, keyed by the widget's id under its screen. These
are the ten sales/inventory `.ui` files listed above, mostly the totals panels.

## Step 3 — label HTML colours + palettes
- Coloured spans become a `nexTone` on the label, or a second label.
- Cell-editor palettes become template rules.

## Step 4 — `setFont` on widgets
Widget fonts go to the template. Painter and model `FontRole` fonts stay: a stylesheet cannot reach them.

## Step 5 — painted colours → tokens
Each literal maps to the nearest existing token. A colour with no token (the violet "four numbers"
card, the verdict tints) needs a NEW key, which means extending `APP_THEME_TOKENS` on the server.
That will be a notes file for Prathap, written when this step starts.

## Step 6 — guard
`tests/nostyle/check.sh` fails the build check on any new:
- `setStyleSheet`, `styleSheet` / `<font>` / `<palette>` in a `.ui`;
- `setPalette`;
- colour literal outside the exempt files.
