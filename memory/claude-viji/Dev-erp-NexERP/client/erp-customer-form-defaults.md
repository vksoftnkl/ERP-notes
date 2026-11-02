---
name: erp-customer-form-defaults
description: Master form templates are the masters.*_form_defaults app settings — JSON shape (written by the POS), the session-scope read/write rule, and the shared Save-as-Default-Template module
metadata: 
  node_type: memory
  type: project
  originSessionId: c1c46a39-7ed5-4841-9849-9d11ef266a91
  modified: 2026-08-31T07:27:27.870Z
---

Since 2026-08-31 the **Customer Template** screen (`features/masters/settings/customer-template`)
and the customer master's create-form defaults both live in the
**`masters.customer_form_defaults` app setting**, not `/configs?configId=1` (that path is gone
from the client; the old `customer_template` config row is stale and unread).

- Def seeded by server migration `20260829061000_seed_masters_form_defaults`: TEXT,
  `asd_max_scope = BRANCH` (also seeds `masters.item_form_defaults`, still unused by the client).
- **The JSON shape is the POS's, not ours** — the dev DB already held COMPANY + BRANCH rows
  written by VKPOS. Keys are the customer form's own `cus*` names, values TYPED
  (booleans/numbers, `cusCollectionDays` an array), each id carrying a `*Name` companion
  (`cusAreaId`+`cusAreaName`, also Group/State/PriceLevel/Company). No company key in the POS
  rows — it takes company from the scope. Two gotchas: `cusGstType` is stored as the LABEL
  ("Unregistered") while the form's option value is the enum, so the parser upper-cases it; and
  a save merges over the stored JSON so keys this screen does not render survive.
- **Read and write must name the same layer or the screen lies**: read `/effective` with the
  session's ids, write at the DEEPEST layer the session names (branch → company → global).
  A GLOBAL write would vanish under the existing COMPANY/BRANCH rows.
- **Two writers.** The Customer Template settings screen (one form, saves at the session's
  deepest layer, merges over the stored JSON), and **"Save as Default Template"** in the
  Customer Entry modal footer (2026-08-31): SUPER-ADMIN-gated, three-way scope prompt
  (All Branches = COMPANY / This Branch Only = BRANCH / Cancel), builds the document from the
  draft alone. It writes ONLY on click — no effect, no autosave — and never runs the customer
  save (nor that this); its prompt takes Escape and the save chords in the CAPTURE phase, or
  they reach the entry modal underneath and close/submit it.
- Plumbing: `features/masters/shared/form-defaults-setting.ts` (session scope hook, override
  builder), parser `parseCustomerFormDefaults` in `customer-dropdowns.ts` (+ unit test), both
  screens go through `store/api/appSettingsApi` (a lazy trigger would break — see
  [[erp-quotation-entry-port]]).

**"All Branches" was shadowed by the branch row (fixed 2026-09-01).** The button wrote the
COMPANY override, toasted success, and the create popup went on showing the old values — the
BRANCH override still won `/effective`, which reads exactly like a save that never reached the
server. The shared prompt now looks up the session's own BRANCH override
(`findSessionBranchOverride` in `form-defaults-setting.ts`), warns about it, and offers a
ticked-by-default "Remove this branch's own template" that DELETEs it after the COMPANY write
succeeds (never before — a failed save would otherwise leave the branch with nothing). Unticked,
the save still happens and the toast is a WARNING saying this branch is unchanged. Customer and
Item both get it; the settings screens write at the deepest layer so they never had the problem.

**Prefill trap (both template screens):** the session scope hydrates in STAGES — company
first, branch once the header resolves it — so the first `/effective` read to land can be from
a shallower layer than the session ends up in, and a prefill-once screen then shows the company
row where a branch row exists. Fix in place: re-arm the once-per-open prefill whenever the
scope arg changes (effect declared before the prefill effect so it wins the same commit).

**Item Master has the same button** (2026-08-31), on `masters.item_form_defaults`. The whole
module is shared: `masters/shared/build-form-defaults.ts` + `apply-form-defaults.ts` (pure, spec
driven) and `masters/shared/form-template/*` (button, scope prompt, save hook); each master
supplies `template/excluded.ts` + `template/field-specs.ts` only. Item traps: exclude the
linked-row editors by PREFIX (`ipm_ iuc_ ir_ ean_`) plus the four `*_rows_json` blobs, and only
the eight LAZY dropdowns need a stored `*_name` — branch/base-unit/HSN load eagerly and resolve
their own label. The Settings → Item Template screen was rebuilt on the same setting (2026-08-31): it mirrors
Item Entry's fields, and `/configs?configId=2` (`item_template`, 3 picks) is now unused by
anything. All four screens verified end to end in the browser (prefill, save → 201 upsert,
create-form seeding).
Related: [[erp-app-settings-contract]], [[erp-client-ui-browser-verification]].
