---
name: theme-template-decision
description: User rule 2026-10-01 — no app.qss, no stylesheet in any .ui/code; ONE QSS template in the DB + theme tokens; sweep screen by screen. ThemeManager + seed v1 built; waiting on Prathap's template routes
metadata:
  type: project
---

**User rule (2026-10-01):** no `app.qss`, no stylesheet in any `.ui`, and none in code; styling is applied via the theme master only.

**Decisions (asked):**
- **Rules:** ONE shared QSS template in the DB (`app_theme_template`, `{{key}}` placeholders), filled with the company theme's tokens. Not a stylesheet per theme, and not one compiled into the app.
- **Sweep:** screen by screen. Each step moves a screen's inline styles into the template as selectors, then deletes them.

**Why:** a rule is written once, and a theme stays a palette. Fixed colours that are not the brand (action buttons, scroll bars) stay as literals INSIDE the template, which is still theme-master data.

**Written:** `share/theme/plan-app-theme-template.md`:
- table 44, one active row;
- routes `/app-themes/template`, `/template/save` (409 on a stale `tplModifiedOn`);
- `/effective` gains `template`;
- an unauthenticated `/bootstrap` for the login window;
- v2 keys `primary.soft`, `.softer`, `.soft.border`, `.pressed.bg`, `.pressed.border`;
- MAROON `table.*` corrected to today's look: header `#FAFAFA` / `#8B1A1A`, selected `#F2DADA` / `#111827`, row.alt `#FFF8F7`. The 09-30 seed took `theme.h` values; grids are painted by the QSS.

**Built 2026-10-01 (client):**
- `src/core/theme/theme_tokens.h` (defaults = MAROON as it looks today + v2) and `theme_manager.h/.cpp`: fill / resolve / applyEffective / applyCached / applyStartup / fetchForCompany / color; QSettings cache `theme/<companyId>` + `theme/last`.
- `ApiEndpoints::AppThemes`.
- Wired into `main.cpp` (`applyStartup` after `loadAppStyle`) and `MenuWindow::onCompanyChanged`.
- **It does nothing until `/effective` carries a template**, so `app.qss` is still loaded for now.

**Seed template:** `share/theme/seed-template-v1.qss` = `app.qss` (59 placeholders, line-mapped) + the MenuWindow sheet scoped under `MenuWindow` (24 placeholders). It reproduces today's sheet exactly, except the combo hover `#F3E8E8` → `primary.soft` `#F8E8E8`.

Test: `tests/theme/run.sh`, 16/16, no network.

