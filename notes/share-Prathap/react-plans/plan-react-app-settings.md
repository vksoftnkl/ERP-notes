# Application Settings — React port plan

Scope: **one screen**, `AppSettingsEntry`
(`src/modules/admin/settings/app_settings_entry.cpp`), ported to React/TypeScript
against the unchanged backend.

This is a **delta plan**, but it shares almost nothing with `plan-react.md`. That
document specifies a *voucher* — pricing, grids, charge distribution, a draft
reducer over line items. This screen has none of that. What it does have is one
idea the voucher screens do not:

> **The screen is generated from the catalog. There is no list of settings in the
> client, and there must never be one.**

Every row — its label, its control, its permitted values, how deep it may be
overridden — comes from `app_setting_def`. Adding a setting is one INSERT and a
one-line binding; it is **not** a front-end change. If the port ends up with a
`SETTINGS = [...]` array anywhere, it has failed.

Read §2 (the scope model) before anything else. Everything awkward about this
screen follows from it.

---

## 1. What the screen is

A two-pane editor over the setting catalog. Left: a module → group tree. Right:
the selected group's settings, each a label, a control, and a badge saying where
its current value came from. A scope bar above chooses **which layer is being
edited**. One Save.

33 settings live today across SYSTEM, SALES, INVENTORY, PRINTING and LOYALTY.
That number is data and will move; nothing in the port may depend on it.

---

## 2. The scope model — read this first

Five layers, narrowest wins:

```
USER  >  DEVICE  >  BRANCH  >  COMPANY  >  GLOBAL  >  the catalog default
```

`fn_app_settings()` resolves them server-side and returns one flat answer per
key. **The client never merges layers.** Three layers merged in a front-end store
is exactly where this rots.

Two rules that shape the whole UI:

**The scope bar does not have a "level" control.** The level is *derived* from
which of Branch and Counter are set:

| Company | Branch | Counter | Editing at |
|---|---|---|---|
| set | All branches | All counters | `COMPANY` |
| set | a branch | All counters | `BRANCH` |
| set | a branch | a counter | `DEVICE` |

Derive it; never store it. The Qt screen has `currentScope()` doing exactly this
and it is the right shape.

**`asdMaxScope` caps how deep each setting may go.** A setting whose max scope is
`COMPANY` cannot be set per branch. When the bar is deeper than a setting allows,
that row renders **read-only with an explanation**, not hidden — an operator
hunting for "Regional language" at branch level must be told it is company-wide,
not left wondering where it went.

```ts
const depth = { GLOBAL: 1, COMPANY: 2, BRANCH: 3, DEVICE: 4, USER: 5 } as const;
const editable = depth[currentScope] <= depth[s.asdMaxScope];
```

The screen edits `COMPANY | BRANCH | DEVICE` only. `USER` scope exists in the
model and is deliberately **not** editable here — a person's own preferences
belong in a profile screen, not in an administrator's tool. `GLOBAL` is
installation-wide and is schema-managed.

---

## 3. The API contract, and the four ways to get it wrong

Three endpoints. Every one of the traps below cost real debugging on the Qt
screen; none of them is guessable from the DTOs alone.

### 3.1 Read — `GET /app-setting-values/effective`

Query: `companyId`, `branchId`, `deviceId`, `userId` — all optional, all
narrowing. Returns one row per catalog key:

```ts
type EffectiveSetting = {
  asdKey: string; asdModule: string; asdGroup: string;
  asdLabel: string; asdDescription: string;
  asdDataType: 'BOOL' | 'INT' | 'DECIMAL' | 'TEXT' | 'JSON';
  asdMaxScope: Scope;
  asdSortOrder: number;
  asdNeedsRelogin: boolean;
  asdAllowedValues: string[] | null;      // non-null ⇒ render a select
  asdMinValue: number | null;
  asdMaxValue: number | null;
  value: unknown;                          // TYPED — see below
  source: 'DEFAULT' | Scope;               // where the winning value came from
  override: { asvId: string; asvScope: Scope; ... } | null;
};
```

**`value` arrives TYPED** — a real JSON `true`, a real number — because the
resolver casts per `asdDataType`. **Writes go back as TEXT.** That asymmetry is
the contract, not an oversight, and §3.2 explains why.

`source` and `override` are what the badge reads. `override` non-null means a row
exists *at some layer*; `override.asvScope === currentScope` means it exists **at
this one**, which is a different question and the one that matters for §3.2.

### 3.2 Write — `POST /app-setting-values/create`

Body is `{ data: [ ... ] }`, an array, upserted per entry's scope target.

**Trap 1 — `asvValue` must be a STRING, whatever the data type.**
`app_setting_value.asv_value` is a `text` column, and the catalog stores its own
defaults as text beside it (`'true'`, `'15000'`, `'PRINT'`). The allowed-values
trigger settles it: it tests `asd_allowed_values @> to_jsonb(NEW.asv_value)`, a
jsonb array of **strings**, so any setting with an allowed list is refused
outright unless a string arrives.

