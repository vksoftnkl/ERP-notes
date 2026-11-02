---
name: print-assignments-screen
description: Menu 248 Print Assignments coded + registered 2026-09-29 (web parity) — files, data sources for company/branch/counter, the full-echo save, and the live test that creates/edits/removes a ZT- EMAIL row
metadata:
  type: project
---

Coded 2026-09-29, uncommitted; `fm.reg<PrintAssignments>(248)` in form_setting_menu.h. VKPOS already
had 248 all-true in user_menus, so no grant was needed.

Files (src/modules/admin/printing):
- `print_assignment_rules.*`: PURE (rungOf from the scope ids, never ptaSpecificity; buildMatrix
  filtered by branch/counter/OUTPUT MODE; the web does not filter by mode); Form/validate;
  templatesForScope; buildBody sends EVERY key, company null out loud, isActive on update only;
- `print_assignment_editor.*`: the popup, combos wired on `activated`;
- `print_assignments.{h,cpp,ui}`: a NexDialog with two tabs, By scope (the matrix plus "Effective here"
  = /resolve per purpose) and All assignments (paged at 20; scope / purpose / output / search filters).

Data sources:
- companies: `/master-lookups/name-id/all-masters?module=companies`;
- branches: `/master-lookups/branches/by-company/<id>`;
- counters: device **grid 31** (it has dev_branch_id; the name-id lookup lists every till with no branch);
- purposes: dd 48.

Endpoints: PrintAssignments Create/Get/Delete/remove(); list() now takes template, output, search,
page and globalOnly; the PrintScopeLookups namespace was added. `rungLabel()` was REMOVED.

Test: `tests/printing/test_print_assignments.cpp` (in run.sh), 25 pure + 15 live checks. It creates a
counter-VKPOS1 / SALE_ORDER / **EMAIL** row with remarks "ZT- …", edits copies, checks the
other fields survive, then soft-deletes it. It clears any leftover first. ux_pta_scope is partial
(WHERE NOT deleted), so re-runs never collide.
`PRINT_SNAP=<dir>` saves PNGs of the matrix and the editor.

Not built (web has them): printer-profile picking (no endpoint), a "used by" link from the Designer.
See [[document-print-built]], [[print-designer-port-analysis]].
