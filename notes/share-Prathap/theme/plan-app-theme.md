# Company theme (app_theme_master) — backend + Qt client plan

**Written 2026-09-30. For Prathap (§2–§4) and for the Qt client (§5–§8). New module: nothing
here reuses an old route, DTO or DB function.**

**Scope.** A company picks a theme in Company master (Preferences tab, already there) and the
client is painted in that theme from the moment the company is known after login, and again
on every company switch in the shell. One theme row serves the Qt client and the React client.

**Sources checked.** `3.0` DB `public.style_sheets` + `Settings/fiscalyearselection.cpp` in
`~/Projects/NextLvl/3.0/pos`; ERP DB `public.app_theme_master`, `companys.comp_stylesheet_id`,
`fixed.dropdown_details` 27; `nexerp-api-ro/src/modules/settings/companyMaster/*`;
NexERP `src/main.cpp`, `src/shared/styles/app.qss`, `src/shared/ui/theme.h`,
`src/app/shell/menu_window.cpp`, `src/modules/admin/master/company/company_entry.cpp`;
`nexerp-client/app/globals.css`.

> **The whole feature in one line:** the database stores *colour values* per theme, never a
> stylesheet; each client owns one stylesheet template and fills it from those values.

---

## 1. What is live today (checked 2026-09-30 against 192.168.0.106 and nexerp-api-ro)

| piece | state |
|---|---|
| `public.app_theme_master` | ✅ exists, but only `thm_id, thm_name, thm_is_active, thm_is_deleted, thm_sync_date`. Rows 1 MAROON, 2 YELLOW, 3 BLUE. **No colour, no CSS.** |
| `companys.comp_stylesheet_id` | ✅ `FK → app_theme_master(thm_id) ON UPDATE CASCADE ON DELETE SET NULL`. 3 of 6 test companies point at 3. |
| Company master API | ✅ `compStylesheetId` saved by `/company-masters/create`, returned by `/get` with `compStylesheetName` |
| Dropdown 27 "APP THEMES" (`fixed.dropdown_details`) | ✅ `thm_id::text, thm_name` where active and not deleted |
| Qt company entry (`company_entry.cpp:154`) | ✅ Preferences tab, `apiDropdown("compStylesheetId", 27)`. A stale `TODO` comment above it says the id is 0; it is not. |
| React company module (`features/masters/settings/companies/module.ts`) | ✅ same field, same dropdown |
| `/auth/login` response | carries `dev_company_id` only, no theme |
| Anything that **reads** the theme | ❌ none, in either client or the API |
| Qt styling | one compiled-in `app.qss` (619 lines, fixed colours) applied in `main.cpp` **before login**; `theme.h` `NexTableTheme` defaults never overridden; 188 inline `setStyleSheet` calls, 508 colour literals in .cpp, 630 in 35 .ui |
| React styling | `:root` CSS variables in `app/globals.css` (`--primary: #7b1515`, `--header-bg`, `--row-selected`, …), fixed |
| Settings catalogue | `app_setting_def` has no UI-size key; scopes COMPANY / BRANCH / DEVICE / USER exist and `/app-setting-values/effective` is already re-read on branch change (`MenuWindow::fetchApplicationSettings`) |

So the *choosing* half is done end to end. This plan adds the *content* of a theme and the
*applying* of it.

---

## 2. Data — **written as `../42_app_theme_tokens.sql` (2026-09-30, dry-ran twice on the schema clone).** Where this section and the file differ, the file wins; the differences are the sequence (§2.2) and the CHECK helper.

### 2.1 Why tokens and not a stylesheet

3.0 stored a ~4 KB Qt stylesheet per row with `&&STYLE_*&&` placeholders. That text names Qt
widget classes and object names, so it belongs to a **client version**; the three 3.0 rows had
already drifted apart (Maroon uses `px`, Default uses `em`), and the React client could never
use it. Here the row stores **named colour values** only. Each client keeps its own template
and knows what to do with `table.selected`; the database does not need to know what a
`QHeaderView::section` is.

### 2.2 Columns

