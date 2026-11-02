# Notes 140 — Prices from a future date ("rate from tomorrow"), branch-wise (2026-10-22)

**User's ask:** a client wants to set a new price today that starts tomorrow,
for one branch or for all branches. Today there is no way to do that.

**Half of it is already built:**
- `ex_ipm_overlap` lets one item / unit / company / branch / bucket have several
  rows, as long as their `ipm_effective_from … ipm_effective_to` ranges don't
  overlap.
- Billing already picks the row in force on the document date:
  `item-price.lookup.ts` → `resolveEffectivePrice(unitRows, key, docDate, caller)`,
  which runs `isInForce` in `price-resolver.ts`.
- Change Selling Price reads and updates only today's row
  (`price-bucket.gateway.ts`: `CURRENT_DATE BETWEEN ipm_effective_from AND ipm_effective_to`).

So **no scheduled job and no DB function are needed**. Once a future row
exists, bills dated on or after its from-date pick it up by themselves.

What's missing is a way to write such a row, and the item card's save
tolerating one. Live check today: all 54 live price rows have the default
dates (1900-01-01 … 9999-12-31).

---

## A. Change Selling Price save takes a from-date

`POST /stock/price-bulk` gets an optional body field **`effectiveFrom`**
(date, `YYYY-MM-DD`). Absent or today = today's behaviour, unchanged.

For each changed row, with D = effectiveFrom, under the row's own key
(company, branch from `scope`, item, unit, key_mrp, key_sp):

1. Find the row in force **on D** (not on CURRENT_DATE), `FOR UPDATE`.
2. If its `ipm_effective_from = D`, update it in place. This covers a second
   edit of the same scheduled change.
3. Otherwise **split** it:
   - the old row gets `ipm_effective_to = D − 1`;
   - a new row is inserted with the new prices, `ipm_effective_from = D` and
     `ipm_effective_to` = the old row's `ipm_effective_to`.

   The new row copies every other column from the old row (cost, min,
   round-off, godown, the other levels' prices).
4. If there's no row on D for that key (a new bucket or branch override),
   insert one from D to 9999-12-31. Before D the item keeps its wider price,
   the chain or headline row, exactly as today.

Do both writes in one transaction. `ex_ipm_overlap` is `DEFERRABLE`, so set it
`DEFERRED` for the split, or shorten the old row before inserting.

**Refusals:**
- `effectiveFrom` before today → 400 `PRICE_FROM_IN_PAST`. Bills already made
  at the old price must not change meaning.
- More than a year ahead → 400 `PRICE_FROM_TOO_FAR`. Optional; it's a guard
  against a typo in the year.

The below-cost and above-MRP checks, and `needsConfirm`, judge the row as it
will stand on D.

The response gets `effectiveFrom` back, and per row `scheduled: true` when a
new future row was made.

## B. List and cancel scheduled changes

- **`GET /stock/price-schedule?companyId=&branchId=&itemId=`** (itemId
  optional): every live row with `ipm_effective_from > CURRENT_DATE`, in the
  caller's scope.
  - Same row shape as `price-bulk`, plus `effectiveFrom`, `effectiveTo`, and
    the prices of the row it replaces (`previous.levels[]`), so the screen can
    show "110 → 118 from 23-10".
- **`POST /stock/price-schedule/cancel {ipmId}`**: only a row whose from-date
  is still in the future.
  - Soft-delete it, then give the row just before it (same key, `to = from − 1`)
    the cancelled row's `ipm_effective_to` back.
  - Refuse a row already in force with 409 `PRICE_SCHEDULE_STARTED`.
  - Both writes are audited like a price save.

## C. Item save must leave future rows alone (otherwise A breaks the item card)

`ItemMasterUpdateService.syncPrices` matches payload rows to stored rows by
(company, branch, unit, bucket), with **no date**:
- A future row shares its key with today's row, so `existingByKey` keeps only
  one of them.
- `refuseDuplicateKeys` refuses a save that sends both back.
- If the card sent only today's row, the future row would be soft-deleted as
  "unclaimed".

**Ask:**
- `/items/get` → `prices[]` = only the rows in force **today**.
- Add a separate read-only `scheduled_prices[]` for the rows with
  `from > today`.
- `syncPrices` works only on rows in force today. It never matches, updates or
  soft-deletes a row with `ipm_effective_from > CURRENT_DATE`.
- A row the card adds starts today (the column defaults stay as they are).

Also, `findByItemId` (items-price-master.service.ts ~l.206) has no date filter
today. Past rows closed by a split (`to < today`) must not come back on the
card either. They're history: either filter them out, or return them under a
separate key if you want a price-history view later.

## D. The bucket is re-derived on every item save (FYI, handled on the Qt side)

`PriceBucketService.deriveBuckets` re-keys a stored row from what it holds now:
- MRP-tracked → `ipm_bucket_mrp = ipm_max_price`;
- sale-price-tracked → `ipm_bucket_sp` = the price at `sales.default_price_level`.

So editing the MRP of an "MRP 300 stock" row on the item card silently moved
that price onto other stock.

**Done on the Qt side (2026-10-22):** on a bucket row, the item card now locks
the MRP cell (MRP buckets) and the default level's price / profit % (sale-price
buckets), with a tooltip saying why.

**Server ask (small):** refuse it as well. On an **update by `ipm_id`**, if the
re-derived bucket differs from the stored one, return 400
`PRICE_BUCKET_CHANGED` naming the row, instead of re-keying. An older client
or a direct API call can otherwise still do it. A new row is free to pick its
bucket.

---

## Qt side once A–C are live (mine)

- **Change Selling Price:**
  - Add a "Price from" date next to the This branch / All branches switch,
    defaulting to today, and send `effectiveFrom`.
  - Add a "Scheduled" button that lists B with a Cancel per row.
- **Item card:** add a small "₹118 from 23-10" mark beside a level that has a
  scheduled change, read-only, from `scheduled_prices[]`.

**Branch-wise needs nothing new.** `scope: BRANCH` vs `CHAIN` already decides
whether the row is that branch's override or the price for every branch. The
from-date applies to whichever is chosen.
