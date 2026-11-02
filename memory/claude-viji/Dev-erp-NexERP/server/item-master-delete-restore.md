---
name: item-master-delete-restore
description: Item master since notes 67/68 (2026-09-30) — no preset = follows group policy; DELETE only deletes (409 if deleted), POST /items/restore restores only children stamped >= the item's deletion instant; prices keyed on company+branch+unit; three item unique indexes partial
metadata:
  type: project
---

`ItemsMasterService.softDeleteComposite` / `restoreComposite` (one transaction each). DELETE on a
deleted item is 409 (it was a toggle). Restore picks children with `*_updated_on`/`*_modified_on >=
item_modified_on` (the deletion instant; children are toggled after the item in the same tx), so rows
an earlier sync removed stay deleted; it re-runs `syncFromItem`, delete runs `retireForItem`. The Qt
"Restore" action must call `POST /items/restore?item_id=` now.

Composite sync keys: prices `ipm_company_id+ipm_branch_id+ipm_uc_unit_id` (= uq_item_price_master_scope;
godown is an attribute), reorders `ir_branch_id+ir_unit_id+ir_godown_id`; omitted scope = null;
duplicate keys in one payload → 400. The unique indexes are partial on not-deleted and DB-only (no
@@unique in fragments); since notes 137 (20261021100000) they are `uq_item_name_en_company_ci` /
`uq_item_code_company_ci` / `uq_item_sku_company_ci`, `uq_ean_code_ci` (upper) and
`uq_ir_item_branch_unit_godown` — see [[item-master-notes-137]].
Prisma 6.19 P2002 on them reports `meta.target` as a COLUMN array (`['item_name_en']`); P2003 gives
`meta.constraint` — read both via `violatedConstraintOf`.

Stock track policy (notes 68, the user's rule, 2026-09-30): an item follows its GROUP unless it names
its own preset. `item_track_preset_id` is the only item-level choice (NONE included); no preset → no
ITEM row, and a derived one is retired (migration 20260930140000 retired CHILLI POWDER's 'M').
`deriveFromItem` is gone. The item flags still feed sales' negative-stock answer
(`saleLineAllowsNegativeStock`: godown AND company AND item) and bulk-load's `tracking_type` hint,
which can now disagree with the resolved policy — reported, not changed.
A GROUP's derived row is SHARED (stp_company_id NULL, branch NULL) since notes 69: it was filed under
`requestContextService.getCompanyId()`, which is the LOGIN TOKEN's company (access-token.guard.ts),
not the working company, so other companies' items never saw it. Migration 20260930150000 re-filed the
existing ones; stranded company rows are moved/retired on the group's next save. Never file anything
by the context company expecting it to be the working company.

**Why:** notes 67 (re-check of notes 50). **How to apply:** e2e proof in
`test/items-master-notes-67.e2e-spec.ts` (rolled back). See [[dto-hasownproperty-always-true]],
[[avoid-item-tax-master]].
