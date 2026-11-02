# A2 — the shared posting services. One page, then the files.

**For Prathap. A1 is finished** (all SQL run on 192.168.0.106, `prisma db pull`
done, box cleared to masters-only on 2026-09-21). A2 is the layer every
document posts through. Nothing user-visible comes out of it, and everything
after it is wrong without it.

---

## Read in this order

| # | file | what for | how much |
|---|---|---|---|
| 1 | `README.md` → "Where logic lives" | **the rule**, and the offline rider that overrides it | 2 min |
| 2 | `offline-sync-invariants.md` | why six over-draw CHECKs are gone and what replaces them | 5 min |
| 3 | `notes.txt` (42) | the narrative: what changed on 21 Sept and why | 10 min |
| 4 | `plan-backend-sales.md` **§3** | **the build** — every service, method by method | the main read |
| 5 | `sales_posting_flow.md` | the spec: what each document writes, row by row | reference |
| 6 | `HANDOVER_endpoints.md` **§1, §9, §10** | the blocks the services return and the codes they raise | 15 min |
| 7 | the four `.sql` contracts (below) | the algorithms, beside the tables they touch | as you build each |

Skip `HANDOVER` §2–§7 for now — that is A3 onward. Read §1 though: `posting`,
`locks` and `warnings[]` are **built by these services**, not by the
controllers, so their shape is an A2 decision.

---

## What to build (`plan-backend-sales.md` §3)

| § | service | contract lives in |
|---|---|---|
| 3.1 | `StockPostingService` | `schema/stock/19_stock_posting.sql`, `20_stock_transfer.sql`, and the four flow docs in `schema/stock/` |
| 3.2 | `SalesPostingService` | `sales_posting_flow.md` §5 |
| 3.3 | `DocRegisterService` | flow §5.7, `gst/gsp_flow.md` §6a |
| 3.4 | `LoyaltyLedgerService` | `schema/13_loyalty_ledger.sql` + `14_loyalty_coupon.sql` |
| 3.5 | `PromotionUsageService` | `schema/15_promotion_scheme.sql` §6 |
| 3.6 | `StatutoryService` | `29_sales_documents.sql` §5 |
| 3.7 | `sales.guards.ts` | flow §9, `HANDOVER` §1.5 + §9 |
| 3.8 | `ChargeCarryService` | flow §7 |

Also migrate the existing stock-voucher module onto `StockPostingService` and
get its tests green — otherwise there are two stock engines again, which is the
thing this whole change was about.

---

## Nine rules that must survive. Each has already been got wrong once.

1. **No functions, no triggers in the database.** The till posts offline and
   pushes to the cloud; a trigger fires on the SERVER during that push, on rows
   written hours earlier, in a language nobody here debugs. `13`, `14`, `16`,
   `19`, `20` now declare two objects between them, both DDL partition helpers.

2. **One private method writes each ledger table**, and it recomputes before it
   returns. A trigger could not be forgotten; a service call can, and there are
   NINE write paths into `loyalty_ledger`. Assert it in a test: no file outside
   the service writes those tables. (§3.4b)

3. **`consume()` takes `SELECT … FOR UPDATE` on the wallet — never
   `SKIP LOCKED`.** Two tills redeeming one wallet must serialise; in a
   supermarket that is a real race. The lock must sit in the CALLER's
   interactive `$transaction`, or the bill's post and the redeem land in
   different transactions and it protects nothing.

4. **Recompute order:** lots before members (the wallet's `next_expiry_on`
   reads `lld_lot_balance`), vouchers before batches (`lcb_redeemed_count`
   counts vouchers by status). Both silently produce wrong numbers if reversed.

5. **Deterministic sweep ids** — `md5(company|date)` for points,
   `md5(company|coupon|date)` for vouchers, and they must not collide. A
   same-day re-run is then rejected by the unique index instead of writing a
   second set.

6. **Recompute, never increment.** A recompute is idempotent under replay and
   independent of arrival order. An incrementing cache double-counts the moment
   a sync batch is re-sent.

7. **Over-draws are RECORDED, not refused.** Two tills offline can each spend
   the last of one wallet; both bills are real. The refusal belongs in the
   service at post, while it can see the true balance — the database now only
   records it, and the partial indexes (`ix_*_overdrawn`) are the report.

8. **Derive the locks, never store them.** Lock 1 is `*_status='POSTED'`,
   lock 2 is `gde_status='GENERATED' OR gdw_status='GENERATED'`. A stored flag
   drifts from the thing it describes.

9. **`assertSalesmen()` replaces a foreign key.** `*_salesman_id` is `uuid[]`
   and PostgreSQL cannot FK array elements, so NOTHING in the database will
   catch a salesman id that does not exist, is deleted, or belongs to another
   company. Element 1 is the primary, by convention — `abl_salesman_id` takes
   `sb_salesman_id[1]`.

---

## Done when

* every service in §3 exists with the methods named there;
* the stock-voucher module posts through `StockPostingService`, tests green;
* the conformance test of `plan-backend-sales.md` §12 passes — all twenty
  crew/vehicle columns present across the four goods-moving documents;
* a test asserts no code outside `LoyaltyLedgerService` writes
  `loyalty_ledger`, `loyalty_coupon_txn` or `loyalty_coupon`;
* the four loyalty tests of `13`'s contract block: two concurrent redeems
  serialise; an over-large redeem throws and writes nothing; a redeem spanning
  three lots writes three rows in FIFO order; a sweep re-run on the same date
  writes nothing the second time.

Then A3 — the bill and order routes — and **tell me when those answer**, because
the Qt bill screen is built to `HANDOVER` §2 the same day and its three
pictures are already confirmed.

---

## Do not

* re-add an over-draw `CHECK` "just to be safe" — read
  `offline-sync-invariants.md` §1 first;
* write a `ttd_is_locked` or any other stored lock flag;
* let a report fall back to `= ANY(...)` on a salesman array — the GIN indexes
  need `@> ARRAY[:id]::uuid[]`;
* port the client-built voucher JSON, the ±0.50 balance tolerance, or approval
  as a posting gate (`plan-backend-sales.md` §14).