**Prathap shipped it 2026-10-01; live-verified:**
- `app_theme_template` (row 1 NEXERP = the seed, byte-identical, md5 b13900f3…);
- `/template` (with `placeholders[]`) and `/template/save`: unknown placeholder 400, unclosed { 400, non-`:/` url() 400, >512 KB 400, stale `tplModifiedOn` 409;
- `/effective` carries `template`; the ETag includes the tpl, and a conditional GET → 304;
- `/bootstrap` with NO auth → tokens + template;
- `audit.audit_log` rows with `log_table_name = 'app theme template'`;
- v2 keys accepted: all 3 themes now have 38 tokens, and MAROON's `table.*` is corrected; server MAROON == client defaults on all 38.

**MY MISTAKE (fixed):** a "valid url()" probe SAVED a junk rule into the live template for ~11 s; it was restored to the exact seed (md5 checked). Never send a VALID template body with the current stamp as a "test".

**Sweep step 1 DONE 2026-10-01:**
- `app.qss` deleted (`git rm`), along with its qrc entry, `loadAppStyle()` and the 4 `menu_window.ui` styleSheet props;
- `main.cpp` calls `ThemeManager::applyStartup()` after TLS (cache, else `/bootstrap`).

**Proof:** `tests/mainwindow/run.sh` with `MW_OLD` (today's sheets from git, applied from the START) vs the theme master gives PIXEL-IDENTICAL main windows at 1920/1600/1366 wide.
- Trap: swapping sheets after show re-sizes AdjustToContentsOnFirstShow combos. A fair "before" must style from construction.

Theme unit test fixture: `tests/theme/expected-v1.qss`. The issued/register snapshot tests now use `applyTheme()` (ThemeManager) instead of `app.qss` (compiled, not run).

**Sweep step 2 DONE 2026-10-01 (shared widgets):**
- 0 setStyleSheet left in `shared/` + `core/` (only ThemeManager applies the sheet); `loginwindow.ui` and `txn_shipping_dialog.ui` are free of styleSheet.
- Moved: AppMessage (`nexTone` critical/warning/info/confirm, objectNames msg*), NexWarningStrip (`nexRole` badgePill/badgeText/badgeHead + `nexBadge` bucket; noteCode/noteHint), pagers (`nexRole="pager"` + `nexTone="active"`), captions (`nexRole="caption"`), hintBar, toolbarRule, `#nexTokenFrame` / `#nexChip`, `#nexDropButton`, `#nexHeaderMenu`, NexPopupWindow + `#popupSearch`, `#sectionHeading` / `#sectionRule`, `#fetchButton`, `#comboClear`, `#imgThumb`, `#bandTitle` / `#bandHint` (+ tone), `#adjust*`, `QDialog#TxnShippingDialog` + `#lblState[nexTone]`, `QWidget#LoginWindow …`.
- New helper `src/shared/ui/nex_style.h`: `NexStyle::set` / `role` / `tone` = setProperty + unpolish/polish + a **StyleChange event + updateGeometry** (without them a padding change does not re-layout — the shipping strip came out 10 px short).
- Template **v2** (`share/theme/seed-template-v2.qss` = v1 + Part C) is LIVE.
- **Proof:** `tests/sharedstyle/run.sh <dir>` pictures 23 widgets. Before vs after are pixel-identical except the login clock (which also differs run-to-run).
- **Selector rule:** a moved rule must out-rank Part A's pseudo-state rules (`QPushButton:focus` etc.), which a widget-level sheet used to beat for free. So prefix with an id, `[nexRole]` or `[nexTone]`, and put Part C after Part A (equal specificity → later wins).
- Brand-ish literals left as-is for exact MAROON pixels: pager `#8b1a1a`, fetch hover `#991B1E` / `#5F0D0F`, combo clear pressed `#5F0D0F`. They need a `primary.strong` / `primary.deep` key later.
- Not pictured: TxnMainView captions/hint bar and the NexTable header menu (checked by reading the rules).

**Step 2 regression, found + fixed in step 3:** Qt ranks sheets by DISTANCE: own > ancestor screen > app. Once the shared widgets lost their own sheets, a screen's widget-level `QDialog {bg}` / `QPushButton {...}` beat their template rules. Message boxes opened from Receipt etc. took the screen's grey look, and the shipping dialog took the sale bill's background. The standalone harness could not see it.

**Sweep step 3 DONE 2026-10-01:**
- **Accounts:** 0 .ui sheets, 0 setStyleSheet.
- **Part D (screens)** holds ALL 20 screens' dialog-level .ui sheets (accounts also per-label), scoped by `$S/scope_ui.py` (kept in that session's scratchpad):
  - `QDialog#Name X`;
  - the self rule is emitted as `QDialog#Name, QWidget QDialog#Name, QDialog#Name QDialog` so a nested screen (tender in sale bill, cheque books) wins at equal specificity by ORDER.
- **Template order is now A, B, D, C (= v3, live).**
- **Hardened:** AppMessage = `AppMessage#appMessage[nexTone]`; shipping = `QDialog#TxnShippingDialog[nexRole="shipping"]`.
- **TRAP:** set a dialog's id/role BEFORE setupUi / before the children exist. A child styled earlier is NOT re-styled when the ancestor's property changes (the shipping group boxes lost 4 px padding).
- **Accounts code:** status pills, good/bad bands and differences, and party fact colours are now `nexTone`. OB's band rule also reaches its labels (QLabel is a QFrame), so `nexBand` is set on children. Pagers use `nexRole="pager"`. Code-built dialog labels got objectNames + rules.
- **Proof:** `tests/screenstyle/run.sh <dir> [Class…]` pictures 20 screens + a message box and the shipping dialog PARENTED to each. Before vs after: 19/20 identical (PhysicalStock live row = noise); all 20 message boxes == the no-parent reference; shipping in the sale bill == standalone.

**Remaining whole app (before step 4):** 345 .ui styleSheet props in 31 files and 147 setStyleSheet in 41 files. **Sweep step 4 DONE 2026-10-01 (sales):**
- 0 .ui sheets and 0 setStyleSheet in built sales files. `modules/sales/quotation_entry.ui` is an orphan, not in CMake, and was left alone.
- Per-widget sheets are converted by `$S/scope_children.py`:
  - a plain sheet becomes `W` for a QLabel and `W, W *` otherwise (a selector-less sheet reaches everything inside the widget);
  - a selector sheet becomes `W S`, plus the widget itself when S can match it.
  - These rules are inserted right AFTER the screen's Part D block.
- Code states become tones: credit alerts, fulfilment, tender chips, the balance sign, cash limit, party facts (success / warning), loyalty member. The retender and open-sources labels got objectNames.
- **Template v4 is live.**
- **TRAP #2:** some widgets (a QGroupBox's contentsMargins) are computed at FIRST polish. A widget-level setStyleSheet() re-styled the whole tree after setupUi; template rules do not, and the sale bill totals box lost 1 px of border margin.
  - **FIX:** `NexDialog::event(Show)` runs `restyleTree()` once (unpolish / polish + StyleChange on every child). This covers every screen.
- **Proof:** every sales screen is pixel-identical when run alone. Full-run order effects (OpeningStock, Quotation) are timing — identical alone.
- Change Selling ui_table 44 was EDITED on the server at 17:04 on 10-01 (column 9 "Net Rate" etc.) by someone else — a data change, not style.

**Sweep step 5 DONE 2026-10-01 (inventory):**
- 0 .ui sheets and 0 setStyleSheet in inventory. **Template v5 is live.**
- Status pills (opening / physical / adjustment), relot balance, Change Selling validation strip / row card / legend chips are now tones or fixed rules.
- Code-built dialogs: the pick dialog got objectNames; the Change Selling dialogs got roles `dlgTitle` / `dlgMuted` / `barPrimary` / `barButton`.
- **TRAP #3:** one-line `<property name="styleSheet">` (inside the widget's own line, as in the stock adjustment / change selling .ui) is missed by a removal regex anchored on a newline. Its rules were still generated; only the removal missed them. Re-check with `grep -c 'name="styleSheet"'`.
- **Proof:** each inventory screen alone is identical to before (PhysicalStock live row = noise; Change Selling == its step-4 picture — its header text differs only because ui_table 44 was edited on the server).
- **CONCURRENT WORK:** another session (`nexerp-4d`) edited the change_selling_price_* files at 17:30-17:31 on 10-01 (SellingPriceFilterDialog F8, "notes 76") and ui_table 44. It KEPT the step-5 changes. Coordinate before touching those files again.

**Sweep step 6 DONE 2026-10-01 (admin + printing) — THE SWEEP IS COMPLETE:**
- **End check:** 0 styleSheet props in built .ui, 0 setStyleSheet calls, no .qss file. The ONLY setter is `ThemeManager::applyValues` (`qApp->setStyleSheet`).
- **Orphans left alone (not in CMake):** `modules/sales/quotation_entry.ui` (21 sheets) and `modules/admin/devices/Temp/device_entry.ui` (8). The user's call whether to delete them.
- **Moved:**
  - canvas `kStyle`, properties panel, problems panel and shortcuts dialog sheets → window-scoped rules generated from the git HEAD text (`cpp_sheet.py`);
  - designer preview dialog;
  - app settings (rows `nexTone` odd/even, badges `nexTone` locked/here/from/default; maroon → `{{primary}}`);
  - template list roles `templateAction` / `templateActionPlain`;
  - names on every print-dialog label.
- **Colour swatch** (canvas properties) = the operator's chosen colour → painted as an icon, not styled. Data, not theme.
- `lblCanvasNote` "cleared" state = `color: palette(window-text)` (a rule cannot unset).
- **Template v6 is LIVE** (135 KB): `share/theme/seed-template-v6.qss`.
- **Proof:** `tests/adminstyle/run.sh` (17 windows) is identical to before; NexComboEntry's height varies run-to-run, and it matches the before of the same size. Final regression: 20 screens + shared + message boxes + shipping all hold; the theme unit test is 16/16.

**FOLLOW-UPS (not done):**
- the theme master screen's Template tab (plan §5.3);
- ~500 painted `QColor` literals in delegates → `ThemeManager::color()` (painted, not styled, so outside the user's rule, but they won't follow a theme);
- brand-ish dark-maroon literals (`#8b1a1a` pager, `#8b1d1d` canvas, …) need a `primary.deep` key to follow BLUE / YELLOW.

**2026-10-01 later:** the Template tab follow-up is DONE — menu 266 screen built ([[theme-master-screen]]), template v7 live. Remaining follow-ups: painted QColor literals (~113 in 21 files), primary.deep.

**2026-10-01 sweep 2 (user: "do not keep any stylesheet in code/ui in any form"):** plan = share/theme/plan-app-theme-sweep-2.md. Step 1 DONE: template v8 live = ONE button system in Part A (primary/danger chosen by objectName btnSave/btnPost/btnDelete…, nexRole="primary" for unnamed code buttons, any :checked = primary fill); per-screen button rules + bar/kind/verb/primary .ui props removed. Next = step 2 (.ui <font> blocks, 256 in 10 files), then label HTML colours, setPalette, setFont, painted QColor → tokens (needs new token keys = notes for Prathap), then a guard script. Harnesses take SS_QSS=<seed> to preview an unpublished template (screenstyle + adminstyle). Always md5-check live == previous seed before publishing.

**2026-10-01 NEW DESIGN (user: "no need to keep our existing ui design… do it completely new, no legacy"):** mockup share/theme/new_design_mockup.png approved; user chose (a) new look app-wide via template FIRST, then (b) per-screen layout rework one screen at a time (sale bill first). (a) DONE: template v9 LIVE = fresh system by widget type (parts 1-10) + generated "meaning map" (part 11: old ids → title/caption/muted/panel/pill/strip by tone, NO per-screen values) + kept-verbatim part 12 (canvas, login, masterActionButton, badges). Built by scratchpad build_v9.py from v9_system.qss + v9_shell_shared.qss + v8. All 256 .ui <font> blocks deleted (step 2 done). Known for (b): receipt frameHeader has max-height 62 in .ui (cramped); grid header text uses token table.header.fg (MAROON = maroon, mockup showed grey). Next = (b) sale bill layout.

**2026-10-01 later — ROLLED BACK to v8:** after the sale bill (b) rebuild was revoked the user asked to seed v8; v8 is LIVE again (md5 verified). v9/v10 seeds stay in the share as files only. The 256 .ui <font> blocks remain deleted (user's no-style-in-ui rule), — MEASURED 2026-10-01: removing them changed NOTHING (995 widgets, 0 font differences under v8 with vs without the .ui fonts): the template's app-wide QDialog font rule (from the old app.qss) overrides .ui fonts, so those labels were never bold. Rules that WOULD make them bold are generated (scratchpad fontfix/v8b.qss, 15 rules / 256 labels) but NOT published — only on the user's ask, as a new look. Do NOT re-publish v9 without the user asking.

**Lesson:** a stylesheet font rule on an ancestor (QDialog { font-family; font-size }) overrides a .ui <font> on the child entirely (weight included) — measure with SS_FONTS before claiming a look changed.

**2026-10-02: v11 LIVE** = v8 with only the OpeningStockEntry block replaced (Change Selling look: white, cards, pill, frameButtons). Published after checking live md5 == v8; read back = 06eb30b6…. Save body is `{tplId, tplQss, tplRemarks, tplModifiedOn}` (GET returns tplQss, not tplBody). The permission classifier now blocks further shared-box writes/reads from harnesses in this session; ask the user before the next publish.

**2026-10-02 later: v12 LIVE** (md5 140d8d57…) = v11 + 3.0 grid rules in Part A (borders #A9A9A9, bold header, 40px gutter, :checked-only border on the current row, QTableCornerButton styled; sale bill header overrides dropped). MAROON token table.header.bg #FAFAFA → #EEF1F5 (user: keep today colours, change header bg); theme_tokens.h default + tests/theme/expected-v1.qss updated to match. See [[grid-look-theme-only]].

**2026-10-02:** MAROON table.row.alt #FFF8F7 → #FFFBFA and table.selected #F2DADA → #E8C4C4 (the user said alt and selection looked the same). The defaults + expected-v1.qss match.

**2026-10-05: v18 written (NOT published) = v17 + menu option A "light bar".** User compared 3 menu looks (share/theme/menu_style_diff.png: A light / B refined maroon / C slate) and chose A. Part B (MenuWindow) replaced by ONE token-based set (surface bar, text, primary accents, white rounded drop-downs, :/resources/assets/icons/quick/chevron_right.svg — the old chevron_right_white.png never existed; old Part B had every rule twice). Code: MenuWindow::tintModuleIcons() tints module icons with ThemeManager color("primary") (was hard Qt::white), re-run on ThemeManager::changed; buttons carry "iconPath". menu.bg / menu.fg tokens are no longer read by the bar. Harness tests/mainwindow: MW_QSS=<seed> (preview with live tokens), MW_EXTRA, MW_ICON_TINT, MW_MENU=<button> (picture with drop-down open). Publishing is blocked for me by the auto-mode classifier — the user publishes via menu 266. MDI work area stays Qt grey (QSS on QMdiArea had no effect; needs code if wanted).
**2026-10-05 19:12: the USER published the FIRST v18 (A, md5 132c4de6, no icon-tint line) via menu 266.** Then asked to "try C". Code changed after: icon colour is now `Q_PROPERTY QColor menuIconTint` on MenuWindow, set by the template (`MenuWindow { qproperty-menuIconTint: … }`), default WHITE (so ≤v17 maroon bars stay right). Seeds updated: v18 (A) md5 024f8ff9 adds `qproperty-menuIconTint: {{primary}}`; v18c (C slate) md5 04dc1c57 = `#E5E7EB`, slate #1F2937 bar + {{primary}} bottom edge, red #E5484D open underline, light drop-downs. Live (first v18) + new code = white icons on white bar → the user must publish an updated seed with the new build.
**2026-10-05 later (user: selectors uneven/ugly; try icon-left list buttons):** MenuWindow::setupContextSelectors now fixes company/branch/year cards to kSelectorWidth=210 each (combos AdjustToMinimumContentsLengthWithIcon, min 8), container Fixed width, layout spacing/margins 0, the .ui's 40px leading spacer zeroed, container height 48. Seeds v18 (A, md5 below) and v18c (C): selectors = ONE bordered container, flat combos, dividers on #frameSelectorBranch/#frameSelectorYear, drop-down border none; masterActionButton = `qproperty-toolButtonStyle: ToolButtonTextBesideIcon; qproperty-iconSize: 18px` (template overrides the code's TextUnderIcon — no code change). Harness: MW_OPEN=<menuId> opens a screen. Preview share/theme/menu_v18_selectors_and_list_buttons.png. Still unpublished; live = first v18 (A, no icon tint).
**Also 2026-10-05:** menu labels escape & → && for display (buildMenu + top-level buttons) — "Offer & Schemes" / "Profit && Loss" rendered a mnemonic underline; userBox max width 160 → 220 in menu_window.ui ("SUPER ADMIN" was cut).

**v19 LIVE 2026-10-06** (`share/theme/seed-template-v19.qss`, v18 + one Part A block):
- QDateTimeEdit (physical stock freeze from/to) and QDoubleSpinBox were missing from the field base rule, so they drew the native look.
- No rule styled the up/down buttons of any QAbstractSpinBox; they drew as broken half-boxes with no arrows.
- No rule for the calendar popup (QDateTimeEdit popup + NexDateEdit's picker): it had a native blue header.
- Now: `QAbstractSpinBox` buttons + arrows + `:disabled`; `QDateTimeEdit::drop-down` like the combo; `QCalendarWidget` nav bar = primary.
- **New icon `up_arrow.svg` in icons.qrc.** A client built before 2026-10-06 shows no up arrow; the dist bundle needs a rebuild.
- Probe: a standalone qmake widget sheet in the scratchpad; screens checked with `SS_QSS=… tests/screenstyle/run.sh dir PhysicalStockEntry LoyaltySchemeEntry PromotionSchemeEntry`.
- The live template has no trailing newline; compare with `.rstrip()`.

**v20 LIVE 2026-10-09 12:45 (published by the user via menu 266):** = v19 + Part D blocks TillMastersEntry, TillOpenSessionDialog, TillEndShiftDialog, TillCashMovementDialog, TillSessionCockpit (md5 c47df50c). Checked after publish: live == share/theme/seed-template-v20.qss; masters snapshot pixel-identical to preview.
**v21 LIVE 2026-10-09 13:01 (user published via menu 266):** = v20 + TillBusinessDayEntry block (md5 770dd78a). Checked: live == share/theme/seed-template-v21.qss; Business Day render pixel-identical to preview.
**v22 LIVE 2026-10-09 (user published via menu 266):** = v21 + TillApprovalSetupEntry block (md5 48eaf8c9).
**v23 LIVE 2026-10-09 (user published via menu 266):** = v22 + ExpenseVoucherEntry block (md5 824ad772).
**v24 LIVE 2026-10-09 (user published via menu 266):** = v23 + SettlementReconciliationEntry / SettlementImportDialog / SettlementFormatDialog blocks (md5 64a1a2e0).
**v25 LIVE 2026-10-09 (user published via menu 266):** = v24 + TillSlipCheckDialog block (md5 97d88098, verified against app_theme_template tpl_id 1).
**v26 written 2026-10-12 (NOT published):** = v25 with the TillCashMovementDialog block replaced (rev 3 layout). Before publishing check live md5 == v25 (97d88098).

**2026-10-22 v29 LIVE** (md5 da226b65, published by me on the user's "publish v29"): v28 + the 4 `QTableWidget QLineEdit/QSpinBox(:focus)` rules → `background: {{table.row}}` — cell editors were transparent, old value showed through. Read back == seed.

**2026-10-17 v28 LIVE:** v27 + share/theme/block-till-cash-position.qss (QDialog#TillCashPositionEntry: facts tiles, refusals flag band, sectionCap). Published with GET /app-themes/template → check live == previous seed → POST /template/save with its tplModifiedOn → read back == v28. Role selectors are scoped per dialog objectName, so a new screen needs its own block.
**2026-10-22: v30 LIVE** = v29 + the ItemQtyPriceEntry redesign block (11 lines). Published by scratchpad qp/publish_v30.py (it refuses unless live == v29), read back md5 033b3278.

**Publish rule learned 2026-10-23:** before every publish, compare the LIVE template's md5 with the previous seed file — on 10-23 the live one had been re-saved (part of the v33 draft published by someone else, remark still "v32"). Diff live vs the new seed; publish only when the diff is purely my additions, using the live tplModifiedOn (409 guards a race).
