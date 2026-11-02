# NexDropdown — React port plan

Scope: the **shared** server-configured dropdown (`src/shared/ui/dropdown/`,
1,575 lines of Qt) ported to React/TypeScript against the unchanged backend.

Companion documents: `plan-react.md` (Quotation Entry, beside this file);
`plan.md` (Qt) and the Qt sources this ports (`src/shared/ui/dropdown/`) live
in the Qt repo, `~/Projects/Qt6/NexERP/`.

**This is a prerequisite, not a peer.** Quotation alone needs six of these
(customer 39, POS state 21, area, agent, salesman, price level), and every
master screen needs more. Nothing in `plan-react.md` §6 (customer snapshot)
or §7 (place of supply) can be built until this component exists. Build it
first, alone, against a test screen.

---

## 1. What this component actually is

Not a `<select>`. A **remote-search combobox whose columns, formatting,
sorting, width and completion behaviour are defined server-side per dropdown
id.** Two GETs:

| Call | Endpoint | When |
|---|---|---|
| config | `/dropdown-details/get?dropdownId=13` | once per id, ever |
| rows | `/dropdown-details/run?dropdown_id=13&search=am&page=1&limit=10&dropdown_param={}` | per keystroke, debounced |

The config decides everything the popup looks like; the rows are opaque JSON
objects keyed by `dropdown_columns_sql_name`. Two conventions are load-bearing
and undocumented anywhere but the code:

- **Column 0 is the id.** `NexDropdownConfig::idKey()` returns the first
  column's key, full stop. That is the value stored on selection.
- **`dropdown_completion`** names the column whose value fills the input after
  a pick. It is also the fallback "something must be visible" column.

Three variants ship: **Single** (one value), **Multi** (chips), and the
`useDropdown` internals both share.

---

## 2. Module layout

```
src/shared/dropdown/
  NexDropdownSingle.tsx     // the common case
  NexDropdownMulti.tsx      // chips, ordered by pick order
  DropdownPopup.tsx         // the listbox: columns, highlight, empty/error/loading
  useDropdownSearch.ts      // config query + rows query + debounce + abort
  useDropdownKeys.ts        // open / highlight / choose / dismiss state machine
  masters.tsx               // dropdownId -> { menuId, Entry } registry (Alt+C / Alt+A)
  config.ts                 // types + normalizeConfig (the all-hidden fallback)
  format.ts                 // dataType -> display string (pure)
  api.ts                    // config() / run() URL builders
  types.ts
```

`config.ts` and `format.ts` import nothing — pure, and the only parts worth
unit-testing exhaustively (§8). Everything else is React.

---

## 3. Config — port `normalizeConfig` exactly

```ts
type DropdownColumn = {
  jsonKey: string;      // dropdown_columns_sql_name, else _name
  heading: string;      // _alias if present, else _name
  dataType: 'Text' | 'Number' | 'NumberTS' | 'NumericTS' | 'Date' | 'DateTime';
  align: 'left' | 'center' | 'right';
  widthPercent: number;
  visible: boolean;
  filterable: boolean;
};

type DropdownConfig = {
  dropdownId: number;
  name: string;
  completionKey: string;   // fills the input on select
  sortColumn: string; sortOrder: 'asc' | 'desc';
  maxVisibleItems: number; // default 10 — ALSO the request page limit (§9.3)
  showHeader: boolean;
  widthPercent: number;    // 0 => match the input's width
  columns: DropdownColumn[];
};
```

Note `dropdown_columns_allignment` and `dropdown_columns_visiblity` — both
misspelled in the API payload. Read them as they are; correct the names only
on our side of the boundary.

**`ensureSomethingVisible` is not optional.** A definition where every column
is `visible: false` is not a request for an empty popup; it is a dropdown
nobody finished configuring. Dropdown **38 (employees)** shipped exactly that
way and is still the only one, which is why Salesman and Agent opened blank on
three screens. The rule, from `nex_dropdown_config.h:116`: if nothing is
visible, force the completion column visible; if that isn't found, force
column 1 (index 1, not 0 — column 0 is the id); if there is only one column,
force that.

