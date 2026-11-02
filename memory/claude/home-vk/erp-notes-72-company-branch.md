---
name: erp-notes-72-company-branch
description: "Notes 72 (Company + Branch masters) built 2026-10-01 (755f52bc) + uncommitted April–March year fix; GSTIN lookup config, the React route's hardcoded GSP password, and test/data facts it exposed"
metadata:
  node_type: memory
  type: project
  originSessionId: 84c44cd4-9bd8-4f0e-b23a-37a6a86c2f0c
  modified: 2026-10-01T06:01:30.473Z
---

Notes 72 built 2026-10-01 (committed 755f52bc; status in repo `plan/notes-72-company-branch.md`): migration
`20261001120000_company_branch_notes_72` (applied to dev: comp_tds_applicable → boolean, GST reg type
CHECKs, current-FY backfill), company create seeds `fiscal_years`, company/branch restore routes +
delete/move guards, GSTIN↔state/PAN check (`settings/shared/gst-registration.ts`), signature stored as
a data URL, new `GET /gst/search` (`settings/gstinLookup`). Specs: `test/company-branch-notes-72*.e2e-spec.ts`.

2026-10-01 follow-up (uncommitted): first-year derivation aligned with the share's
`43_fiscal_year_seed.sql` (now `prisma/seed/Fiscal_Year_Seed.sql`): always 1 April–31 March,
non-April From / non-31-March To are a 400. The earlier version turned a Qt-sent 2026-03-20..28
into a year named '2026-2026' (ZT-CO-72B, since corrected on dev).

**Why / how to apply:**
- Any fy_year_name must be 'YYYY-(YYYY+1)', 1 Apr–31 Mar: ck_caa_fin_year / isValidAccYear assume it.
- `/gst/search` picks its provider from env `GST_LOOKUP_PROVIDER_CODE`, else the company's
  `fixed.gsp_company_service` row, else the only active provider. Dev has TWO active providers, so it
  answers 503 until that env is set. Never call it un-mocked in a test: it hits the real GSP.
- The React client's `app/api/gst/search/route.ts` has the TaxPro ASP id/password hardcoded as
  fallbacks (same values as `fixed.gsp_provider_master`). They were flagged to the user for rotation.
- `test/opening-balance-http.e2e-spec.ts` "null differenceLedgerId" failed before notes 72: a shared
  (company NULL) OPENING_DIFFERENCE mapping was added on 2026-09-15.
- Acme Foods' comp_pan_no (ABCDE1234F) doesn't match its GSTIN, so a full save of Acme now returns 400.
- While this was being built, someone ran `git add` on part of the uncommitted work (index state
  `MM`). Don't assume the index is yours; check `git status` before committing.
Related: [[erp-notes-70-item-masters]], [[erp-server-http-testing-without-credentials]].
