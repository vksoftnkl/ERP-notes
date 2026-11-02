---
name: erp-notes-70-item-masters
description: "Notes 70 (item masters) built 2026-09-30, committed in 0626a3c7 (pushed); plus three ERP-wide gotchas it exposed: DTO own-property trap, Prisma insensitive-equals = ILIKE, grid/dropdown SQL rules"
metadata:
  node_type: memory
  type: project
  originSessionId: 6109bc63-7e3c-43d8-b369-81d2e4e0b065
  modified: 2026-09-30T13:02:21.921Z
---

Notes 70 is built and committed (bundled into 0626a3c7 "fix(sales): …", on origin/DEV-v1; re-verified 2026-10-01: e2e 14/14), status in repo `plan/notes-70-item-masters.md`: migration `20260930170000_item_masters_notes_70` applied to dev, shared tree rules in `src/modules/Inventory/utils/master-tree.helper.ts`, DELETE/RESTORE split on 7 masters, `test/item-masters-notes-70.e2e-spec.ts`. D2-D5 decided 2026-10-01 and built (UNCOMMITTED): migration `20261001100000_item_masters_notes_70_d2_d5` (applied to dev), category stores its 4 defaults, item CREATE inherits blank tax/HSN/base unit from group then category (`inheritClassDefaults`), dropdown 18 got `iexclude_id`, grid 50 + dropdown 19 order by `sec_sort`; e2e now 19 cases.

**Why:** three traps found here apply across the whole ERP server, not just these masters.

**How to apply:**
- DTO own-property trap: tsconfig targets ES2022, so every declared DTO field is an own property (`undefined`) even when unsent. `hasOwnProperty(dto,k) ? dto.k ?? null : stored` CLEARS omitted keys; use `dto.k !== undefined`. `NullableUuid` maps `""` → null, so `!== undefined` keeps "empty string clears".
- Prisma `equals` + `mode: insensitive` compiles to ILIKE: `%`/`_` are wildcards ("GST 0%" matched "GST 0.25%"). Use raw `lower(x) = lower($1)`.
- Stored grid/dropdown SQL: must start with SELECT, no comments, no `$N`. Grid params bind bare tokens to `$N`; dropdown params substitute literals, and an unbound bare token is a SQL error. An OPTIONAL dropdown param is safe as `NULLIF(NULLIF('itoken',''),'itoken'||'_x')` style (quoted token, inert when unbound).
- A Grid Master save (seen from the Qt client at 192.168.0.103, 2026-10-01) resends the grid_sql it loaded, silently reverting migration fixes — grid 67 lost its A3 WHERE that way (10:35; re-applied on dev 12:25). When a grid test regresses, check `fixed.grid_details.grid_modified_by/on` and audit.audit_log (`log_table_name = 'grid details'`) before suspecting the code.
- Seeds use `ON CONFLICT DO NOTHING`, so a drifted grid/dropdown row on a DB is only repaired by a migration.
Related: [[erp-one-price-table]], [[erp-live-vps-deploy]].
