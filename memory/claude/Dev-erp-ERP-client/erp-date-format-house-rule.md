---
name: erp-date-format-house-rule
description: "Every date the client shows is dd/MM/yyyy (user rule, 2026-10-28) — lib/date-format helpers, shared DateInput instead of native type=date, ReusableTable re-spells server DD-MM-YYYY text"
metadata:
  node_type: memory
  type: project
  originSessionId: 85490c3b-0039-4ccb-826f-a261323d7a50
  modified: 2026-10-28T13:45:13.172Z
---

User asked (2026-10-28) that dates read `dd/MM/yyyy` EVERYWHERE in the client. Before that, the
house style was Qt's `dd-MM-yyyy` plus native `<input type="date">` (browser-locale, so mm/dd/yyyy on
en-US / headless). Migrated app-wide in one pass (~220 files, uncommitted at the time).

**How to apply on new screens:**
- Display: `lib/date-format.ts` — `formatDate`, `isoToDisplayDate` (literal, no TZ), `formatLocalDate`,
  `formatDateTime` / `formatLocalDateTime` (`dd/MM/yyyy hh:mm AM`), `DATE_PLACEHOLDER`, `DATE_PATTERN`.
  `lib/time-format` `dateTime12Of` / `dayMonthTime12Of` also use slashes. Never hand-build `${d}-${m}-${y}`.
- Typing: `parseDisplayDate(text, {allowShortYear})` accepts `/ - .`, d/M, 8 digits, ISO → ISO.
- Inputs: never a bare `<input type="date">` / `datetime-local`; use `DateInput` / `DateTimeInput`
  from `@/components/design-system/ui` — `value`/`onChange(value)` are ISO strings (not events);
  `block` when the old input stretched (`width:100%`), `arrowStep={false}` + `noPicker` in entry grids.
  Its hidden native picker carries `data-date-picker-proxy` + tabIndex -1 — exclude it from any
  focus walk that doesn't filter tabIndex.
- Modal forms: `type: "date"` fields already render `DateInput` (dynamic-modal-form).
- Lists: `components/ui/table.tsx` runs string cells through `normalizeDateText`, so configured-grid
  SQL that writes `to_char(…,'DD-MM-YYYY')` shows slashes without touching the DB. Screens that draw
  grid rows in their own `<table>` must call `normalizeDateText` themselves (till screens use their
  local `displayText`, which also handles year-less `DD-MM hh:mm AM`; txn history dialog does too).
- Print designer defaults to `dd/MM/yyyy`; dev-DB print templates (RECEIPT_A5 etc.) and
  `docs/live-sync/*.sql` grid SQL still carry `dd-MM-yyyy` — not migrated, user not yet asked.

Wire/state/sort values stay ISO. Peer session staged all ~220 files twice mid-task — see
[[erp-workspace-layout]].