```sql
-- (abridged; the real statements, IF NOT EXISTS guarded, are in ../42_app_theme_tokens.sql)
ALTER TABLE public.app_theme_master
    ADD COLUMN thm_base        varchar(5)  NOT NULL DEFAULT 'LIGHT'
               CONSTRAINT ck_thm_base CHECK (thm_base IN ('LIGHT','DARK')),
    ADD COLUMN thm_tokens      jsonb       NOT NULL DEFAULT '{}'::jsonb,
    ADD COLUMN thm_is_default  boolean     NOT NULL DEFAULT false,
    ADD COLUMN thm_remarks     varchar(250),
    ADD COLUMN thm_created_on  timestamptz NOT NULL DEFAULT now(),
    ADD COLUMN thm_created_by  varchar(100),
    ADD COLUMN thm_modified_on timestamptz,
    ADD COLUMN thm_modified_by varchar(100);

-- exactly one default among the live rows: the fallback for a company with NULL comp_stylesheet_id
CREATE UNIQUE INDEX ux_thm_default ON public.app_theme_master (thm_is_default)
    WHERE thm_is_default = true AND thm_is_deleted = false;

-- every value is a colour: #rrggbb or #rrggbbaa. Keys are validated by the DTO (§3.2), not here,
-- so a client can add a key without a migration.
CREATE OR REPLACE FUNCTION public.fn_theme_tokens_valid(p jsonb) RETURNS boolean
LANGUAGE sql IMMUTABLE AS $$
    SELECT jsonb_typeof(p) = 'object'
       AND NOT EXISTS (SELECT 1 FROM jsonb_each_text(p) e
                       WHERE e.value !~ '^#[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$');
$$;
ALTER TABLE public.app_theme_master
    ADD CONSTRAINT ck_thm_tokens CHECK (public.fn_theme_tokens_valid(thm_tokens));
```

`thm_tokens` is a flat object `{ "<key>": "#rrggbb", ... }`. A key that is missing means
"use the client's compiled default for that key", which is what makes old rows survive new
client builds and new rows survive old builds. `thm_base` is reserved for a dark theme; phase 1
ships LIGHT only.

The table is **shared** (no company_id) on purpose, like the printing tables: a theme is a
palette, a company picks one. It already has `thm_sync_date`, so it rides the cloud push as it is.

### 2.3 The token keys (v1)

Semantic names, so MAROON / BLUE / YELLOW differ only in values. The Qt default column is
today's `app.qss` / `theme.h` colour, so step 1 of the client (§5) changes nothing visually.

| key | means | Qt default today | React variable today |
|---|---|---|---|
| `primary` | brand colour: buttons, tab underline, title band | `#7B1113` | `--primary` |
| `primary.hover` | pressed/hover of primary | `#611010` | `--primary-hover` |
| `on.primary` | text on primary | `#FFFFFF` | — |
| `surface` | dialog / card background | `#FFFFFF` | `--card-bg` |
| `surface.alt` | page / pane background | `#F3F4F6` | `--page-bg` |
| `text` | body text | `#111827` | `--text-main` |
| `text.muted` | hints, captions | `#6B7280` | `--text-muted` |
| `border` | frames, inputs | `#E5E7EB` | `--border` |
| `focus` | focused input border | `#2563EB` | — |
| `title.bg` / `title.fg` | entry-form title band | `#7B1113` / `#FFFFFF` | — |
| `menu.bg` / `menu.fg` | shell module buttons / menu bar | `#7B1113` / `#FFFFFF` | — |
| `table.header.bg` / `table.header.fg` | grid headers | `#DDDDDD` / `#000000` | `--header-bg` / `--table-header-text` |
| `table.row` / `table.row.alt` | grid rows | `#FFFFFF` / `#F5F5F5` | — |
| `table.row.hover` | hover row (React only today) | `#F8FBFF` | `--row-hover` |
| `table.selected` / `table.selected.fg` | selected row in a list | `#0078D7` / `#FFFFFF` | `--row-selected` |
| `table.txn.selected` | selected line in an **entry** grid (3.0's `tratbl`) | `#B9D8DF` | — |
| `table.checked` | ticked row (pick dialogs) | `#98FB98` | — |
| `table.free` | promotion free line | `#FFF4D6` | — |
| `danger` / `danger.bg` | errors, refusals | `#C01C28` / `#FECACA` | `--danger` / `--danger-light` |
| `warning` / `warning.bg` | warning strip | `#92400E` / `#FEF3C7` | — |
| `success` / `success.bg` | posted, cleared | `#15803D` / `#DCFCE7` | `--success-text` / `--success-bg` |
| `info` / `info.bg` | badges, hints | `#2563EB` / `#EFF6FF` | — |
| `rate.below` / `rate.above` | rate moved below / above price list (`theme.h`) | `#C62828` / `#2E7D32` | — |

33 keys (the table above counts the paired `x` / `x.bg` and `x` / `x.fg` rows as one line each). Sizes (font pt, icon px, header px) are **not** tokens — see §5.3.

### 2.4 Seed the three rows

```sql
UPDATE public.app_theme_master SET thm_is_default = true, thm_tokens = '{
 "primary":"#7B1113","primary.hover":"#611010","on.primary":"#FFFFFF",
 "surface":"#FFFFFF","surface.alt":"#F3F4F6","text":"#111827","text.muted":"#6B7280",
 "border":"#E5E7EB","focus":"#2563EB",
 "title.bg":"#7B1113","title.fg":"#FFFFFF","menu.bg":"#7B1113","menu.fg":"#FFFFFF",
 "table.header.bg":"#DDDDDD","table.header.fg":"#000000",
 "table.row":"#FFFFFF","table.row.alt":"#F5F5F5","table.row.hover":"#F8FBFF",
 "table.selected":"#0078D7","table.selected.fg":"#FFFFFF","table.txn.selected":"#B9D8DF",
 "table.checked":"#98FB98","table.free":"#FFF4D6",
 "danger":"#C01C28","danger.bg":"#FECACA","warning":"#92400E","warning.bg":"#FEF3C7",
 "success":"#15803D","success.bg":"#DCFCE7","info":"#2563EB","info.bg":"#EFF6FF",
 "rate.below":"#C62828","rate.above":"#2E7D32"}'::jsonb
WHERE thm_id = 1;   -- MAROON = what the client looks like today

-- BLUE and YELLOW: the same object with these keys changed
--   BLUE   : primary #1D4ED8, primary.hover #1E40AF, title.bg #1D4ED8, menu.bg #1D4ED8,
--            table.selected #1D4ED8, table.txn.selected #BFDBFE, table.header.bg #DBEAFE
--   YELLOW : primary #B45309, primary.hover #92400E, title.bg #B45309, menu.bg #B45309,
--            table.selected #B45309, table.txn.selected #FDE68A, table.header.bg #FEF3C7
```

Everything else stays as MAROON's values. (Write the two full objects in the .sql; the note
above is so the intent is reviewable.)

