# Notes 132: grid 119 is broken: its SQL has comments (2026-10-17)

**STATUS:** DONE + verified 2026-10-17 (tests/register grid119 16/16, popup119 3/3). Since the notes 130 change (2026-10-16 17:02), the ledger list in the Voucher Register shows nothing on every voucher type.

## What happens

Every call to grid 119 now fails:

```
GET /configured-grid-sql/run?grid_id=119&…&grid_param={"icompany_id":"019c8ea6-…","iside":"DR","itype_code":"PmtV"}
400  GRID_RUN_SQL_INVALID  "Invalid grid SQL: Comments are not allowed in configured query"
```

The new SQL has four `--` comment lines (the `-- notes 130: …` block and `-- ...and the money side …`). The grid runner refuses any SQL that has a comment. Grid 119 is the only grid with comments in `fixed.grid_details`.

## Ask

1. Save **`notes-132-grid-119.sql`** (beside this note) into `fixed.grid_details.grid_sql` for grid 119. It is your SQL of 10-16 with only the four comment lines removed; nothing else is changed. Checked by hand: PmtV Dr = 556 ledgers, PmtV Cr = 0 (as notes 130 wants).
2. Please also copy it into `grids/voucher_ledger_picker.sql`. That file is still the 2026-09-28 version (notes 130 item 3).

We can't write `fixed.grid_details`. Tell Viji when it is done. Claude will check grid 119 through the runner on all 8 types.
