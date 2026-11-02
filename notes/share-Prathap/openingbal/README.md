# openingbal — everything about opening balances

`accounts.acc_opening_balance` is deployed, correct and **empty**, and has no
route of any kind. Neither does the OPENING side of `accounts.acc_bill_balance`.
So a company cannot be opened on the system today. These files are what is
needed to change that.

**Read in this order:**

| file | what it is |
|---|---|
| `opening_balance_tables.png` | **start here** — which table, which column, in what order, as one diagram |
| `opening_balance_flow.md` | the prose spec: every table, column, default, generated column and CHECK, read off the live DB 2026-09-12, with the three flows written column by column |
| `plan-backend-opening-balance.md` | **what Prathap builds** — rewritten 2026-09-14: §2 is the schema DDL to run first, then six NestJS endpoints, the carry-forward transaction step by step, 11 decisions (§11), and a "do not build" list (§12) |
| `opening_balance_ui_mockup.png` | the Qt screen (menu 55), regenerated 2026-09-14 against the rewritten plan — ONE page: scope from the session plus a single This-branch / All-branches picker, the trial-balance strip, the ledger grid, and the bill-wise breakup of the selected party as a detail panel below it. Shows the MANUAL flip on edit, the frozen receipted bill, and the plug via the OPENING_DIFFERENCE role (`.py` beside it regenerates it) |
| `schema_gaps_2026-09-14.md` | the audit behind the plan's §2 — why the table needs FKs and a tenant id, why the plug/P&L ledgers are posting roles, why `abl_branch_id` NOT NULL forces branch-scoped bills, why `abl_src_*` / `abl_parent_*` must be set. Reference, not a to-do list |
| `plan-qt-opening-balance.md` | **the Qt client plan — REVISED 2026-09-15** — and §0 carries the HOUSE RULES for all five accounts screens (own `.ui` on `NexDialog`, no `NexCrud*`, delegate `cellEdited`, Ctrl+Enter, scope captured once) — plus four learned the hard way while building it: width and visibility belong to `ui_tables`, every `ApiClient` callback needs a `QPointer`, never gate on `ctx.can*` directly, and the ledger grid loads the openings that EXIST and PICKS the rest |
| `client_findings_2026-09-15.md` | **what the Qt client found when it was wired to the live routes** — one field to add (`opRemarks` is written but never returned by `/list`), one default to make safe (an absent `includeZero` answers with an empty chart), one rule to document (`replace:true` still needs each bill's `ablId`), plus the list of what was verified working so nobody re-tests it |

**The one thing that blocks everything:** there are two homes for an opening
balance, and the wrong one has the data —
`acc_ledger_master.led_ob_amount` / `led_ob_type` / `led_ob_as_on` (one figure
per ledger, **no year**, 2 of 44 non-zero) versus `accounts.acc_opening_balance`
(per year, with provenance, 0 rows). That is DECISION 1 in both documents.

Nothing in these files is assumed. Where the schema does not decide something it
is marked as a decision and left open.
