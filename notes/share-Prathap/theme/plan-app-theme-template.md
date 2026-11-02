# App theme, part 2: the stylesheet template lives in the database

2026-10-01. This extends `plan-app-theme.md`. It does not replace it: the theme master (tokens, routes, menu 266, grid 125) stays exactly as built, and it is live-verified.

What changes is **where the stylesheet RULES live**.

## 0. The user's rule (2026-10-01)

> No `app.qss` file. No stylesheet in any `.ui`. Styling is applied through the theme master only.

The user decided the layout together with this rule:

- **Rules:** the QSS rules live **once**, in the database, as a template with `{{key}}` placeholders.
- **Colours:** each theme row (MAROON / BLUE / YELLOW) supplies the colours. That part already works.
- **Client:** it fetches template + tokens, fills one with the other, and applies the result. Nothing in the client carries a colour or a rule of its own.
- **Order:** the 353 `.ui` stylesheets and 197 `setStyleSheet()` calls move into the template **screen by screen** (§6). An unmoved screen keeps its inline style until its turn.

## 1. Why one template and not a stylesheet per theme

3.0 stored a full stylesheet per company. Every rule then existed N times, and a new screen's rule had to be added to every copy.

With one template, a rule is written once. A theme is only a palette, which is what `thm_tokens` already is.

The template may still contain **fixed literals** where a colour is not the brand's. For example, the master-list Add button is always green and Delete always red. That is still "via the theme master", because the template is a theme-master record.

## 2. Data: `public.app_theme_template` (new SQL file, 44)

```sql
CREATE TABLE public.app_theme_template (
    tpl_id          integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tpl_name        varchar(100) NOT NULL,
    tpl_qss         text         NOT NULL,          -- the rules, {{key}} placeholders
    tpl_is_active   boolean      NOT NULL DEFAULT true,
    tpl_is_deleted  boolean      NOT NULL DEFAULT false,
    tpl_remarks     varchar(250),
    tpl_sync_date   timestamptz,
    tpl_created_on  timestamptz  NOT NULL DEFAULT now(),
    tpl_created_by  varchar(100),
    tpl_modified_on timestamptz,
    tpl_modified_by varchar(100)
);
-- exactly one live template: the one every client applies
CREATE UNIQUE INDEX ux_tpl_active ON public.app_theme_template (tpl_is_active)
    WHERE tpl_is_active = true AND tpl_is_deleted = false;
```

**Notes on the table**

- It is **shared**, with no company column, like `app_theme_master`. One app, one set of rules.
- It has `tpl_sync_date` so it rides the cloud push.
- **One active row**, by design. A theme with its own rules would bring back the 3.0 problem. If a company ever needs a different layout, that is a new feature, not a second template.
- **Seed:** one row, `tpl_name = 'NEXERP'`. Its `tpl_qss` is the file `theme/seed-template-v1.qss`, which the client team writes (§5) from today's `app.qss` plus the shell's `.ui` styles, already converted to placeholders.

## 3. Token keys: v2 adds the brand tints and fixes MAROON's grid values

### 3.1 New keys

Hover and pressed states use light tints of the brand colour. None of the 33 keys covers them, so a BLUE company would get maroon-pink hovers.

| key | used for | MAROON (= today) | BLUE | YELLOW |
|---|---|---|---|---|
| `primary.soft` | hover background of buttons and list items | `#F8E8E8` | `#E8EEFC` | `#FEF3E2` |
| `primary.softer` | hover background of the master-list action buttons | `#FFF7F7` | `#F5F8FF` | `#FFFAF0` |
| `primary.soft.border` | border of the above on hover | `#F1CFCF` | `#C7D7F5` | `#F5D9A8` |
| `primary.pressed.bg` | pressed background of the master-list action buttons | `#F4E7E7` | `#DCE6FA` | `#FBEBCF` |
| `primary.pressed.border` | border when pressed | `#D9B3B3` | `#A9BFEA` | `#E8C07A` |

Add them to `APP_THEME_TOKEN_KEYS`. As before, a key a row leaves out falls back to the client default, so old rows keep working.

### 3.2 MAROON's grid values do not match today's screens

The seed took the `table.*` defaults from `theme.h`. The grids, though, are painted by `app.qss`. Applied as stored, MAROON would turn every grid header grey and every selected cell bright blue.

Correct MAROON's row to what users see today:

| key | MAROON as seeded | MAROON as seen today (correct value) |
|---|---|---|
| `table.header.bg` | `#DDDDDD` | `#FAFAFA` |
| `table.header.fg` | `#000000` | `#8B1A1A` |
| `table.selected` | `#0078D7` | `#F2DADA` |
| `table.selected.fg` | `#FFFFFF` | `#111827` |
| `table.row.alt` | `#F5F5F5` | `#FFF8F7` |

BLUE and YELLOW keep their own values; they were never "today's look".

**This is a data change only.** It goes through `POST /app-themes/save` on row 1, so no migration is needed. The client team can do it now that VKPOS has menu 266 rights. Say if you would rather do it yourself.

## 4. API

### 4.1 Routes (same module, `app-themes`)