Port the comment with it. The next person to meet dropdown 38 should not have
to rediscover this.

---

## 4. Data flow — the two queries

```ts
useDropdownConfig(dropdownId)          // staleTime: Infinity, cached app-wide
useDropdownRows(dropdownId, search, params, { enabled: configLoaded && open })
```

**Cache the config globally by id, not per component.** The Qt widget refetches
the definition once per *instance*, so a screen with six dropdowns makes six
config calls and a grid popup makes one per cell. Keyed by `dropdownId` in
React Query (or a module-level `Map`), the app fetches each definition once for
its lifetime. This is the single biggest win of the port and costs nothing.

**Rows must be last-wins.** The Qt version has no cancellation: type `am`, then
`amb`, and if the first response is slower it lands second and repaints the
list with results for text the user has already moved past. React Query's
keying makes this correct by default; if you hand-roll, carry an
`AbortController` and drop any response whose search text is no longer current.

**Debounce 250 ms** on the search text only (`kSearchDebounceMs`). Opening the
popup fires immediately with the current text — a click on the chevron must
not wait a quarter second for an empty search.

**Config-not-yet-loaded is a state, not a dropped request.** `requestData()`
currently returns early when the config is missing (`nex_dropdown.cpp:387`),
and only the config callback re-issues it, and only when the popup happens to
be visible. Keystrokes in that window vanish. As dependent queries this
disappears: rows are `enabled` once the config resolves, and the search text
is just an input to the key.

---

## 5. Value model

### Single

```tsx
<NexDropdownSingle
  dropdownId={39}
  value={{ id, text } | null}
  onChange={(sel) => ...}       // null when cleared
  params={{ iemp_branch_id: branchId }}
  placeholder="All branches"
  disabled readOnly autoFocus
/>
```

Controlled. This deletes a Qt subtlety worth understanding before you delete
it: `setSelection()` sets the value *without* emitting `selected`, while
`handleRowChosen()` emits — because the load path must not re-trigger the
side effects a user pick triggers (`plan-react.md` §9.3 fights the same fight
with `QSignalBlocker`). With a controlled prop, a programmatic value change is
a re-render and fires no callback. There is nothing to block.

### Multi

```tsx
<NexDropdownMulti dropdownId={20} value={ids} onChange={(ids) => ...} />
```

`value` is **an ordered array in pick order**, and that ordering is the
contract — `NexDropdownMulti::selectedIds()` returns `m_order` specifically
because `m_chips.keys()` would come back key-sorted. Do not reach for a `Set`.

Re-picking an already-selected id is a **no-op**, not a toggle.

---

## 6. Interaction rules worth porting deliberately

### 6.1 Focus never leaves the input

The Qt popup is `NoFocus`, parented to the top-level window, positioned in its
coordinate space, and navigated entirely through an event filter on the line
edit. Do not port the mechanism — `floating-ui` and a portal do the placement,
and a fixed-position portal cannot steal focus the way a Qt tool window can.

Do port the **model**: the input keeps focus at all times, the list is a
suggestion surface, and clicking a row must not blur the input (`onMouseDown`
+ `preventDefault`, not `onClick` alone). This is also exactly the ARIA
combobox pattern, so correctness and accessibility land together:
`role="combobox"`, `aria-expanded`, `aria-controls`, `aria-activedescendant`
pointing at the highlighted `role="option"`.

### 6.2 Keyboard

| Key | Behaviour |
|---|---|
| any printable | opens the popup if closed, then debounced search |
| ↓ / ↑ | move highlight, clamped at both ends (no wrap — `moveSelection` clamps) |
| Enter | choose the highlighted row; **only when the popup is open** |
| Esc | close the popup, keep the text and the focus |
| Backspace | see 6.3 |
| Alt+C / Alt+A | master entry (§7) |