```ts
const toText = (v: unknown, t: DataType): string => {
  if (v == null) return '';
  if (t === 'BOOL')    return v ? 'true' : 'false';        // lower case, as seeded
  if (t === 'INT')     return String(Math.round(Number(v)));
  if (t === 'DECIMAL') return trimZeros(Number(v).toFixed(6)); // never exponent form
  return String(v);
};
```

`DECIMAL` must not go out in exponent notation — `1e-05` in a text column is a
value every later reader has to guess at. Fix the decimals, then trim.

**Trap 2 — send `asvId` ONLY when updating an override at THIS scope.**
The endpoint upserts on the scope target; a supplied `asvId` means "update that
row". A client-minted uuid is by definition an id the server cannot find:

```
{"errors":[{"field":"data[0].asvId","message":"No override found with id …"}]}
```

`override != null` is **not** the test — the override may sit at a broader layer
and editing narrower has to create a new row:

```ts
const updatingHere = s.override?.asvScope === currentScope;
...(updatingHere ? { asvId: s.override!.asvId } : {}),
```

Omit the key entirely. `asvSettingKey` and `asvScope` are required whenever
`asvId` is absent, and harmless when present.

**Trap 3 — exactly ONE scope id, the one the scope names.** The others must be
`null`. The server says it plainly:

> `asvBranchId must be omitted when asvScope is DEVICE — an override carries the id its scope names and nothing else`

```ts
asvCompanyId: scope === 'COMPANY' ? companyId : null,
asvBranchId : scope === 'BRANCH'  ? branchId  : null,
asvDeviceId : scope === 'DEVICE'  ? deviceId  : null,
asvUserId   : null,
```

**Note the live database disagrees with the schema file here.** `10_app_settings.sql`
in the share still carries an older cumulative `ck_asv_scope_ids` (company NOT
NULL at every layer below GLOBAL); the deployed constraint is one-id-per-scope.
Trust the deployed one; the file needs correcting.

### 3.3 Reset — `DELETE /app-setting-values/delete?asvId=`

Removes one override so the setting falls back to the layer above. Only offer it
when `override?.asvScope === currentScope` — you cannot reset a value set
somewhere else from here.

---

## 4. Module layout

```
src/features/appSettings/
  AppSettingsScreen.tsx
  components/
    ScopeBar.tsx            // company / branch / counter + search + changed-only
    ModuleTree.tsx          // module -> group, with per-group counts
    SettingGroup.tsx        // one card, striped rows
    SettingRow.tsx          // label, description, control, source badge, reset
    SettingControl.tsx      // §5 — the datatype -> widget switch
    ScopeChip.tsx           // "EDITING AT BRANCH LEVEL"
  state/
    useSettings.ts          // fetch + dirty map + save/reset
    scope.ts                // derive scope, depth, editability — PURE
  api/
    appSettings.ts          // effective / create / delete
  payload/
    buildOverride.ts        // §3.2, all three traps, PURE
    toText.ts               // typed value -> text, PURE
  session/
    applyToSession.ts       // §7 — push saved values into the running app
```

`scope.ts`, `toText.ts` and `buildOverride.ts` are **pure and tested first**.
They are where every one of the §3 traps lives, and they are all trivially
testable without a browser.

---

## 5. The control is chosen by data, never by key

```
BOOL                          -> switch
INT                           -> number input, step 1,  clamped to min/max
DECIMAL                       -> number input, step .01, clamped to min/max
TEXT + asdAllowedValues != null -> select of exactly those values
TEXT                          -> text input
JSON                          -> textarea + parse check (none live today)
```

Never switch on `asdKey`. The moment one setting gets a bespoke control by key,
the catalog stops being the source of truth and the next setting needs a
front-end release.

Client-side validation **mirrors** the server, it does not replace it: min/max
from `asdMinValue`/`asdMaxValue`, membership for `asdAllowedValues`. The server
re-checks all of it and is the authority.

---

## 6. State, dirty tracking, and reads

One record keyed by `asdKey`. Two values per setting: the **effective** value
from the server, and the **pending** edit if the user has touched it. Never
overwrite the first with the second — the badge and the reset button both need to
know what the server currently says.

```ts
type Row = EffectiveSetting & { pending?: unknown; dirty: boolean };
```

After a save or reset: **re-fetch, do not patch in place.** The server decides
what the effective value now is and at which layer it sits, and a local guess
about the fallback will eventually be wrong.

**Coalesce the reads.** Changing company repopulates branch and counter, each of
which fires its own change; the Qt screen fired three GETs of `/effective` for
one company change and the last to arrive won. Debounce ~80ms.

