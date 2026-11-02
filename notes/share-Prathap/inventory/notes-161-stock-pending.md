# notes-161 — stock section: what is still pending on the server

Date: 2026-10-27. From: Viji / Claude. To: Prathap.

The user asked for one list of everything still open in stock. Checked this morning against the live box (.106) and the server source snapshot. In order of priority.

## 1. Unit factor direction (notes-147 A / 148): the code is fixed, the DATA is not

`item-unit-conversion/README.md` now follows the item card's rule (Bottle (1) / PCS (10, base) → Bottle = 10). Thanks. But rows saved before the fix still carry the old inverted figure. On .106 today, three items have a row ABOVE the base with a to-base factor under 1:

| item | unit (slno 1) | `iuc_to_base_factor` now | should be |
|---|---|---|---|
| Samsung H1 | Bottle | 0.1 | 10 |
| Sakthi Manjal Powder | BOX | 0.1 | 10 |
| SUGAR | Kilogram | 0.02 | 50 |

(A row BELOW the base with a factor under 1 is correct and must stay.)

One posted stock voucher used a flipped factor: **OPN0006**, Samsung H1, `svi_to_base_factor 0.1`. It is posted, so its base qty and per-PCS cost are wrong in the ledger and the balances.

**Asked:**
1. Re-derive `iuc_to_base_factor` for every item with the new chain (one script, dry run first, list what changes).
2. OPN0006: cancel it and re-post it with factor 10, or tell us and the user will cancel + re-key it from the screen. **Please do this before anything else posts Samsung H1 in Bottles.**
3. Check the same items in sales lines (bill / DC / return) for the old factor. We found none in stock vouchers apart from OPN0006, but we did not check sales.

## 2. notes-160 — still pending, all three parts

- **A.** `freeAvailableQty` on pick-stock + the engine refusing free qty above the free balance. Until then Free stays locked on transfers.
- **B.** `POST /stock/transfer/recall` for an untouched Branch Outward. The Qt button is built and switched off (`kRecallLive = false`).
- **C.** Short device code in stock voucher numbers. A transfer at 09:28 today still got `TRF/2026-2027/44c01ccc-a162-8314-34b0-1dbf2f5c7bab/17`. The Qt screen now SHOWS the first 8 characters only, as a stop-gap. The stored number still needs `dev_code`.

## 3. notes-152 — Prev/Next walk order

It is in the source (`txn-adjacent.service.ts`). Please confirm it is in the running build on .106. We will re-test once you confirm.

## 4. Before the first year-end (not urgent this week, but please plan for them)

These were in the 09-26 engine-port plan (`schema/stock/plan-nestjs-stock-engine-port.md`) and are still absent:

1. **Valuation snapshot / period close.** A month-end freeze of qty, value, MRP/SP valuation and age days. Without it, the audited year-end figures cannot be reproduced later.
2. **Goods in transit in the valuation.** A despatched but not-yet-received branch transfer is in neither branch's balance, so as-on-date valuation leaves it out. Year-end needs a goods-in-transit line.
3. **Nightly rebuild / balance assertion job.** `/stock/admin/balance-assertion` and `rebuild-costs` exist, but nothing runs them on a schedule.

I will write the detailed design for each as its own note. This is just so you know they are coming.

## 5. Purchase module: absent

There is no purchase entry on the server (only suppliers). It is the normal way stock comes in, and the textile label plan (`schema/stock/textile/plan-textile-lot-tag.md`, accepted 09-27) depends on it. Nothing to build yet. The plan note will come separately, once the user sets the order.

## Not for the server

Stock reports (ledger, summary, ageing, valuation, transit loss) and the hidden Stock menus (Split Stock, Production, Gate Inward/Outward, Own Consumption…) are client work or user decisions. Nothing is needed from you for them yet.