### 2.5 Also in the migration

- **Menu row** "App Themes" under Admin (hidden until the phase-2 screen exists). Tell me the id.
- **Grid** "MAIN LIST - APP THEMES": `thm_id, thm_name, thm_base, thm_is_default, thm_is_active,
  (select count(*) from companys c where c.comp_stylesheet_id = t.thm_id and not c.comp_is_deleted) as used_by`.
  Register and tell me the id. Search on `thm_name`.
- **Three settings** in `app_setting_def` (see §5.3 for why they are settings, not tokens):

  | key | module | type | default | max scope |
  |---|---|---|---|---|
  | `system.ui_font_pt` | SYSTEM | INT | 10 | USER |
  | `system.ui_icon_px` | SYSTEM | INT | 20 | DEVICE |
  | `system.ui_table_header_px` | SYSTEM | INT | 26 | DEVICE |

  `asd_needs_relogin = false`; the client re-applies them live.
- Dropdown 27 and the company master stay as they are.

---

## 3. API — new module `src/modules/settings/appTheme/`, `@Controller('app-themes')`

House rules apply: no `/list` (the grid lists), `thmId` on get/delete, create + update on one
`/save`, `forbidNonWhitelisted` on every DTO.

### 3.1 Routes

| route | who | does |
|---|---|---|
| `GET /app-themes/get?thmId=` | any logged-in user | one row: `thmId, thmName, thmBase, thmIsDefault, thmIsActive, thmRemarks, tokens, usedByCount, thmModifiedOn` |
| `GET /app-themes/effective?companyId=` | any logged-in user | **the route both clients call after login and on company switch.** Resolves `companys.comp_stylesheet_id`; if NULL, or it points at an inactive/deleted row, returns the `thm_is_default` row. Response = `/get`'s shape plus `resolvedFrom: 'COMPANY' \| 'DEFAULT'`. 404 only when there is no default row at all (a seed defect, not a runtime case). |
| `POST /app-themes/save` | menu right on the new menu (create / edit) | body §3.2. `thmId` present → update, absent → create (`thmId` comes from `app_theme_master_thm_id_seq`, added by the 42 file, so the insert simply omits it). Setting `thmIsDefault = true` clears it on the previous default in the same transaction (the partial unique index would otherwise refuse). |
| `POST /app-themes/delete?thmId=` | menu right (delete) | soft delete. **409** when `usedByCount > 0` ("in use by N companies") or when it is the default. The FK's `SET NULL` never fires because rows are never hard-deleted. |
| `POST /app-themes/restore?thmId=` | menu right (edit) | the `/items/restore` pattern |