| route | who | does |
|---|---|---|
| `GET /app-themes/template` | any logged-in user | the active row: `tplId, tplName, tplQss, tplModifiedOn, placeholders[]`. `placeholders` lists the distinct `{{key}}`s found. |
| `POST /app-themes/template/save` | menu 266, edit | body `{ tplId, tplQss, tplRemarks?, tplModifiedOn }`. `tplModifiedOn` is the value the editor loaded; if it differs from the row's, answer **409** "changed by someone else", so two editors never silently overwrite each other. |
| `GET /app-themes/effective?companyId=` | (exists) | **add** a `template: { tplId, tplQss, tplModifiedOn }` object to the response, so the client still makes one call at login and at a company switch. Extend the ETag to `thmId:thmModifiedOn:resolvedFrom:tplId:tplModifiedOn`. |
| `GET /app-themes/bootstrap` | **no auth** | the default theme's tokens + the active template, nothing else. The login window needs styling before anyone has logged in. These are colours and layout rules, not data. If an unauthenticated route is unacceptable, say so; the client then keeps the last cache and shows an unstyled login only on a machine's very first run. |

### 4.2 Validating `tplQss` (400, before anything is written)

- Every `{{...}}` must be a known token key (§3, `APP_THEME_TOKEN_KEYS`) or one of the size keys `size.font`, `size.icon`, `size.header`. Otherwise 400 `tplQss` naming each unknown placeholder, so a typo cannot ship.
- Braces must be balanced, counting `{` and `}` outside the placeholders. QSS that parses badly fails silently in Qt, and a missing brace drops every rule after it.
- At most 512 KB.
- `url(...)` may only point at `:/` resources. A template must not make every client fetch from the network.

### 4.3 Audit

A template save writes `audit.audit_log` with the before and after text. It is the one record that changes every screen at once.

## 5. Client (Qt), the client team's half

### 5.1 ThemeManager (`src/core/theme/`)

- **`fetchForCompany(companyId)`:** `/effective` → tokens + template → fill → `qApp->setStyleSheet()`. It also fills `nexTableTheme` (gutter, row colours) and the `QPalette` from the same tokens, so painted cells and styled widgets agree.
- **`color(key)`:** for painting code (delegates, chips). It replaces the 524 colour literals in C++ as their screens are swept.
- **Cache:** `QSettings theme/<companyId>` holds `{ tokens, tplQss, thmModifiedOn, tplModifiedOn }`, plus `theme/last`.
  - At start-up, before login, the last company's cache is applied. No file is shipped.
  - On a company switch, the cache is applied at once, then the fetched copy only if either timestamp differs.
- **Fill rules:**
  - a key the theme leaves out uses the client's compiled default for that key;
  - an unknown placeholder is left visible in the sheet and logged, so it shows up rather than vanishing;
  - a value that is not a colour is ignored.

### 5.2 The sweep (screen by screen; each step builds and looks the same)

| step | what moves into the template | then deleted |
|---|---|---|
| 1 | `app.qss` (all of it) + the shell (`menu_window.ui`, 6 brand uses) | `app.qss`, `loadAppStyle()`, the shell's inline sheets |
| 2 | shared widgets: `AppMessage`, `NexTable`, dropdowns, warning strip, date edit, popup window, dialogs' common parts | their `setStyleSheet` calls |
| 3 | accounts screens (receipt, payment, cheques, opening balance, ledger map, register) | their `.ui` / code styles |
| 4 | sales screens (bill, quotation, order, DC, tender, shipping) | the same |
| 5 | inventory / stock screens (item card, opening, physical, adjustment, change selling) | the same |
| 6 | admin / printing / everything left | the same; `grep styleSheet` finds nothing, which ends the sweep |

**Selectors in the template** come in three kinds:

- by role: `[nexRole="formTitle"]`, `[nexRole="card"]`;
- by tone: `[nexTone="danger"]`;
- for one-offs: `#objectName` inside the screen's class, e.g. `ChangeSellingPriceEntry QLabel#lblChangedChip`.

A screen sets properties; it never sets a stylesheet. The `NexStyle::setTone(widget, tone)` helper does the unpolish/polish.

The template is written as `theme/seed-template-vN.qss` in this folder. Each step's version is posted to `/app-themes/template/save`, so the box always holds the latest rules.

### 5.3 Theme master screen (menu 266)

It gets a second tab, **Template**: a plain-text editor with a placeholder list and a Preview that applies the template to the open window. Saving it uses §4.1. This is what makes "via the theme master only" true for the rules as well as the colours.

## 6. Order

1. **Prathap:** the 44 SQL file (§2), the v2 keys (§3.1), the routes and validation (§4). Seed the template with `seed-template-v1.qss` once it is in this folder.
2. **Client:** MAROON's corrections (§3.2) and the v2 values for all three rows, via `/app-themes/save`; then ThemeManager and sweep step 1. Step 1 can be built and unit-tested against the seed file before the routes exist.
3. Sweep steps 2–6, one at a time. Each step ends with a snapshot comparison and a new template version.

## 7. Tests

- **Server:**
  - an unknown placeholder → 400 naming it;
  - unbalanced braces → 400;
  - a stale `tplModifiedOn` → 409;
  - `/effective` carries `template` and the extended ETag (304 on a conditional GET);
  - `/bootstrap` answers without a token and carries nothing but tokens + template.
- **Client:**
  - every placeholder in the seed file is a known key;
  - MAROON + seed reproduces today's sheet: a snapshot of each swept screen before and after shows **no pixel change**;
  - BLUE changes the brand pixels and nothing else.