**Give the first read a floor.** The company list is fetched asynchronously, and
if that call fails or comes back empty the chain that would trigger the first
read never runs — leaving a screen that shows nothing, for ever. Read once with
the session's own company before any lookup resolves; refine when they land. The
Qt port hit this exact regression.

---

## 7. Saving must not require a re-login

The Qt screen wrote the override, re-read its own list, and stopped — so **every**
setting needed a sign-out, including the ones the catalog marks
`asdNeedsRelogin: false`, while the success message implied otherwise.

After a successful save or reset, re-read the **session's own** effective
settings and push them into whatever the React app's equivalent of `AppSession`
is (a settings context / store).

Two things to get right:

**Guard it.** Only re-apply when the scope bar is pointed at the context this
session is actually inside — same company, and branch/counter either "all" or
this session's own. An administrator editing another company's settings must not
have those values take hold in their running app.

**Reset to defaults first, then apply.** Applying only writes the keys it is
given, so a just-deleted override would otherwise linger.

The message should then say what is true: *"n settings saved, and in effect now"*,
naming only the `asdNeedsRelogin` ones as still needing a sign-in.

---

## 8. The scope bar

**Company** — from `master-lookups/name-id/all-masters?module=companies`. Opens
on the session's company; an administrator may pick another. Show the **name**;
the Qt screen showed the literal words "Current company", which says nothing
about which company is being edited.

**Branch** — `master-lookups/branches/by-company/{companyId}`, rebuilt whenever
company moves.

**Counter** — `…?module=devices`. **Not company-scoped, and it cannot be**: the
lookup takes no company parameter, and adding one makes it return *nothing*
rather than filtering. So it lists every till on the installation and picking one
from another company would write that company's device an override.
`/device-list-masters/list` does filter on `devCompanyId` and is the right
source — it needs its grid configured first (`grids/device_list_grid.sql`). Port
against the devices lookup, switch when the grid lands.

A counter with no branch selected is incoherent — a till belongs to a branch —
so warn, and treat the scope as BRANCH until one is chosen.

---

## 9. Backend gaps this screen inherits

1. **No endpoint creates a catalog definition.** `/app-setting-values/*` operate
   on values only; `app_setting_def` is schema-managed and the API role has no
   INSERT on it. Correct, and worth stating: a new setting is a migration plus a
   client binding, never a runtime action. There is no "add setting" button and
   there should not be one.
2. **`ck_asv_scope_ids` drift** between the deployed database and
   `10_app_settings.sql` (§3.2).
3. **The devices lookup cannot be scoped** (§8).
4. **`source` vs `override`** are two different answers and the DTO does not
   spell out the difference; §3.1 does.

---

## 10. Tests

Pure units first, and they cover every trap:

- `toText` — BOOL both ways, INT rounding, DECIMAL never exponent and trailing
  zeros trimmed, TEXT passthrough, null → `''`.
- `buildOverride` — new override omits `asvId`; override at *this* scope sends
  it; override at a *broader* scope omits it; exactly one scope id non-null for
  each of COMPANY / BRANCH / DEVICE.
- `scope.ts` — the derivation table in §2, and `editable` against every
  `asdMaxScope`.

Then one integration test against a running backend that does what the Qt port
was verified with, in order:

```
COMPANY new (no asvId)                  -> 201, one row,  value stored as text
COMPANY again with its asvId            -> updated in place, still one row
BRANCH  new, company sent as null       -> created, does not touch the company row
DEVICE  new, branch sent as null        -> created
```

Then delete all four and confirm the table is as it was.

---

## 11. Phases

1. `scope.ts`, `toText.ts`, `buildOverride.ts` + their tests. No UI.
2. Read path: `/effective`, module tree, groups, read-only rows with badges.
3. Controls and dirty tracking (§5, §6).
4. Save and reset, with all three traps (§3.2), then the integration test (§10).
5. Scope bar and its cascade (§8), with the debounce and the floor read (§6).
6. Session re-apply (§7).
7. Search, "changed only", and the max-scope read-only explanation.

Phase 4 before phase 5, deliberately: saving correctly at one scope is worth more
than browsing three, and the traps are all in the save.

---

## 12. Do not port

- **`NexLineEditCapz` capitalization.** It is a Qt font trick. In React, honour
  `system.font_capitalization` with a CSS `text-transform` for display and a
  normalise-on-blur for the stored value — and keep the rule that a Tamil word
  carrying a left-joining vowel sign (U+0BC6, U+0BC7) is left exactly as typed,
  because capitalizing one mangles it.
- **The striped-row stylesheet.** The Qt screen needed `QWidget#settingRow`
  scoping because an unscoped stylesheet propagates to children and erased every
  checkbox indicator. CSS has none of that problem; just style the rows.
- **The fixed 1460×900 dialog.** Make it a routed page.

