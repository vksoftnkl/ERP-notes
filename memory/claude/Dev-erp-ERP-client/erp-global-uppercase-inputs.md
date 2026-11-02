---
name: erp-global-uppercase-inputs
description: All free-text inputs app-wide are rewritten by one global capture-phase listener, driven by the system.font_capitalization setting; how to opt a field out
metadata:
  type: project
---

Every free-text field in the ERP client rewrites what is typed into it. There is no
shared `Input` component (~76 files render raw `<input>`), so it is done once in
`components/feedback/global-text-capitalization.tsx`, mounted in `app/layout.tsx`
next to `GlobalLoader`. (Was force-uppercase only, 2026-08-22; became
setting-driven 2026-08-26 — the old `global-uppercase-input.tsx` is gone.)

**The mode comes from the setting, not from code:** `system.font_capitalization`
(SYSTEM/Display, TEXT, default `TITLE`, max scope USER, no relogin) with four values
— `TITLE` (first letter of each word upper, rest lower), `UPPER`, `LOWER`, `MIXED`
(no listener at all). Transform + parser live in `lib/text-capitalization.ts`
(unit-tested). Tamil words are exempt in every mode — caseless script, and a
left-joining vowel sign has no "first letter".

**Where the value comes from:** `components/layout/session-app-settings.tsx`, mounted
above the routes, reads `/app-setting-values/effective` for the session's own
company+branch+counter+user and dispatches `appSettingsApplied`; the listener reads
it with `selectAppSettingText`. This is the ONLY app-wide settings read — before it,
`state.appSettings` was populated only by the settings screen after a save. Saving an
override invalidates the `AppSettings` tag, so a change applies with no sign-in.
`StoreBootstrap` now rehydrates `userInfo` too, or a reload loses `deviceId` and the
counter layer stops resolving.

**How the rewrite works:** a capture-phase `input` listener on `window` rewrites the
value *before* the event reaches React, using the `HTMLInputElement.prototype` value
setter — assigning `target.value` directly would update React's instance-level value
tracker and React would swallow the `onChange`. Transforms that change length (ß→SS)
are skipped so the caret stays put.

**Shift is a per-character escape hatch** (2026-09-04): holding Shift over a
printable key pins that character's case, so `McDonald` survives a TITLE pass and a
lone capital survives LOWER. Pins are per-element (`WeakMap`), keyed by code-unit
offset, and are slid across each edit by `diffEdit`/`remapPinnedPositions` in
`lib/text-capitalization.ts`; `applyCapitalization` takes them as a third argument.
This needs two more capture listeners: `keydown` (sets the flag only for
`key.length === 1` with no Ctrl/Meta/Alt, so Ctrl+Shift+V does not pin) and
`beforeinput` (snapshots the value while the field still holds the old text — the
only moment the edit can be read off exactly). Only a single-character insertion
pins, so a paste or autofill never does.

**Excluded** (user's call): credential fields (`autocomplete` matching
username/password/one-time-code/webauthn — this also covers a reveal-password toggle
flipping the field to `type="text"`), email-ish, search/filter-ish (`type=search`,
`role=searchbox`, or "search"/"filter" in name/id/placeholder/aria-label), and
anything non-text (`input.type !== "text"`, or `inputMode` numeric/decimal/tel/email/url).

**Opt out:** `autocapitalize="off"` on the field, or `data-uppercase="off"` on any
ancestor (attribute name kept for compatibility). Do NOT add per-field
`.toUpperCase()` in onChange handlers — the global handler already covers it. All
four modes verified in a real browser against the dev stack; see
[[erp-client-ui-browser-verification]] and [[erp-app-settings-contract]].
