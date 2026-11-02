---
name: erp-company-edit-defaults-not-a-bug
description: "Company edit form falls back to create defaults for blank stored fields (branch doesn't) — harmless, the DB columns are NOT NULL with the same defaults; don't re-report"
metadata:
  node_type: memory
  type: project
  originSessionId: 6918836f-3415-408d-9833-2afd302bc492
  modified: 2026-10-27T05:54:15.905Z
---

`mapCompanyFormValues` (features/masters/settings/companies/companies.ts) fills a
saved company's blank text fields with the create defaults, while branches show
exactly what's stored ("A saved record shows what it holds"). A test-writing
review on 2026-10-27 flagged this as a bug; it isn't.

**Why:** the only text fields with non-empty defaults are comp_country,
comp_currency_code, comp_locale_code and comp_aato_class, and all four are
NOT NULL in `public.companys` with DB defaults identical to the form's
('India', 'INR', 'en-IN', 'LE_5CR'). A stored value can never be blank, so the
fallback never shows or saves anything that isn't already true.

**How to apply:** don't "fix" it to match branches. Switching to the branch
rule would be riskier: any key the GET doesn't return would then show blank and
be saved back as blank. Re-check only if one of those columns becomes nullable.
Related: [[erp-company-branch-qt-parity]].
