---
name: erp-grep-skips-binary-files
description: "grep silently skips two large source files as \"binary\"; always use grep -a for repo-wide audits"
metadata: 
  node_type: memory
  type: project
  originSessionId: 55c8e9da-98b5-4bec-a01d-43350f1fea66
  modified: 2026-07-27T05:34:25.104Z
---

Two source files in the ERP client contain a byte that makes GNU grep treat them
as binary, so they are **silently omitted** from `grep -rn` / `grep -rl` results
with no warning:

- `features/masters/inventory/item/item-master-page.tsx` (~205 KB)
- `features/masters/settings/dropdown-designer/dropdownConstant.ts`

`file` reports both as ordinary "JavaScript source, Unicode text, UTF-8 text",
so the cause is not obvious. Adding `-a` (`grep -ran ...`) finds the matches.

**Why:** on 2026-07-27 a repo-wide audit of right-click handlers reported 18
call sites; item-master-page.tsx held a 19th that plain grep never listed. The
same audit for z-index missed nothing only by luck (that file has none).

**How to apply:** for any exhaustive repo-wide sweep in this project — counting
call sites, verifying a migration is complete, auditing before a refactor — use
`grep -a`. Confirm completeness afterwards with a second `grep -ra` for the old
pattern expecting zero hits. Related: [[erp-client-ui-browser-verification]].
