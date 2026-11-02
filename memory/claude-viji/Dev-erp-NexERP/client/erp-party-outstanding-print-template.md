---
name: erp-party-outstanding-print-template
description: "PARTY_OUTSTANDING_A5 — rev 1 (2026-10-15) all providers; rev 3 (2026-10-28, user asked: ALL data bands SQL) head + company + parties stored SQL, a second copy of the report rules kept in step by a parity harness; 20k-char cap nearly hit; cast-to-int refusals"
metadata:
  node_type: memory
  type: project
  originSessionId: 32165694-d2ce-4bcf-b1a6-dedf47d77f47
  modified: 2026-10-28T10:36:10.331Z
---

Handover folder: `/home/vk/Dev/erp/docs/print-templates/party-outstanding-a5/` (README, deploy
script, layout JSON, params, `datasets/0-head.sql` + `1-company.sql` + `2-parties.sql`, `verify/`).

**Rev 1 (2026-10-15):** `head` / `parties` were code providers
(`print-render/data/providers/party-outstanding.providers.ts`) calling
`PartyOutstandingService.export({shape:'PARTIES'})` — chosen so "pending as on D" had ONE definition.

**Rev 2 (2026-10-28, the user asked for the data band in SQL):** head + parties stored SQL.
**Rev 3 (same day, user: "change all the data band as SQL"):** `company` (letterhead) is SQL too —
`datasets/1-company.sql`, the company.profile provider's 23 columns by `:company_id` (server-owned,
so tenant-safe). The user wants NO providers in this design. Same layout/prompts/field names, so the client is untouched. Rev 3 published in
dev via `POST /print-templates/create` (full header, new PUBLISHED version, no ptlPublishedRevId —
see [[erp-print-template-save-clears-header]]). The deploy script adds the all-SQL revision to a DB whose published
revision has any non-SQL dataset, or installs fresh; no server release needed any more. Providers still in the server
(rev 1 history) — removing them was NOT discussed with the user.

- Both queries share one CTE chain word for word (p, k, grp, pty, cand, bill, ob, pa, pdc, pf) =
  `reports/party-outstanding/party-outstanding.sql.ts` + ageing.ts written out. **When the report
  rules change, change both files and run `verify/parity.ts`** (tsx from the server dir): service
  export vs the datasets via the engine's own binder, 36 filter sets, in one ROLLED-BACK tx that
  restores soft-deleted bills + adds synthetic PDC/chain/post-dated/tender rows. Dev ids only.
- `head` is 19,571 of the 20,000-char `ck_ptd_sql_size` cap — trim comments before extending.
- Stored SQL cannot RAISE: refusals cast the sentence `::int` in head's WHERE (inside CASE, the
  value from a MATERIALIZED CTE so the planner cannot fold it); the dialog shows
  `invalid input syntax for type integer: "Party-wise Outstanding refused - …"`.
- `p` is `NOT MATERIALIZED` (prompts fold to constants in the unnamed-statement custom plan);
  `k` is MATERIALIZED. Indian money in SQL: `to_char(x,'FM99,99,99,99,99,990.00')`.
  Bucket index = `by_due::int + width_bucket(age - 1, edges)`.
- **LIVE VPS (2026-10-28, user authorised):** the user had made "Party-Wise Outstanding A5" (code
  PARTY_OUTSTANDING_A5, shared, NO assignment row) by hand in the designer — exact copy of rev 3's
  layout + SQL but `ptv_params` NULL → "has no prompt named 'as_on'. It asks for: nothing." Fixed with
  `party-outstanding-a5-patch-existing.sql` (adds a published revision with the 18 prompts + SQL
  datasets, keeps name/assignments): live rev 2 = 01a18214-12d2-75a9-b81b-97fdf6993037. Backup:
  `/root/print-template-backups/print-templates-before-party-outstanding-fix-20261028-160427.dump`.
  Live DB = `erp` via `/opt/erp-server/app/.env` DATABASE_URL (user erp_app). Side effect left: a
  `pm2 ls` as root spawned an EMPTY root PM2 daemon (killing it was blocked; harmless).
  Copying a design by hand drops prompts: the layout JSON holds no ptv_params.
- Dev data was thin on 2026-10-28: a peer session soft-deleted ~300 purchase/sale bills that day.
- Checked: parity 36/36; rev 3 vs rev 1/2 PDF text identical for 7 sets (up to 8 pages) + 2 other
  companies' letterheads; company SQL == provider row for all 15 dev companies; deploy dry-run 4 ways.
- Save guard trap hit: a `:company_id` inside a `--` comment is refused (PRINT_TEMPLATE_SQL_QUOTED_PARAM). Related: [[erp-party-outstanding-port]], [[erp-ledger-statement-print-template]].
