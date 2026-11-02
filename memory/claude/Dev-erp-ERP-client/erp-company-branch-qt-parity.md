---
name: erp-company-branch-qt-parity
description: Company + Branch masters brought to the Qt forms (2026-10-02, plan/company-branch-qt-parity.md); decisions taken, deviations, server-side GSTIN lookup that 503s until configured; user said KEEP app/api/gst/search/route.ts
metadata:
  type: project
---

Implemented all five phases of `plan/company-branch-qt-parity.md` on 2026-10-02.
Branch now has `features/masters/settings/branches/module.ts` like the company.
Shared pieces live in `features/masters/shared/`: `gst-registration.ts`
(4 reg types incl. SEZ, mod-36 checksum, GSTIN/State/PAN validators),
`gstin-lookup.ts`, `fiscal-year.ts`, `edit-mode-field.ts`,
`use-soft-delete-actions.ts`, `server-error-text.ts`.

Decisions taken (plan's recommendations): D1 Qt rounding words (old
"rounding off/up" rows display the raw word and keep it on save, they do not
blank out); D2 column defaults (SMS off, negative stock on); D3 lock date
dropped; D4 stylesheet stays required; D5 two columns; D6 SEZ kept LOCAL to
company/branch — customer/supplier/ledger servers do not accept it; D7
signature is an image upload (key omitted when no new file = server keeps it).

Deviation: the branch GSTIN lookup only fills a BLANK Branch Name — branches in
one state share the company's GSTIN, so overwriting would collide names.

**Why:** the edit-mode sentinel `__editing` is a declared hidden field because
`buildInitialValues` drops undeclared keys on reset (openModal keeps them).

**How to apply:** GSTIN lookup for company/branch/customer/supplier/ledger now
goes through server `GET /gst/search`. `app/api/gst/search/route.ts` (the Next
route with hardcoded ASP creds) is KEPT: the plan's Phase 4 deleted it, and the
user said "do not delete the route.ts file" (2026-10-02) — it was restored, and
nothing calls it now. Never delete it as clean-up. The server route answers 503 until `GST_LOOKUP_PROVIDER_CODE` is set
in the server env (local server `.env` had none on 2026-10-02) — a lookup
"not configured" message on those screens is that, not a client bug.
Dropdown 26 (godowns) takes `{ibranch_id}` via the lazy hook's new `params`;
its SQL guards the token, so it stays `paramBound:false`.
Related: [[erp-crud-master-extensions]], [[erp-client-ui-browser-verification]].