Company master: no change, except **validate on save** that `compStylesheetId` is active and
not deleted (the FK accepts a soft-deleted row today) → 400 `compStylesheetId` "theme is not active".

### 3.2 Save DTO

```ts
class SaveAppThemeDto {
  thmId?: number;                 // absent = create
  thmName!: string;               // 1..100, unique among live rows (409 on clash)
  thmBase!: 'LIGHT' | 'DARK';
  thmIsDefault?: boolean;         // default false
  thmIsActive?: boolean;          // default true
  thmRemarks?: string | null;     // ≤ 250
  tokens!: Record<string, string>;
}
```

Validation of `tokens`, in the service, before the DB CHECK sees it:
- every key must be one of the v1 keys (§2.3) — 400 `tokens.<key>` "unknown token";
- every value `^#[0-9A-Fa-f]{6}([0-9A-Fa-f]{2})?$` — 400 `tokens.<key>` "not a colour";
- keys may be **omitted** (client default applies), never null.

Keep the key list in one exported constant `APP_THEME_TOKEN_KEYS` so the React client can import
it for its editor.

### 3.3 Caching

`/effective` is read by every client at every login. Send `Cache-Control: private, max-age=0`
and an `ETag` = `thmId:thmModifiedOn:compStylesheetId` so a client's conditional GET costs a
304. Not required for phase 1, but the response must carry `thmModifiedOn` from day one so the
Qt cache (§5.4) can compare.

---

## 4. Backend gaps this feature inherits

1. `app_theme_master` has no audit columns — added in §2.2, unlike every other master.
2. `thm_id` was a plain integer with no sequence — the 42 file adds `app_theme_master_thm_id_seq` and sets it as the column default, so `/save` omits the id on create.
3. The FK from `companys` accepts a soft-deleted theme; the guard is in `/company-masters/create`
   (§3.1) because a CHECK cannot look across tables.
4. `/auth/login` does not know the company's theme and should not: the shell's company combo is
   the real apply point (§5.2), the device's login company is only the first value of it.

---

## 5. Qt client — phase 1: the chrome follows the company

### 5.1 Module layout

```
src/core/theme/
  theme_tokens.h          the 29 keys + compiled defaults (the ONLY place a colour literal
                          for chrome may live from now on); toCss() moves here from theme.h
  theme_manager.h/.cpp    ThemeManager::instance()
                            apply(tokens, sizes)  fill template → qApp->setStyleSheet,
                                                  fill nexTableTheme, set QPalette, emit changed()
                            color(key) -> QColor  for painting code (delegates, chips, strips)
                            fetchForCompany(id)   GET /app-themes/effective, cache, apply
                            applyCached(id)       QSettings copy, before the network answers
src/shared/styles/
  app.qss                 becomes the TEMPLATE: every colour literal → {{key}}; the file keeps
                          its name and selectors, so nothing that references it changes
src/shared/ui/theme.h     NexTableTheme stays; ThemeManager writes it. Its defaults are
                          removed in favour of theme_tokens.h so there is one truth.
src/core/api/api_endpoints.h   namespace AppThemes { Get, Effective, Save, Delete, Restore }
```

Placeholder syntax is `{{primary}}`, `{{table.selected}}` — plain text, no meaning to QSS, so a
missed one is visible (the literal string shows in the sheet and the unit test catches it).

### 5.2 Apply points

| when | what |
|---|---|
| `main.cpp` | `ThemeManager::instance().apply(defaults)` replaces `loadAppStyle()`. If QSettings has a cached theme for the **last** company used on this PC, apply that instead — so the login window and the first paint already look like the company, offline or not. |
| `MenuWindow::onCompanyChanged` (`menu_window.cpp:297`) | after `AppSession::setCompanyId`: `applyCached(companyId)` then `fetchForCompany(companyId)`. This fires for the device's login company (the combo's first fill) and for every switch, so there is **one** apply path. |
| `MenuWindow::fetchApplicationSettings` (branch change) | already re-reads `/app-setting-values/effective`; `AppSession::applySettings` now hands the three size settings (§5.3) to `ThemeManager::applySizes`, which re-fills the template. |
| a second user logging in on the same shell | `onCompanyChanged` runs again via `loadCompanies()`; nothing extra. |

Failure to fetch is **silent** like the settings fetch: the cached theme, or the compiled
MAROON, stays in force. It is a menu the operator is looking at, not a settings screen.

