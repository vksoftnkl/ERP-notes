---
name: company-branch-masters-audit
description: "2026-10-01 audit of Company (grid 52) + Branch (grid 56) masters — client fields added, grids re-registered, notes 72 (fiscal_years never seeded = new company unusable, aato class unsettable, no delete guards/restore)"
metadata:
  node_type: memory
  type: project
  originSessionId: 01ac2b4a-c3c7-4c7b-a203-afc8822d0080
  modified: 2026-10-01T05:22:00.301Z
---

Write-up for Prathap: `share/admin/notes-72-company-branch-masters.md`. The old grid JSON is saved beside it.

**Client changes made 2026-10-01:**
- **Company entry:** added compCode, support phone/email, region state/country, and a Books section (FY From/To + Books Begin, defaulted on create to the current 1 Apr–31 Mar). Validation: GSTIN state prefix must equal the State, GSTIN chars 3-12 must equal the PAN (a blank PAN is filled), and FY To must be after From. The lock date is deliberately NOT offered, because it belongs to fiscal_years.
- **Branch entry:** added brCode; the Company field is locked in edit mode; Default Godown is filtered `ibranch_id` = this branch (dd 26) and disabled on create. It has the same GSTIN/state/PAN checks. The dead beforeSubmit was removed.
- **Both lists:** Show Only Deleted works (branch list gained the checkbox). In the deleted view, Edit and Delete show an info message, because there is no restore route.

**Box changes (with the user's go-ahead, "you can also add"):** grids 52/56 are now `*_is_deleted = icomp_is_deleted` / `ibr_is_deleted`, with search flags set (56 had none).

**A1 seed SQL written 2026-10-01:** `share/43_fiscal_year_seed.sql` is an idempotent backfill (always April–March, anchored on books-begin, then fin-year-from, then today). It dry-ran on erp_dry, and the live preview is 1 row (LEAPSWITCH). The NestJS create-time seed is spelled out at the bottom of the file. No trigger, per [[sql-vs-nestjs-rule]]. Correction: the posting guards ALLOW a missing year; what breaks is the empty acc-year combo in MenuWindow.

**Server gaps (notes 72):**
- A1 nothing writes `fiscal_years`, so a new company has no year.
- A2 `comp_aato_class` (read by statutory.service) and A3 `comp_dc_purposes` are not in the DTO.
- B1 no restore route.
- B2 a company deletes with a live branch.
- B3 there is no branch delete guard.
- B4 a branch can be moved to another company.
- C4 many br_/comp_ "settings" columns are read by no service (rounding, prefixes, price fixing).

When A2 lands, add a compAatoClass combo on the Tax tab. When B1 lands, switch the lists to the /restore pattern like [[item-masters-audit]].

The DB role `claude` has no INSERT, so DB changes go to Prathap as SQL. See [[db-live-connection]], [[grid-runner-binding-rules]], [[law-params-per-company]].

**Notes 72 APPLIED by Prathap + re-verified 2026-10-01 (live + source):** all of A–C is in. E1 (April–March only) was FIXED and verified the same day, so **notes 72 is CLOSED**. The form's own 62-key body round-trips with no mismatch. B3/B4 are relaxed by design (an unused branch may move or be deleted).
**Client follow-up done the same day (full build clean, smoke 34/34):**
- Reg type is a staticComboCoded with REGULAR/COMPOSITION/UNREGISTERED/SEZ. The server returns upper case, so the old text combo would have blanked on edit.
- compAatoClass combo, defaulting to LE_5CR.
- compTdsApplicable checkbox (TCS is now its own label).
- compDcPurposes uses the CompanyDcPurposesWidget checklist (at least one required).
- FY fields are read-only on edit; on create, From must be 1 April and To the next 31 March.
- Both lists Restore via /company-masters/restore and /branch-masters/restore.
