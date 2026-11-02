---
name: grid-timestamptz-shows-utc
description: Configured grids send a timestamptz as a UTC ISO instant and the Qt client shows its clock face (15:10 IST read 09:40) — emit IST text with offset in grid SQL
metadata:
  type: project
---

`ConfiguredGridSqlService.serializeRawQueryValue` passes a pg `Date` through, so a timestamptz column
leaves as `2026-10-09T09:40:00.000Z`; the Qt grid's DateTime cell shows 09:40 (notes 99 §3). `date`
columns are fine — `PgService` hands them back as `'YYYY-MM-DD'` text.

**Why:** the client does not convert the `Z` instant to local time.

**How to apply:** in grid SQL return `to_char(x AT TIME ZONE 'Asia/Kolkata',
'YYYY-MM-DD"T"HH24:MI:SS"+05:30"') AS x` (right whether the client honours the offset or drops it,
still sortable as text); the user's own grids 126/132/144+ use `to_char(..., 'DD-MM-YYYY HH24:MI')`
Text columns instead. Done for the settlement grids 138/139/141/142 + PICK ROW (migration
20261009170000). Still raw (would show UTC): grids 31, 62, 96, 125, 131 DateTime columns. Related:
[[db-session-timezone-ist]], [[configured-grid-param-convention]].
