---
name: erp-pasted-design-notes-need-verification
description: "ERP server: the user pastes long SQL design notes to be turned into migrations; they are drafted against an idealised schema and must be checked against the live DB first"
metadata: 
  node_type: memory
  type: project
  originSessionId: 7662a923-c4b5-499c-a215-d4a6f91491a9
  modified: 2026-09-21T05:32:45.452Z
---

The user's workflow in `/home/vk/Dev/erp/ERP server` is to paste a long, heavily-commented SQL design note (a numbered "chain" file: 11 GSP integration, 13 loyalty points engine, …) and ask for a Prisma migration + models. These notes are written against an idealised version of the schema and reliably contain a few claims that are false for the live database.

**Why:** they are authored outside this repo and are not run before being handed over. Applying one verbatim produces either a failed migration or, worse, a silent no-op.

**How to apply:** before writing anything, verify against the live DB (`PGPASSWORD` from `.env`, `psql -h localhost -U postgres -d ERP`) — every referenced table, every PK column name, every function called, and whether target columns already exist. Classes of error seen so far:
- a helper function that doesn't exist (see [[erp-partition-helper-correction]]);
- **spelling drift in column names** — the note targeted `gde_cancelled_on`, the column is `gde_canceled_on` (one `l`). This one is severe when the name is used as *data* (the GSP field map stores a target column name read straight into an UPDATE), because it fails only at run time on a rarely-exercised path;
- **internal self-contradiction** — the loyalty note says at length that it creates no triggers, yet its `COMMENT ON` texts say "Trigger-maintained" and one paragraph cites a `tr_lld_refresh` that nothing creates. Align the COMMENTs to what is actually built; they are what the next person reads out of `\d+`;
- **a `DEFAULT` that its own `CHECK` forbids** (`lgd_qty NOT NULL DEFAULT 0` under `CHECK (lgd_qty > 0)`);
- tables the note ALTERs that don't exist yet, where `ALTER TABLE IF EXISTS` is a silent no-op;
- **`DROP COLUMN` on live tables whose data the note assumes lives elsewhere.** The sales-alters note dropped `sale_bill.sb_posted_on` because "txn_status_log holds it" — true for 19 of the 20 rows that had it. ALWAYS count the non-null values and confirm the claimed new home actually covers them; backfill the gap first, then drop. A drop is the one mistake with no undo.

Follow the note's design intent; correct its facts, and say plainly in the reply which things were changed and why.

**A schema change is not done when the migration applies.** Dropping a column or retyping one breaks TypeScript that reads it, DTOs that publish it, specs that assert on it, and — the one nothing catches at compile time — the grid SQL stored as TEXT in `fixed.grid_details`, which fails with a 42703 the first time somebody opens that screen. Run `npm run typecheck` and grep the repo (`src/`, `prisma/seed/`) for every dropped name before calling it finished.
