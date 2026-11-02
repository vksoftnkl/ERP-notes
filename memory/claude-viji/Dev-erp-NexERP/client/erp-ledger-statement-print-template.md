---
name: erp-ledger-statement-print-template
description: "LEDGER_STATEMENT_A5 (2026-10-15) — a REPORT printed through the document print path (docId = ledger, period as ptv_params prompts), the PAGE-max c/f trick, jexl traps, and the client params plumbing"
metadata:
  node_type: memory
  type: project
  originSessionId: 32165694-d2ce-4bcf-b1a6-dedf47d77f47
  modified: 2026-10-15T05:41:18.416Z
---

Built 2026-10-15 like [[erp-receipt-print-template]]: purpose `LEDGER_STATEMENT`
(REPORT / LEDGER_STATEMENT) via server migration `20261015120000_ledger_statement_print_purpose`
(applied to dev), template `LEDGER_STATEMENT_A5` shared, **rev 2 published** in dev (rev 1
superseded but still status PUBLISHED — the API cannot retire it, see
[[erp-print-template-save-clears-header]]), GLOBAL / PRINT assignment. Handover:
`/home/vk/Dev/erp/docs/print-templates/ledger-statement-a5/` (deploy script dry-run tested in
rolled-back txns, incl. the "purpose missing" path). Not known to be applied on live.

- **A report through the document path:** docId = `led_id`; from_date/to_date (DATE,
  required), `stmt_branch_id` (NOT `branch_id` — unanswered falls back to the session's
  branch, killing "All branches"), 4 BOOLEAN flags. Flag defaults are ALSO in the SQL
  (`COALESCE`) because the designer's PtvParam type has no `default` and a re-save drops it.
- Datasets mirror ledger-statement.service.ts exactly; verified row-for-row against `/export`
  + `/header` (scratchpad comparators, gone). Re-verify the same way if the service changes.
- **Balance c/f per page:** PAGE_FOOTER FIELD with `aggregate {fn:max, scope:PAGE, over:
  row.sl_no}` and value `lines[.sl_no == agg.value].balance_amt` (sl_no must be int4: `|where`
  and node-pg int8-as-string). A b/f at the top of a page is impossible (header drawn first).
- jexl traps: `x|length ? a : b` does not parse — parenthesise the condition; `.prop` on an
  array reads the FIRST element.
- Client: `DocumentPreviewDialog` gained `params` (keyed by JSON in the render-once guard),
  `PrintOptionsDialog` forwards `targets[0].params`; screen Print/PDF/F11 send
  `printPrompts(filters, display)` from the APPLIED filters. The print dialogs handle no
  keys, so the screen's Escape now closes the dialog first. `/print-render/log` takes no
  params (period not logged).
- Found, not fixed: ledger-statement `coreRows` joins user_master with the regex-guarded
  `::uuid` cast → 22P02/500 for ledgers whose vouchers have `avh_created_by` like 'VKPOS'
  (Cash In Hand from October, Kvb bank). Fix = `createdByJoinSql`. See [[erp-ledger-statement-port]].
- **2026-10-29, live refusal "This revision has no prompt named 'from_date'. It asks for: nothing."**
  Live had `LEDGER_STATEMENT_A5` rev 1 (ptv `01a1815f-9936-7cdd-83b0-6375598257c9`, created
  2026-10-28, NOT by the deploy script) whose datasets + body equal dev rev 2 but `ptv_params` was
  NULL. FIXED on live by UPDATE-ing dev's 7 prompts in (user-authorised; backup
  `/root/ledger-stmt-a5-rev1-backup-20261029.tsv`). Render reads the DB per call (`no-store`), no
  restart needed. Whatever copied it to live drops `ptv_params` — check prompts after any such copy.
  Client also gained `answersForRevision` (printOptions.ts): sends only the rendered revision's
  declared prompts, so a promptless design no longer refuses (uncommitted at the time).
