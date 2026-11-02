---
name: nex-date-edit-keys
description: NexDateEdit ↑/↓ = ±1 day; calendar popup cleaned (no week numbers/grid, short day names, today bold, Enter picks)
metadata:
  type: project
---

2026-10-22, user: "use up/down arrows to change ±1; the calendar popup looks bad". In NexDateEdit (src/shared/widgets), an eventFilter on the line edit makes ↑ the next day and ↓ the day before. An empty field starts at today, a read-only one does not move, and each step emits dateEdited. The calendar has NoVerticalHeader (no week numbers), ShortDayNames, no grid, a minimum size of 330×240 (so "Mon"/"Wed" fit), and today in bold + underline. Enter (activated) picks a day like a click, and focus goes into the calendar so the arrows walk the days. The colours are still the theme template's QCalendarWidget rules; the selected-day tint is pale, and fixing that would be a template change, which must be asked about before publishing. Test: tests/dateedit/run.sh (8/8; SNAP_DIR saves calendar.png with the live theme). Related: [[nex-date-edit-signals]].