Enter must fall through to the form when the popup is closed, or Enter-as-Tab
navigation breaks on every screen that has a dropdown in it.

### 6.3 Backspace is different per variant

- **Single**: if a selection is committed, backspace **clears it entirely**
  (id, text, input) and reopens the popup — one keystroke, not a character
  edit. If nothing is committed the user is mid-type, so it deletes a
  character normally.
- **Multi**: if the search box is empty and chips exist, remove the **last**
  chip. Otherwise normal editing.

### 6.4 After a pick

Single: fill the input with the completion value, close the popup, fire
`onChange`, and **advance focus to the next field** (`focusNextChild()`).
Wire that to the same Enter-as-Tab helper `NexDialog::setup()` installs, so
one policy governs; do not hand-roll a second focus walker here.

Multi: append the chip, clear the search text, keep `onChange` firing, and
**close the popup after each pick**. (Arguably wrong — picking five tags means
five reopens — but it is current behaviour; change it deliberately or not at
all.)

### 6.5 Highlight resets to row 0 on every new result set

`onDataReceived` sets `currentIndex` to `(0,0)`. Keep it: Enter after typing
should take the top match without an arrow press.

---

## 7. Alt+C / Alt+A — the master behind the field

The operator adds a missing customer without leaving the field. A registry
maps dropdown id → `{ menuId, Entry }`:

```ts
registerDropdownMaster(39, { menuId: 10, Entry: CustomerEntry });
```

Rules, all of which must survive the port:

