# Notes 136: Textile tags: please build the server side (2026-10-17)

**STATUS:** asked.

Textile shops buy a design ("Deepa Kanchi 402") once and never again, so they do not make an item
per design. Each purchase line gets a **tag**: an automatic barcode printed on a label, carrying
brand, size, colour, fabric, season, model, MRP and selling price. The counter scans the label and
gets everything back. Items stay generic ("Silk Saree", "Shirt").

The design is finished and every decision is accepted. **Nothing is built yet**, on either side.

## What to read

Everything is in `share/schema/stock/textile/`:

| file | read it for |
|---|---|
| **`plan-nestjs-textile-tags.md`** | **the build plan — this is the one to work from.** Tables, routes, logic, errors, tests, build order |
| `DECISIONS.md` | what was accepted and when (T1–T7, G1–G23, B1–B9) |
| `textile_tag_ui_mockup_rev3.png` | the screen your routes feed |
| `plan-textile-lot-tag.md`, `-gaps.md`, `-gaps-2.md` | background only; where they differ, the build plan wins (its opening table says where and why) |

## Three things to know before you start

1. **No stock table changes.** A tag is a new table in `inventory`. Its barcode is the lot's batch
   number; the engine only ever sees an ordinary batch.
2. **The tag knows its lot id before the stock exists.** Your engine builds a lot id from its
   identity (`lotIdentityUuid`, `stock-voucher-posting.helper.ts:1383`), so reserve computes the
   same id with the engine's own function (plan §5.1 step 6 has the exact SQL). That is why there
   is no link step and no nightly job, and why status is worked out on read (§5.6).
   **Please run test 4 (§12) first:** reserve a tag, post an opening with `batch = barcode`, and
   check `stock_lot.slt_id = ilt_lot_id`. If that ever fails, stop and tell me — the design rests on it.
3. **Price lives on the tag, cost stays on the lot.** Textile items use the new `TEXTILE_TAG`
   preset (batch only, issue `MANUAL`), so the lot keeps no MRP or selling price. Your `MANUAL`
   refusal (`STOCK_PICK_MANUAL_LOT_REQUIRED`) already stops a textile line sold without a scan.

## Order (plan §13)

| step | what | done when |
|---|---|---|
| **B-1** | the migration (§1: lists → markup → series → tag table, then the group flag, preset, settings, error wording) + module + reserve / get / update / void / lists save / setup save / suggest-price | tests 1–4 pass |
| **B-2** | grids + dropdowns (§11) + label print purpose and dataset (§8) + price / hold / unhold / print | tests 7, 8, 13 — **a shop can go live with OPENING here** |
| **B-3** | scan (§5.5) + `ITEM_MISMATCH` on opening/adjustment validate (§6.1) + `POLICY_LOCKED` in `StockTrackPolicyService` (§5.2) | tests 5, 6, 9, 10 |
| **B-4** | GST value slab (§7) — **independent, can run beside B-1** | test 12 |
| B-5 | textile stock report grid + re-lot hook | test 14 |
| B-6 | purchase hooks — only once a purchase module exists | — |

**B-4 is not textile-only.** Under GST 2.0 a garment's rate depends on its value per piece *after
discount* (≤ ₹2,500 → 5 %, above → 18 %). Today nothing holds that limit or the rule
(`statutory_limits` has no such code). A ₹2,799 shirt sold at 12 % off is ₹2,463 and must go out at
5 %. The plan adds `GST_VALUE_SLAB_APPAREL`, a small `public.tax_value_slab` table by HSN prefix, a
resolver route, and a check in sale-bill save/post that refuses a wrong slab. Every garment shop
needs this, tags or not, so it has to be live before textile billing.

## Things that touch your existing code

Each is small and in that module's own transaction:

| where | change | plan |
|---|---|---|
| `app.module.ts` | register `TextileTagsModule` | §2 |
| `inventory.item_group_master` | expand-only column `itg_textile_label_per_piece` | §1.5 |
| `StockTrackPolicyService.syncFromItem / syncFromItemGroup` | refuse a policy change on an item / group with OPEN tags | §5.2 |
| opening + adjustment **validate** | a batch equal to a tag's barcode must have the tag's item | §6.1 |
| sale bill save / post | slab check | §7 |
| promotions (where the server re-evaluates) | a brand condition matches the **tag's** brand for a tagged line | §6.2 |
| stock adjustment re-lot | take `newTagId`; void the old tag after posting | §6.3 |

No sales DTO changes: a tagged line sends `lotId` + `batchNo` exactly as a picked lot does today.

## Please also

- **Rights:** the stock and inventory routes check none today. The tag write routes should call
  `assertMenuRight` the way the GST provider routes do (§3). Menu, grid and dropdown ids: please
  assign them and tell me.
- **Error wording:** rows in `Error_Messages.json` for every code in §9, in plain English (e.g.
  `TEXTILE_TAG_NO_SERIES`: "This counter cannot print tags yet. Ask the admin to give it a tag code.").
- **Grid SQL:** no comments in it (the runner refuses them) and every optional token always sent.

## Not needed from you

- **No nightly job, no BullMQ queue, no cron.** Status is derived (§5.6).
- **No down-sync work for tags.** Sync is push-only today; tags follow whatever the platform does
  for item masters (B7). Just give every new table its `<prefix>_sync_date` column.
- **No change to the stock engine, `stock_lot` or the posting helpers** beyond calling
  `lotIdentityUuid`.
- **Purchase.** B-6 waits for the purchase module.

## One question for you

1. **Down-sync (B7).** An offline till cannot scan a tag printed at the back office until the cloud
   sends it down. Is there a plan for masters to sync down to tills (items, prices)? If yes, tags go
   on the same list; if no, textile shops need the till online to scan back-office tags.

## What I do on the Qt side

| after | I build |
|---|---|
| B-1 | New tags, Series and Markup tabs; size / colour / fabric / season lists with quick-add |
| B-2 | Register and Markdown tabs; label printing |
| B-3 | scan on the sale bill, DC, return and physical count (metre items ask the length; hold warning) |
| B-4 | the bill re-resolves the slab when a discount changes |

Please tell me when each step is on the box, with the menu / grid / dropdown ids.
