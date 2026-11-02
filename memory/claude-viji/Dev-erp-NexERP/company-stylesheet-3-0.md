---
name: company-stylesheet-3-0
description: How 3.0 themed the whole app from ONE style_sheets row per company (placeholder CSS + colour columns, qApp->setStyleSheet after company pick) and where NexERP stands (app_theme_master stub with NO css, comp_stylesheet_id saved but never read)
metadata:
  type: project
---

**3.0 mechanism (DB `3.0` on 192.168.0.106, client `~/Projects/NextLvl/3.0/pos`):**
- `public.style_sheets` (3 rows: Default / Maroon / ePosGo): `style_css` (~4 KB Qt QSS
  with `&&STYLE_*&&` placeholders) + 12 colour columns (style/menu/title bg+fg/table
  header/bg/alt/select/row/report bg/focus/tratbl select).
- `companys.stylesheet_id` picks the row (no FK). Company entry combo = dropdown 34.
- `Settings/fiscalyearselection.cpp` after the company+year pick: `ffiscal_year(iflag=4)`
  → replace the 12 colour placeholders from the row, 3 more (`TITLE_FONT_SIZE`,
  `ICON_SIZE`, `TABLE_HEADER_HEIGHT`) from the PC's SettingsManager, then
  `qApp->setStyleSheet(css)`; colours also copied to `global.*` for painting code
  (global.BGColor used in ~73 places).
- Forms are NOT colour-free: 320/446 .ui carry styleSheet fragments, mostly neutral
  (bold/transparent/border:none) but 472 hex literals exist.

**NexERP today (2026-09-30):** `src/main.cpp loadAppStyle()` loads a single compiled-in
`src/shared/styles/app.qss` (619 lines, no placeholders) BEFORE login; `theme.h`
NexTableTheme is never overridden; 188 inline setStyleSheet calls / 508 hex literals in
.cpp, 630 in 35 .ui. New schema has `app_theme_master` (thm_id/name/active only, rows
MAROON/YELLOW/BLUE) ← FK `companys.comp_stylesheet_id`; API company-master saves/returns
compStylesheetId; Qt company_entry.cpp uses dropdown 27 (fixed.dropdown_details "APP
THEMES"); login response does not carry it and nothing applies it.

**How to apply:** a per-company theme needs (1) css + colour columns on
app_theme_master (or a child table), (2) a /get or login field, (3) re-apply
qApp->setStyleSheet after company pick, (4) the inline colours pulled into placeholders.

**PLAN + SQL WRITTEN 2026-09-30:** `42_app_theme_tokens.sql` in the share root (dry-ran twice on erp_dry; adds thm_tokens/base/default + seq + ck/ux, seeds MAROON=today, hidden menu under 60, grid "MAIN LIST - APP THEMES", 3 system.ui_* settings; ids come from Prathap). `/home/vk/Shared/Prathap/theme/plan-app-theme.md` — tokens (29
semantic colour keys in `thm_tokens jsonb`) not CSS; `/app-themes/get|effective|save|delete|restore`;
Qt `ThemeManager` + `app.qss` as `{{key}}` template, applied in `MenuWindow::onCompanyChanged`;
sizes via 3 new `system.ui_*` settings; phases 1-2 = the user's target, 3 = theme screen,
4 = the colour sweep, 5 = React. User decided: different from 3.0, company-wise, apply at login.

**Theme plan APPLIED by Prathap; live-checked 2026-10-01:**
- **DB:** app_theme_master has 13 columns, ck_thm_base / ck_thm_tokens, ux_thm_default / ux_thm_name, and the seq. MAROON(1, default) / YELLOW(2) / BLUE(3) have 33 tokens each. Acme→1, CHANDRA→3, SCM SILK NULL.
- **Ids:** menu **266** "App Themes" (parent 60, hidden); grid **125** (the column is `thm_used_by`, a string). The 3 `system.ui_*` settings are present.
- **Routes:** `/effective` works (COMPANY / DEFAULT; 404 for a deleted or unknown company; ETag `"thmId:modifiedOn:resolvedFrom"`, conditional GET → 304). `/get` works (404 for an unknown id).
- **VKPOS granted menu 266 on 2026-10-01** (V/C/E/D/Export) at the user's ask. The full-echo save was diffed: 84 rows → 85, the only change being +266.
- **Writes ALL VERIFIED live 2026-10-01:**
  - unknown token key 400 `tokens.<key>`; not a colour / null 400; tokens not an object 400; empty name 400;
  - duplicate name 409, case-insensitive;
  - delete the default 409; delete a theme in use 409 ("In use by N companies"); delete again 409; restore a live row 409;
  - a default swap clears the old default, and a company with no theme follows it; the default cannot be deactivated (400).
  - The company master's `ensureThemeIsLive` (code-read): checked on create, and on update only when the theme changes.
  - ZT-THEME = thmId 13, soft-deleted. MAROON was restored as the default (its modified_by/on changed, so its ETag changed).
- **Next:** the Qt client phase 2 (ThemeManager), per plan §5.