- **Permission comes from the master's own `menuId`** — `canCreate` gates
  Alt+C, `canEdit` gates Alt+A. A field must never be a back door into a
  master the user was not granted. Refuse out loud ("You don't have permission
  to add here"), because a silently dead shortcut is indistinguishable from a
  broken one.
- **No registration = the keys do nothing.** Not an error.
- Alt+A with nothing selected is a no-op, not an error.
- After the master saves: re-run the search. If the screen reported *which*
  record it saved, the **single** dropdown re-searches **by that id** and
  selects it — otherwise a rename leaves stale text in the field with no way
  to notice.
- The registry is the one place that may not be a barrel import: each feature
  registers its own dropdowns at startup, so this module never imports a
  master screen.

---

## 8. Formatting (`format.ts`, pure)

By `dataType`, from `NexDropdownModel::formattedValue`:

| dataType | Rendering |
|---|---|
| `NumberTS` | thousands-separated, 0 decimals |
| `NumericTS` | thousands-separated, 2 decimals |
| `Number` | plain number, no grouping, natural precision |
| `Date` | `dd/MM/yyyy`, raw string if unparseable |
| `DateTime` | `dd/MM/yyyy hh:mm AM/PM`, raw if unparseable |
| anything else | raw string |

Two rules underneath it: **read every value as a string first** (the API quotes
numerics — the same trap `NexJson` exists for on the Qt side), and **never
throw on a bad value** — an unparseable date renders as the raw text, it does
not blank the cell or break the row.

---

## 9. Fix rather than port

1. **Fixed popup height.** `applyConfigToView` sets the height to
   `rowHeight * maxVisibleItems` unconditionally, so a search matching one row
   shows one row and nine rows of void. Size to content, capped at
   `maxVisibleItems`.
2. **Popup width off the primary screen.** `widthPercent` is resolved against
   `QGuiApplication::primaryScreen()`. On the web it is a percentage of the
   viewport, and the `0 =>` case means "match the input width" — which is what
   floating-ui gives free.
3. **No paging.** `maxVisibleItems` is passed as the request `limit`, so a
   dropdown configured to show 10 rows can never reach the 11th match; the
   `page` parameter exists and is always 1. Decide once: either infinite-scroll
   the listbox (`page + 1` on scroll-end) or keep the cap and say so in the
   empty state ("showing first 10 — keep typing"). Do not leave it silent.
4. **No loading, empty, or error state.** The popup renders an empty table for
   all three. Give it a spinner row, a "No matches" row, and an error row with
   a retry — an empty table is the single most common "the dropdown is broken"
   report.
5. **`setDropdownId` doesn't clear the model or the selection**, so changing a
   dropdown's id mid-life leaves the previous list and value in place. In React
   the id is a prop: key the state off it and reset on change.
6. **Changing `params` doesn't revalidate the selection.** Set a branch filter
   after an employee was chosen and the field keeps showing an employee who is
   no longer in the list. Decide: clear the selection when `params` change, or
   document that the caller does it. Qt does neither.
7. **Multi has no chevron** (deliberately — the button is parented to the input,
   which floats after the chips). In React, anchor the indicator to the token
   frame and give both variants one.
8. Delete: `FlowLayout` (CSS `flex-wrap`), the hand-painted `QPainter` chevron
   and its commented-out twin, `installDropdownIndicator` /
   `updateDropdownIndicatorGeometry` geometry math, and the dead `eventFilter`
   stub at the bottom of `nex_dropdown.cpp`.

---

## 10. Tests

Pure, and worth being exhaustive about:

- `normalizeConfig`: the all-hidden case (**dropdown 38's real payload as a
  fixture**), completion-key fallback, single-column config, empty columns,
  `_alias` vs `_name` heading, both misspelled keys.
- `format`: every dataType, plus a quoted numeric (`"1234.5"`), an unparseable
  date, `null`, and a missing key.
- `idKey`: first column wins even when hidden.

Component, with a mocked client:

- Type → debounce → one request; type fast → one request, not five.
- Out-of-order responses: resolve the *first* request last, assert the list
  shows the *latest* search's rows.
- ↓↓↑ Enter picks the right row; Enter with the popup closed does not select.
- Backspace clears a committed single selection; backspace with empty text
  removes the last chip in multi.
- Clicking a row does not blur the input.
- Alt+C with no registration is inert; with a registration but no permission
  shows the warning and does not open the screen.
- Config is fetched **once** when two dropdowns share an id.

---

## 11. Phases

| # | Deliverable | Done when |
|---|---|---|
| 1 | `config.ts` + `format.ts` + `api.ts` | every §10 pure test passes; no UI exists |
| 2 | `useDropdownSearch` | debounce, dependent queries and last-wins proven against a mock |
| 3 | `DropdownPopup` + `useDropdownKeys` | keyboard matrix (§6.2) passes; loading/empty/error render |
| 4 | `NexDropdownSingle` | **a real dropdown id off the live API selects a value and reports it** |
| 5 | `NexDropdownMulti` | chips in pick order, backspace removal, no-op re-pick |
| 6 | `masters.tsx` + Alt+C / Alt+A | permission refusal proven both ways; save-then-reselect-by-id works |
| 7 | Polish | paging decision (§9.3) implemented, params/selection rule (§9.6) chosen |

Phases 1–4 are the deliverable that unblocks `plan-react.md`; multi and master
entry are not needed by Quotation Entry.

---

## 12. Do not port

- The event-filter focus machinery — `Qt::Tool`, `WA_ShowWithoutActivating`,
  `setFocusProxy`, the non-focusable child popup, `mapTo(host, ...)`.
- `FlowLayout` — 94 lines of Qt for `display:flex; flex-wrap:wrap`.
- The `QPainter` chevron, its commented-out duplicate, and the manual
  indicator geometry.
- `setSelection()`-doesn't-emit vs `handleRowChosen()`-does — a controlled
  `value` prop makes the distinction disappear (§5).
- Per-instance config fetching (§4).
- `QMap<QString, QWidget*> m_chips` + `QStringList m_order` + `QMap m_chipText`
  — three structures for one ordered list of `{id, text}`.
- The `virtual handleRowChosen()` inheritance seam. Two components sharing a
  hook, not a base class with two subclasses.
