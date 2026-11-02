---
name: textile-tags-notes-136-status
description: Notes 136 (textile tags) — the build plan folder share/schema/stock/textile is NOT on the dev box; the lot-identity premise (test 4) was verified 2026-10-17; free menu/grid/dropdown ids
metadata:
  node_type: memory
  type: project
  originSessionId: 81104474-26c6-4d21-bb34-afa619677ddf
  modified: 2026-10-17T11:54:26.703Z
---

**PARKED 2026-10-23** by the textile batch plan ([[textile-auto-batch-no]]), not cancelled.

Notes 136 (2026-10-17) asks for the textile tag server side (B-1..B-5) built from
`share/schema/stock/textile/plan-nestjs-textile-tags.md` + `DECISIONS.md`. That folder exists
nowhere on this machine (nor do the earlier till design docs TILL_DESIGN / plan-till-counter-claim):
the user's design docs live off-box and arrive by paste or copy. Nothing of notes 136 is built.

**Premise verified (plan §5.1 step 6 / test 4):** on 2026-10-17 an opening posted for the
batch-only item ZZ TEST B (BATCH) (`01a08501-ae40-747c-b09f-f329301357f4`, e2e company) with
`batchNo = barcode` minted `slt_id` = `svi_lot_id` = the uuid v5 computed up front with the
engine's `lotIdentityUuid` formula (`stock-voucher-posting.helper.ts`, namespace
`2f0b7c1e-5d3a-4e8f-9b61-7c4d2a9e0f53`) over
`company|item|upper(btrim(barcode))|-1.000000|-1.000000|0001-01-01|~|00000000-0000-0000-0000-000000000000`.
Reserve MUST fold the barcode with upper(btrim()) exactly as `key_batch` does. The throwaway spec
is in the session scratchpad only; the stock tables on the dev box were empty before and after.

**Why:** the whole design (no link step, no nightly job, status derived on read) rests on that id
being computable before the stock exists.

**How to apply:** ask the user to copy/paste the plan before building anything of notes 136; do not
invent table / route / error names (the Qt side builds from the same plan). Free ids on
2026-10-17 (STALE — see [[textile-auto-batch-no]] for 2026-10-23): menu_master max 289 (till block 271-289; Inventory root 3, Stock root 4, item masters
under 34), grid_details max 174, dropdown_details max 69, print_purpose sort max 210
(`ck_ppo_src_module` has INVENTORY and STOCK). `statutory_limits` has no GST value-slab code yet.
Cloud sync is push-only (local → cloud); nothing in the repo plans a down-sync (B7).
See [[stock-engine-ddl-lives-outside-repo]], [[cloud-sync-is-local-only]], [[grid-sql-no-comments]].