### 5.3 Sizes are settings, not theme

3.0 substituted font size, icon size and table-header height from the PC's `SettingsManager`
into the company sheet. Here they stay out of the theme row and come through the settings
catalogue that already exists: `system.ui_font_pt` (USER — a person's eyes), `system.ui_icon_px`
and `system.ui_table_header_px` (DEVICE — a POS screen vs a desktop). The template references
them as `{{size.font}}`, `{{size.icon}}`, `{{size.header}}`; ThemeManager merges tokens and
sizes into one substitution map. Company decides colour, machine decides size, user decides
text size.

### 5.4 Cache

`QSettings` group `theme/<companyId>`: `{ thmId, thmModifiedOn, tokens }`. `fetchForCompany`
applies the cached copy at once, then the fetched one only if `thmModifiedOn` differs (no
second repaint when nothing changed).

### 5.5 Company entry

- delete the stale TODO at `company_entry.cpp:153`;
- beside the Stylesheet dropdown, a 6-swatch strip (primary, title.bg, table.header.bg,
  table.selected, table.txn.selected, danger) read from `/app-themes/get` on selection, so the
  operator sees what they are choosing. No "apply" here: the theme applies when the company is
  chosen in the shell, not when its master is saved.

### 5.6 Selectors the template gains (so phase 3 has somewhere to move to)

The sheet already styles by dynamic property (`[nexRole="masterActionButton"]`,
`[nexAction="delete"]`, `[class="lblTitle"]`). Phase 1 adds the generic ones every screen
needs, all coloured from tokens:

```
QLabel[nexTone="danger"|"warning"|"success"|"info"|"muted"]
QFrame[nexTone="danger"|"warning"|"success"|"info"]          -- strips, chips, banners
QLabel[nexRole="formTitle"]                                    -- the entry-form title band
QLineEdit[nexRole="amountBig"], QLineEdit[nexRole="amountTotal"]
QTableView[nexRole="txnGrid"]                                  -- entry grids: table.txn.selected
QPushButton[nexTone="primary"|"danger"]
```

A screen sets `setProperty("nexTone", "danger")` + `style()->unpolish/polish` instead of
`setStyleSheet("color:#C01C28")`. A helper `NexStyle::setTone(QWidget*, const char*)` does the
polish dance so nobody writes it 50 times.

---

## 6. Qt client — phase 2: Theme master screen (new menu, §2.5)

Own `.ui` on `NexDialog` (the accounts-screen rule; it is not a CRUD-generated form because of
the colour grid and the preview):

- **left:** the theme list (the §2.5 grid), verbs New / Edit / Delete / Restore / Set default;
- **right:** name, base, remarks, and a **token grid** — one row per key: key, meaning (from
  `theme_tokens.h`), colour swatch cell that opens `QColorDialog`, hex text cell, "reset to
  default" on Delete;
- **Preview** button: applies the edited tokens to the running app; Cancel or Close restores the
  company's theme. Save posts `/app-themes/save`.
- `Delete` on a theme in use shows the 409 text; the used-by count is a column.

Rights: create / edit / delete come from the new menu's `user_menus` grants like every screen.

---

## 7. Qt client — phase 3: the screens follow (the sweep)

Without this, a company theme recolours the frame and leaves the receipt screen in fixed colours,
which is exactly the gap 3.0 had (320 of its 446 .ui files carry local styling, 472 colour
literals). Rules for the sweep:

1. **No colour literal in a .cpp** outside `theme_tokens.h`. State colours → `nexTone` property
   (§5.6); painted colours (delegates, chip backgrounds via `DelegatePaint::background`, warning
   strips) → `ThemeManager::color(key)`, re-read on `changed()`.
2. **A .ui `styleSheet` may keep layout-only text** (`font-weight:bold; border:none;
   background:transparent; padding`) and nothing with a colour in it. Colours move to a property
   + template selector.
3. **Exempt:** the print canvas's *document* colours (`canvas_window.cpp`, `canvas_properties.cpp`)
   — a design's fill and line colours are the design's data, not the app's theme. Its editor
   chrome is not exempt.
4. Done means: `grep -rhoiE '#[0-9a-f]{6}' src --include=*.cpp --include=*.h --include=*.ui | wc -l`
   is the count in `theme_tokens.h` plus the canvas exemption, and nothing else.

File order, worst first (counts from 2026-09-30):

| file | .cpp literals | .ui literals |
|---|---|---|
| `accounts/cheques/received_cheques.*` | 3 calls | 74 |
| `accounts/cheques/issued_cheques.*` | 3 calls | 53 |
| `accounts/receipt/receipt_entry.*` | 15 | 49 |
| `accounts/payment/payment_entry.*` | 15 | 49 |
| `accounts/opening/opening_balance_entry.*` | 12 | 39 |
| `accounts/register/voucher_register_entry.*` | 3 calls | 32 |
| `sales/salebill/sale_bill_entry.*` + `sale_bill_tender.cpp` | 13 + 10 | 30 |
| `sales/master/promotion_scheme`, `loyalty_scheme` .ui | — | 31 + 31 |
| `admin/settings/app_settings_entry.cpp` | 22 | — |
| `shared/ui/app_message.cpp` | 24 | — |
| `shared/ui/txn/txn_transport_band.cpp`, `nex_warning_strip.cpp` | 8 + 6 calls | — |
| `admin/printing/print_designer*.cpp`, `print_template_list.cpp` | 10 + 12 + 9 | 42 |

Do the shared widgets first (`app_message`, `nex_warning_strip`, `nex_dropdown_multi`,
`nex_crud_master_page`, `txn_main_view`): each one fixes every screen that uses it.

---

## 8. Tests

- **Unit** `tests/theme/test_theme_manager`: every `{{key}}` in `app.qss` is a known key (no
  typo survives); a missing key falls back to the compiled default; an invalid value is ignored,
  not applied; `apply()` sets `nexTableTheme` and emits `changed()` once; sizes merge over tokens.
- **Offscreen** (the harness in memory): open Receipt under MAROON and BLUE, snapshot both, and
  assert the title band and header pixels differ while layout pixels do not.
- **Live** `tests/theme/run-live.sh`: `/effective` for Acme Foods (→ 3, `COMPANY`), for a
  company with NULL (→ default, `DEFAULT`); `/save` with an unknown key → 400 `tokens.x`; `/save`
  a second default → the first loses the flag; `/delete` a used theme → 409; company master save
  with an inactive theme → 400.
- **Shell:** switch company in the combo with no window open → repaint; the combo is locked with
  a window open, so a theme can never change under a document (existing rule, now load-bearing).

---

## 9. Phases

| # | phase | ends with |
|---|---|---|
| 1 | §2 migration + §3 routes (Prathap) | `/effective` answers for Acme; MAROON row = today's colours |
| 2 | §5 client: template, ThemeManager, apply on company change, cache, swatches | the chrome of every screen follows the company; **zero visual change** for MAROON |
| 3 | §6 Theme master screen | a theme can be made without SQL |
| 4 | §7 sweep, shared widgets first, then the table in worst-first order | the grep count is the token file only |
| 5 | React: one `applyTheme(tokens)` that writes the `--*` variables in §2.3 at login/company switch | same row, both clients |

Phases 1 and 2 are the target the user set; 3–5 make it complete.

---

## 10. Do not port from 3.0

- **CSS text in the database** (`style_sheets.style_css`). It couples the row to a client
  version and cannot serve the React client.
- **One sheet copy per theme.** The three 3.0 rows drifted (units, a dropped selector). Here
  there is one template per client and N rows of values.
- **`&&STYLE_*&&` string placeholders with PC settings mixed in** (`STYLE_ICON_SIZE`,
  `STYLE_TABLE_HEADER_HEIGHT`). Sizes come from the settings catalogue at DEVICE/USER scope.
- **`global.StyleColor` and friends** read in 60 files. Painting code asks `ThemeManager::color`
  and repaints on `changed()`; a global copy goes stale on a company switch.
- **The company + fiscal-year picker as the apply point.** The shell's company combo is live and
  already the single place the company changes.
- **A theme reference with no FK** (`companys.stylesheet_id` was a bare integer). The FK exists;
  the soft-delete guard is added on top.
- **Silent white.** When the 3.0 fetch failed the app came up unstyled. Here the cached or
  compiled theme stays in force and nothing flashes.

## 11. Open points

| # | point | proposed |
|---|---|---|
| O1 | should a BRANCH be allowed its own theme? | no — the user asked for company-wise; the settings layer is where per-branch overrides live, and colour is not a setting |
| O2 | DARK base | reserved by `thm_base`; no dark template in phase 1 |
| O3 | a per-user "high contrast" toggle | later, as a USER setting that picks an alternate template, not a theme row |
| O4 | React token editor | reuse `APP_THEME_TOKEN_KEYS` from the API package; not in this plan |
