# Change Selling Price — Bulk: the screen plan

Menu 30. The first form to build (PRICING_AND_FORMS_PLAN.md, Phase 1): it
writes one table, fires no trigger, moves no stock. Everything referenced
here exists and is smoke-verified as of 2026-09-04.

References: `change_selling_ui_mockup.png` (the layout) ·
`16_stock.sql` §20 (`stock_mrp_price`, `fn_smp_effective`) ·
`16q` Q23–Q27 + the S1–S3 save statements · legacy 3.0 form of the same name.

---

## The three API endpoints

| Endpoint | Query | Purpose |
|---|---|---|
| `GET  /stock/price-bulk` | **Q25** | the grid: one row per (item × uom × live bucket) that has stock, plus the headline row for items with none. Filters: group, brand, section, supplier — all optional |
| `GET  /stock/price-buckets/:item` | **Q24** | the F12 picker: every live bucket of one item, both dimensions grouped |
| `POST /stock/price-bulk` | **Q26 → S1–S3 → Q27**, one transaction | validate, upsert the changed rows, report what carried no stock |

All three read prices only through `fn_smp_effective` — the screen never
joins `stock_mrp_price` to `item_price_master` itself.

## The grid

Columns per the mockup: `# · Item · Uom · Stock · MRP · Sale Px · Src ·
Cost · [Mkup% · Price] × 4 levels · Min`.

**Src** renders from Q25's two columns:

| price_source | price_scope | Src shows | Save does |
|---|---|---|---|
| BUCKET | BRANCH | `BUCKET·BR` | updates this branch's own row (S2 by `bucket_id`) |
| BUCKET | CHAIN | `BUCKET·CH` | depends on the scope switch — see below |
| MASTER | — | `MASTER` | creates the bucket (S3) — unless the row has *neither* dimension |

**The scope switch** (header, radio): `This branch ▸ / All branches`.
It decides what an edit targets, and rule 4 in 16q is the law:

- *This branch* + a CHAIN-sourced row → S1 finds no branch row → **S3 creates
  the branch override**. The chain row is untouched; other branches keep it.
- *All branches* + a CHAIN-sourced row → S1 finds the chain row → **S2
  updates it**. Every branch without its own override moves at once.
- *All branches* is an HQ-role action; a branch user's switch is fixed to
  *This branch*.

**Untracked items** (no MRP, no sale price — Src `MASTER` with both blank):
these rows edit the **headline**, and the save routes them to the existing
`item_price_master` endpoint, not to smp — `ck_smp_identity` refuses a
(NULL, NULL) bucket on purpose. The grid may still show and edit them; the
API just fans the payload two ways by "has a dimension?".

**Each level is four numbers** (mockup panel): `markup % ⇄ price-wot ⇄
price ⇄ margin %`. Type any one, the other three recompute live —
markup on the tax-INCLUSIVE pair, margin on the tax-EXCLUSIVE pair. Tax %
comes from the item's tax master at load. `smp_round_off` applies at the
price-with-tax step. No recalculate button; the delta chip against the
loaded value (green up / red down) IS the preview.

**Delta baseline:** loading, or picking a bucket in F12, resets the row's
before-values; the chip measures against those.

## The save, exactly

One POST, one DB transaction:

1. **Q26** over the changed rows → any `problem` rows with the
   above-MRP / below-min verdicts abort with the list; `below cost` rows
   consult `inventory.below_cost_price` (block → abort; ask → the API
   returns them once with `needs_confirm`, the client re-posts with
   `confirmed: true`; allow → proceed).
2. Per changed row: **S1** (find at target scope, `FOR UPDATE`) → **S2**
   update or **S3** insert. Collect the touched `smp_id`s.
3. **Q27** over those ids → the "priced, nothing on hand" list.
4. Commit. Response: `{saved: n, no_stock: [...], master_rows_saved: m}`.
5. Toast: “12 buckets saved · 2 have no stock on hand — the price applies
   when stock arrives.” Never plain “Saved” when `no_stock` is non-empty
   (legacy fault #2).

The DB is the last line anyway: `ck_smp_not_above_mrp`,
`ck_smp_prices_nonneg` and `ex_smp_overlap` abort a save the screen failed
to validate, inside the same transaction.

## Validation matrix — who enforces what

| Rule | Client (live) | API (Q26) | Schema |
|---|---|---|---|
| price ≤ MRP | red cell + Save disabled | abort | `ck_smp_not_above_mrp` |
| price ≥ min | red cell | abort | — (min is data, not a CHECK) |
| price vs cost | amber cell | block / ask / allow | — |
| one row per bucket per scope | n/a (S1 decides update-vs-insert) | find-then-write | `ex_smp_overlap` |
| no (NULL, NULL) bucket | route to headline save | route | `ck_smp_identity` |

## Keyboard, matching the legacy

`F5` load · `F7` clear grid · `F8` bulk-load filter dialog · `F12` bucket
list for the current row · `Ctrl+D` remove row · `Ctrl+Enter` save.

## Qt client notes (house rules)

- **Edits come from the delegate's `cellEdited`, never the table's
  `itemChanged`** — the grid-delegate pattern already used on sales and
  charges grids. The four-number recompute runs in the delegate commit path,
  writes siblings with signals blocked at the *model* level, and marks the
  row dirty once.
- Connect only **user-intent signals** (`activated`, `editingFinished`);
  programmatic loads must not fire recompute. If a `QSignalBlocker` seems
  needed, the wiring is wrong.
- The scope switch and filters are load-time inputs; changing scope with
  unsaved edits prompts (it changes what Save means).
- Row model keeps: loaded values (delta baseline), `bucket_id`,
  `price_scope`, tax %, cost, dirty flags per level.

## Out of scope for v1 (deliberately)

- Effective-date scheduling UI — the columns are live and dormant; this
  screen writes "now" rows only.
- Editing MRP / creating buckets for items with no policy — the tracking
  policy decides what buckets exist; this screen prices them.
- The purchase screen's price write — same S1–S3 statements, arrives with
  form 6.

