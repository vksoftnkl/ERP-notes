# Notes 78 — A new company gets its main branch (2026-10-02)

Follow-up to notes 72 (A1 seeded the fiscal year on company create). Since then, a company created
from the screen has books but **no branch**. Every document hangs off a branch: the login token
carries `branch_id`, and godowns, counters, number series and stock balances are all per branch. So
the company still can't be used until somebody adds a branch by hand in Branch master.

The branch service already assumes each company has **exactly one default branch**. `clearDefaultBranch`,
the delete guard (`assertDeletable`), the move guard (`assertMayChangeCompany`) and restore all
assume it. Nothing guarantees it.

## Live state (192.168.0.106 / ERP, read 2026-10-02)

| Company | Live branches | Default |
|---|---|---|
| LEAPSWITCH NETWORKS PRIVATE LIMITED | **0** | – |
| CHANDRA SAW MILL | 1 (saw mill) | **none** |
| HAP Solutions | 1 (hap chennai) | **none** |
| THE SCM SILK | 1 (scm branch) | **none** |
| Acme Foods Pvt Ltd | 3 | 1 |

## Asks

| # | Item | Ask |
|---|---|---|
| 1 | **Seed the main branch on company create.** | In `CompanyMasterService.createCompany()`, inside the same `$transaction` and right after the fiscal-year create, insert one branch: `brName 'Main Branch'`, `brType 'HEAD OFFICE'`, `brIsDefault true`, `brIsActive true`, `brCode` NULL (it is unique across all companies). State/state code, GSTIN, PAN, reg type, address, region address, tel/phone/mail are copied from the new company row. Audit-log it as "Seeded on company create" and return `brId` in the create payload. The TypeScript is at the bottom of `45_main_branch_seed.sql`. |
| 2 | **Seed its godown too (recommended).** | Without a godown the branch exists, but stock screens and the sale bill have nowhere to post. In the same transaction: `inventory.godown_locations` row `gdlName 'Main Godown'`, `gdlType 'WAREHOUSE'`, no parent, under the new branch, then set `brDefaultGodownId` to it. Your call if you'd rather keep godowns manual. If so, say so and the client will prompt after create instead. |
| 3 | **Backfill the box.** | **`45_main_branch_seed.sql` (share root).** It inserts a Main Branch for every live company with none, marks a lone branch default when nothing is, and lists (never guesses) a company with several branches and no default. Idempotent. Dry-ran on erp_dry with all four cases plus a re-run, rolled back. Live preview: 1 insert (LEAPSWITCH) and 3 marks (CHANDRA, HAP, SCM). |
| 4 | **One default per company, enforced.** | The same file adds `uq_branch_master_default` = UNIQUE `(br_comp_id) WHERE br_is_default AND NOT br_is_deleted`. Today nothing stops two defaults: `clearDefaultBranch` is the only guard, and it runs only on the service path. Your create/update already clear the old default before the write in the same transaction, so this index never trips there. Dry run confirmed it refuses a second default. |
| 5 | **Unticking the default leaves the company without one.** | `updateBranch` with `brIsDefault: false` on the current default (`applyOptionalFields` writes it straight through) leaves the company with no default branch. Refuse it with 400 ("make another branch the default instead"). Setting another branch as default is the only way to move the flag. |
| 6 | **Don't let the last branch go while the company lives.** | `assertDeletable` deliberately lets the company's only branch be deleted once it is unused, so that the company itself can be deleted (B2). That stays. But a company left live with 0 branches is the same broken state as item 1. Either keep it as it is (delete company right after), or refuse and say "delete the company instead". Your call. Tell us which and the client will word its message to match. |

## Client side

Nothing new is needed for items 1–4: the seeded row simply appears in the Branch list (grid 56) and
in the login branch combo. Once item 2 lands, the Company entry's "saved" message will name the
branch and godown it created.

## Side note (no ask)

`br_default_godown_id` has no FK to `inventory.godown_locations` (it is cross-schema). It is only
noted here, because the godown delete guard is the place that would have to cover it.
