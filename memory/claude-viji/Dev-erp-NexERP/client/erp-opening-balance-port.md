---
name: erp-opening-balance-port
description: Opening Balance (menu 55) React port — the replace:true invariant, scope semantics, and two places the plan/route docs disagree with the server code
metadata:
  type: project
---

`features/accounts/opening-balance/` — the menu 55 screen, ported from the Qt
`accounts/opening` dialog on 2026-09-24. Route `/accounts/opening-balance`,
ui_tables 30/31, picker grid 107, `store/api/openingBalanceApi.ts`.

Three things that are not obvious from the code alone:

- **`POST /create` carries `replace:true`, so the grid must hold EVERY row that
  has an opening.** `/list` is asked with `includeZero=false` to guarantee that,
  and the picker only adds. A search box, a filter or server paging over
  `draft.rows` would make Save soft-delete every opening scrolled out of view.
  There is no `DELETE /delete` call anywhere in the feature on purpose: absence
  from the body IS the delete, and sending `0` only earns a `skippedZero`.
- **Two places the written contract is wrong and the server code is right.**
  (1) The `/bills` route description says a receipted bill takes "date,
  credit-day and narration changes", and the Qt delegate left Invoice date
  editable — but `bill-wise.service.ts` compares `ablDocDate` alongside amount,
  side and refno and refuses it. Grace days it does write, so those stay open.
  (2) `opRemarks` omitted on an UPDATE is passed through untouched, so "omit
  when empty" (as the plan words it) would silently restore a cleared remark;
  the builder sends an explicit `null` for a row that arrived with one.
- **`ux_op_scope` makes a NULL branch a scope of its own.** "All branches" is the
  desktop screen's wording for the COMPANY-LEVEL set (`op_branch_id IS NULL`),
  not a merge. `abl_branch_id` is NOT NULL, so a bill-wise party can only be
  opened under a branch — the breakup panel is shut under the company-level set.

**Why:** each of these is silent when wrong — the request succeeds and something
else is true afterwards.

**How to apply:** `opening-balance.integration.test.ts` proves all of it against
a live API (skipped without `OPENING_IT_*` env vars; it saves with `replace:true`
over the scope it is pointed at, so it collides with anyone who has menu 55 open
on the same company-year). Related: [[erp-ui-table-id-registry]],
[[erp-configured-grid-registry]], [[erp-ui-table-width-is-not-a-percent]],
[[erp-menu-href-mapping]], [[erp-auth-endpoint-verification]].
