# Loyalty Status — backend plan (NestJS)

**Date:** 2026-10-05
**For:** Prathap
**Screen:** Sales › Loyalty Schemes › **Loyalty Status** — menu **79** (see §2).
**Mockups (the visual spec):** this folder.
- `loyalty_status_ui_mockup_members.png` — tab 1, Members
- `loyalty_status_ui_mockup_expiring.png` — tab 2, Expiring soon
- `loyalty_status_ui_mockup_schemes.png` — tab 3, Scheme summary
- `loyalty_status_ui_mockup.py` renders all three.

**Status:** plan only. Nothing is built on either side.
**3.0 reference:** `pos/Sales/loyaltystatus.*` (the FCondition filter frame) and grid 570 in the `3.0` database.

---

## 1. What this is

A read-only report over the customer loyalty wallets. It answers four questions:

1. Who holds how many points, and what are they worth?
2. Who can collect a gift today?
3. Whose points lapse soon?
4. How is each scheme doing?

It reads three things:
- `sales.loyalty_member`, the per-customer roll-up;
- `sales.loyalty_ledger`, the per-event rows and the FIFO lots;
- `sales.loyalty_scheme_gift`.

It writes nothing. The two actions on the screen (Suspend, Adjust points) are separate write routes, covered in §7. **SMS is out of scope** (§7.3).

**Customer loyalty only.** Agent loyalty (3.0's `agent_loyalty_*`, menus 205–207 / 212 under 202 "Agents") is a different thing and is not touched.

---

## 2. Menu — 79, not 206

There are two menu rows called "Loyalty Status":

| menu | parent | visible | what it is |
|---|---|---|---|
| **79** | 75 "Loyalty Schemes" | no | customer loyalty. Our scheme master is its sibling, **67** "Loyalty Programs" (`form_sales_menu.h:33`) |
| 206 | 202 **"Agents"** | yes | 3.0's agent loyalty report |

**Ask:** set `menu_visiblity = true` on **79** and grant it. Leave 206 alone.

The mockups' subtitle still says "menu 206". Read that as 79. The mockups will be re-rendered with the plan's decisions.

---

## 3. Routes are NEW and SELF-CONTAINED

Every route below lives under **`/api/v1/reports/loyalty-status/`**, in a new module `src/modules/reports/loyalty-status/`. It is shaped like `reports/ledger-statement/` (controller + service + dto + types + README).

- No existing route is called.
- The grid runner (`/configured-grid-sql/run`) and the dropdown runner are not used.
- `/promotion-loyalty-points/*` is not used.
- Prisma models and the shared decorators are fine to use.

**All logic lives in NestJS. No database function, trigger or view is added for this report** (user, 2026-10-05: "dont keep any function in db write in nestjs only"). Every definition in §4, and every query in §5, is SQL issued by the service or TypeScript. The only database change is one index (§5.6).

**The one exception is the DEFINITIONS (§4).** "Redeemable", "open lot" and the FIFO order already live in `LoyaltyLedgerService` (`src/modules/sales/posting/loyalty-ledger.service.ts`):
- `lots()` at `:292`
- `redeemable()` at `:342`

The report must agree with the till to the point. So either:
- (a) inject `LoyaltyLedgerService` and call those two reads, or
- (b) copy their WHERE clauses into the report SQL, with a comment pointing back to them.

Recommendation: **(a) for the single-member card (§5.4), and (b) for the list routes.** The list routes need set-based SQL over thousands of members, and those methods take one member at a time. Your call: **D1**.

---

## 4. The one set of definitions

Every number on the three tabs comes from this table. Here "today" is the server date in the company's time zone.

| name | definition |
|---|---|
| **Balance** | `lmb_balance_points`, the GENERATED column: earned + adjusted − redeemed − expired − gift. |
| **Open lot** | a ledger row with `lld_txn_type = 'EARN'`, `lld_is_deleted = false` and `lld_lot_balance > 0`. *(This follows `lots()`. Read §6 F1 first.)* |
| **Redeemable** | Σ `lld_lot_balance` of open lots where `lld_active_from` is NULL or ≤ today, and `lld_expires_on` is NULL or ≥ today. Exactly `redeemable()` `:342`. |
| **Cooling** | Σ `lld_lot_balance` of open lots where `lld_active_from` > today. |
| **Value ₹** | points × `lsc_redeem_value_per_point` of the member's scheme (`lmb_lsc_id`). It is NULL when the scheme has no redeem value (`lsc_allow_point_redeem = false` or the rate is NULL), and the client then shows "—". |
| **Next expiry** | `lmb_next_expiry_on`. The points due that day = Σ `lld_lot_balance` of open lots with that `lld_expires_on`. |
| **Best gift** | the `loyalty_scheme_gift` row of the member's scheme with `lsg_is_active`, not deleted, today inside `lsg_valid_from`..`lsg_valid_upto` (NULL = open), and `lsg_redeem_points ≤ redeemable`. ORDER BY `lsg_redeem_points DESC, lsg_slno` LIMIT 1. This is grid 570's rule, but on **redeemable**, not on the balance. |
| **Gift qty** | when `lsg_repeat`, floor(redeemable / `lsg_redeem_points`), capped at `lsg_max_qty_per_bill` when that is set; otherwise 1. |
| **Eligible** | Best gift is not NULL **and** `lmb_status = 'ACTIVE'`. |
| **Earned in period** | Σ `lld_points` where `lld_txn_type IN ('EARN','OPENING')` and `lld_txn_date` falls in [from, to]. Reversal rows of those types are included, so a returned bill nets out. Same filter as `recomputeMembers()` `:248`. |
| **Outstanding (per scheme)** | Σ `lld_points` of every non-deleted row with that `lld_lsc_id`, all-time, as on today. |
| **Holders (per scheme)** | count of members whose outstanding on that scheme is > 0. A member can hold points on two schemes, so this column **does not total** (the mockup's footnote says so). |

---

## 5. The routes

Every route takes the house scope: **`companyId`** (required) and **`branchId`** (optional; absent = all branches). The meaning of `branchId` differs by tab, see **D3**.

All list routes take:
- `page` and `limit` (limit ≤ 200);
- `sort`, a column key from the route's list, and `order` (`asc` / `desc`);
- `search`, an ILIKE over card no, mobile and customer name.

A list route returns `{ items, total, summary }`. `summary` is computed over the **whole filtered set**, never over the page, because the tiles and the Total row read it.

### 5.1 `GET /reports/loyalty-status/members` — tab 1 grid + tiles

**Query:**

| param | type | default | 3.0 FCondition equivalent |
|---|---|---|---|
| `lscId` | uuid? | all | Scheme Name |
| `status` | `ACTIVE`/`SUSPENDED`/`CLOSED`/`MERGED`? | ACTIVE | — |
| `balanceGtZero` | bool | true | — |
| `eligibleOnly` | bool | false | "Show eligible customers" |
| `pointsMin`, `pointsMax` | int? | — | From / To Points (on **balance**) |
| `includeMergedClosed` | bool | false | — |
| `earnedFrom`, `earnedTo` | date? | — | "Date From" checkbox. When absent, Earned is the lifetime `lmb_earned_points` |

**Item:**
```
memberId, custId, cardNo, customerName, mobile, lscId, schemeName, status,
earned, redeemed, expired, gift, adjusted, balance, redeemable, cooling,
value, nextExpiryOn, nextExpiryPoints, bestGift { lsgId, itemId, name, points, qty } | null,
lastActivityOn, branchId, branchName
```

`mobile` = `lmb_mobile`, falling back to `cus_phone1`.

The mockup's **Redeemed** column is `redeemed + gift`, because both spent points. Send them separately and let the client add them.

**summary:**
```
members, active, suspended, outstanding, outstandingValue,
earnedInPeriod, earnedBills, redeemedInPeriod, redeemedValue,
giftEligible, giftsLowOnStock, expiring30Points, expiring30Members
```
`expiring30*` uses a 30-day window, the default of tab 2. The tile only links to tab 2.

**Main query shape.** `loyalty_member` drives it:
- JOIN `customers`, `loyalty_scheme` and `branch_master`.
- `LEFT JOIN LATERAL` one aggregate over the member's open lots, giving redeemable, cooling and next-expiry points. `ix_lld_open_lots` covers this exactly: `(lld_member_id, lld_expires_on, lld_txn_date, lld_id) INCLUDE (… lld_lot_balance)`, partial on `lot_balance > 0`.
- `LEFT JOIN LATERAL` the best gift.
- `LEFT JOIN LATERAL` the earned-in-period sum, only when the dates are sent. `ix_lld_member (lld_member_id, lld_txn_date)` covers it.

`eligibleOnly` and the points range filter on the computed columns. So wrap the query, or repeat the expressions in WHERE. A top-level CTE is fine here, because this is NOT the grid runner.

### 5.2 `GET /reports/loyalty-status/statement` — tab 1 bottom left

**Query:** `memberId`, `from`, `to`, `showReversals` (default true).

**Returns:**
```
opening   // Σ lld_points before `from`
rows[] { lldId, accYear, txnDate, txnTime, txnType, srcDocType, srcDocId, srcAccYear,
         srcDocRefno, lscId, schemeName, reason, baseAmount, points, runningBalance,
         expiresOn, activeFrom, isReversal, reversalOfId }
closing
```

- Order by `lld_txn_date, lld_txn_time, lld_row_no, lld_id`. Use `ix_lld_member`.
- **reason** is built on the server. Use `lld_remarks` when present. Otherwise build it from type + scheme (+ "tender ₹x" when a REDEEM has `lld_money_value`, + "approved by <user name>" for ADJUST).
- `showReversals = false` hides **both** halves of a reversed pair: the row and its `lld_reversal_of_id` partner. This is the same pair rule as the ledger statement, so the running balance stays true.
- The client opens the source document from `srcDocType + srcDocId + srcAccYear`. Send the refno too, for display.

### 5.3 `GET /reports/loyalty-status/member` — tab 1 bottom right

**Query:** `memberId`.

**Returns:**
- the card: card no, mobile, enrolled on, lifetime bill amount and count, last redeem, status, block reason;
- `balance`, `redeemable`, `cooling` (with the earliest `activeFrom`);
- `lots[]`, in `lots()` order: lot id, earned on, source refno, points, used (`lld_consumed_points`), left, expires on, active from;
- `bestGift`, and `tenderValue` = redeemable × rate;
- the scheme's redeem rules (`lsc_min_redeem_points`, `lsc_redeem_multiple`, `lsc_max_redeem_perc`), which the card quotes;
- **`unlotted` = balance − Σ lots.left.** This is the gap F1 creates. The client shows it only when it is ≠ 0.

### 5.4 `GET /reports/loyalty-status/expiring` — tab 2 grid + tiles

**Query:**

| param | default |
|---|---|
| `lscId` | all |
| `withinDays` | 30, range 1–365 |
| `hasMobile` | true |
| `activeOnly` | true |
| `search` | — |

**Grain:** one row per **(member, `lld_expires_on`)**, summing the open lots that lapse that day, where `lld_expires_on` is between today and today + `withinDays`. `ix_lld_expiry (lld_comp_id, lld_expires_on)` covers it.

**Item:**
```
memberId, custId, customerName, cardNo, mobile, lscId, schemeName, status,
expiresOn, daysLeft, points, value, balance, balanceAfter, lastActivityOn
```

**summary:**
```
buckets: [{ label: '≤7', points, members }, { '8-15' }, { '16-30' }]
// the cut points scale with withinDays: ≤7 / 8–15 / 16–withinDays
valueAtRisk, members
```

A member who appears on two dates counts once in `members`.

No `lastSmsOn` column yet: see §7.3.

### 5.5 `GET /reports/loyalty-status/expiring/calendar` — tab 2 chart

**Query:** `lscId?`, `days` (default 90).

**Returns:** `weeks[] { weekStart, points, members }`. Weeks start on Monday.

### 5.6 `GET /reports/loyalty-status/schemes` — tab 3 grid

**Query:** `lscId?`, `from`, `to`, `includeClosedHolding` (default true), `splitBy` = `scheme` | `scheme_branch` | `scheme_month`.

**Item** (one per scheme, or per scheme × branch / month):
```
lscId, schemeName, status, startDate, endDate, pointsValidTill,
holders, bills,
opening            // Σ lld_points before `from`
earned, redeemed, gift, expired, adjusted   // movement in [from, to], by txn type
outstanding        // opening + movement — equals §4 when to = today
value, usedPct     // (redeemed + gift) / earned; null when earned = 0
```

**summary:** column totals, with `holders` left out (§4).

- `bills` = COUNT DISTINCT `lld_src_doc_id` on EARN rows in the period.
- **`pointsValidTill`** for a closed scheme = MAX `lld_expires_on` of its open lots. It is NULL when a lot never expires.
- `includeClosedHolding` keeps a CLOSED / expired scheme only while its outstanding is > 0.

**Index ask (the only one).** This route groups by `lld_lsc_id` over a date range. `ix_lld_scheme (lld_lsc_id)` has no date and no INCLUDE. Add:
```sql
CREATE INDEX ix_lld_scheme_period ON ONLY sales.loyalty_ledger
  (lld_comp_id, lld_lsc_id, lld_txn_date)
  INCLUDE (lld_txn_type, lld_points, lld_branch_id, lld_src_doc_id)
  WHERE lld_is_deleted = false;
```
It must go in as a Prisma migration, because the live database is Prisma-managed.

### 5.7 `GET /reports/loyalty-status/schemes/monthly` — tab 3 bottom left

**Query:** `lscId`, `from`, `to`.

**Returns:** `months[] { month, earned, redeemedPlusGift, expired }`. Months with no movement are still returned, with zeros. The last month carries `partial: true` when `to` falls inside it.

### 5.8 `GET /reports/loyalty-status/schemes/gifts` — tab 3 bottom right

**Query:** `lscId`.

**Returns:**
```
gifts[] { lsgId, itemId, itemName, points, repeat, maxQtyPerBill, stockCheck,
          eligibleMembers, issuedInPeriod, inStock | null }
```

- **eligibleMembers** = active members of that scheme whose redeemable is ≥ the gift's points. Count against the gift's own points, not the best gift: a member eligible for the mixer is also eligible for the flask.
- **issuedInPeriod** comes from `sales.loyalty_gift_redeem_item` for the gift item, in the period. It needs `from` / `to`, so add them to the query.
- **inStock** is the gift item's current stock in `branchId` (or all branches). Read it the way the sale bill reads stock: **D5**. It is NULL when `lsg_stock_check = false`.

### 5.9 `GET /reports/loyalty-status/export`

**Query:** `tab` = `members` | `expiring` | `schemes` | `statement`, the same filters as that tab, and `format` = `pdf` | `xlsx`.

- **Every** filter that was applied goes into the report header as one line. This is what 3.0 printed as `iconditions`; the mockups show it as "Printed as:".
- The server builds that line from the query, so a print can never disagree with the screen.

---

## 6. Findings from reading the code — read before building

**F1 — OPENING and positive ADJUST / TRANSFER points sit in the balance but are not lots.**
- `lld_lot_balance` is GENERATED for every positive row.
- `lots()` (`:292`), `redeemable()` (`:342`), `recomputeMembers()`'s `next_expiry_on` (`:260`) and the expiry sweep `expiryRun()` (`:1444`) all filter on **`lld_txn_type = 'EARN'` only**.

The comment at `:286` says this is deliberate until opening balances are loaded. But **ADJUST(+) and TRANSFER(+) take the same path**. A +50 goodwill adjustment:
- raises `lmb_balance_points`;
- is never redeemable;
- never expires;
- never shows as a lot.

So **balance ≠ redeemable + cooling + Σ lapsed-not-swept** for any member who ever had an adjustment or an opening.

- **Live today:** 8 ledger rows, all EARN, so it does no harm yet.
- **The report:** it follows the service (§4), and shows the difference as `unlotted` (§5.3) so the gap is visible rather than silent.
- **The fix belongs in `LoyaltyLedgerService`**, in all four places at once, as `:288` itself says. **D2.**

Note: the mockup's statement shows an ADJUST +50 lot that "never" expires. That is the mockup assuming F1 is fixed. Until it is, that row shows under "Unlotted 50".

**F2 — the "one wallet per customer" scope and `lmb_lsc_id`.**
`ux_lmb_cust` is UNIQUE on `(company, customer)`, and the member row carries ONE `lmb_lsc_id`. But each ledger row carries its own `lld_lsc_id`, and a member can earn on two schemes (Gold + Festive in the mockup).

- **Members tab:** shows the member's `lmb_lsc_id` scheme and the **whole** wallet balance.
- **Scheme summary:** splits by `lld_lsc_id`.
- **Value ₹ on the Members tab** uses `lmb_lsc_id`'s rate. If two schemes have different rates, that value is approximate. **D4.**

**F3 — the menu** (§2).

**F4 — the plan says `fn_loyalty_lots`; the live database has none.**
Schema `sales` holds zero functions (`loyalty-ledger.service.ts:44–55` explains why). Every "fn_…" in older share files means the NestJS method now.

---

## 7. Actions on the screen (not report routes)

These write. They belong in the loyalty module with their **own** URLs, not under `/reports`. Both go through `LoyaltyLedgerService.writeLedgerRows`, the one funnel (`:84`; `no-outside-writer.spec.ts` enforces it).

### 7.1 `POST /api/v1/loyalty/members/status`

**Body:**
```
{ companyId, memberId, status: 'ACTIVE' | 'SUSPENDED' | 'CLOSED', reason }
```

- It writes `lmb_status` and `lmb_block_reason`, plus a `txn_status_log` row. The screen's Ctrl+H History reads that log.
- It writes **no** ledger row.
- **MERGED is not settable here.** Merge is its own operation and is not built.
- **Guard:** CLOSED is refused while the balance is > 0, unless `force` is sent and the caller has the right. **D6.**

### 7.2 `POST /api/v1/loyalty/members/adjust`

**Body:**
```
{ companyId, branchId, memberId, points (≠ 0), reason, approvedBy }
```

- It writes ONE `ADJUST` row through `writeLedgerRows`. `ck_lld_adjust_approval` already refuses a row without `approvedBy`. The client asks for the approver in the dialog.
- A **negative** adjust is refused beyond the redeemable balance, and draws on lots FIFO through `consume()`, like a redeem.
- A positive adjust: see F1.

### 7.3 SMS — deferred, button hidden

**User, 2026-10-05: "later we can decide SMS module".** It is not part of this plan.

There is no SMS gateway anywhere in `nexerp-api-ro`: no `sms` module and no provider config.

- The Expiring tab's **Send SMS** button and **Last SMS** column stay hidden until one exists.
- The tab still works as a list with Print / Excel.
- When a gateway is planned, it gets its own plan. It will need an `sms_log` table, which is where Last SMS will come from.

---

## 8. Rules only the service can enforce

- `companyId` is required on every route. Never return another company's members.
- A member with `lmb_is_deleted = true` never appears. MERGED members appear only with `includeMergedClosed`, and their balance should be 0.
- Dates are company-local. "Today" is not the server's UTC date.
- `limit` ≤ 200 on list routes. The export is the only route that streams everything.
- The Total row and the tiles come from `summary`, never from summing a page on the client.

---

## 9. Client side (Qt), for reference

Not for Prathap. It is listed so both sides agree on the shape.

- **Files:** `src/modules/sales/report/loyalty_status/`, with its own `.ui` on `NexDialog` and three tabs. Registered as menu **79** in `form_sales_menu.h`.
- **Tab 1:** strip + tiles + member grid. Enter fills the statement (§5.2) and the card (§5.3).
- **Tab 2:** strip + tiles + expiring grid + calendar. No tick boxes or SMS panel until §7.3.
- **Tab 3:** strip + scheme grid. Enter fills monthly (§5.7) and gifts (§5.8).
- Filters are kept per tab. Ctrl+Tab switches tabs.
- Enter on a source document opens that document. Enter on a gift switches to tab 1 with `eligibleOnly` on that scheme.

---

## 10. Test checklist

The live database has 4 schemes, 8 members and 8 EARN rows. That is enough for the shape, not for the edges.

1. `members` with no filters returns 8 rows. Its `summary.outstanding` = Σ `lmb_balance_points` (88 today).
2. `redeemable` on a member = `LoyaltyLedgerService.redeemable()` for the same member and day. Check this for every member (D1 parity).
3. A bill posted today, whose scheme has `lsc_activation_days > 0`, shows its points under `cooling`, not `redeemable`, and drops out of `eligibleOnly`.
4. `earnedFrom`/`earnedTo` around a bill that was later returned: earned nets the reversal.
5. `statement` closing = `lmb_balance_points`. With `showReversals = false` the closing is unchanged.
6. `expiring` with `withinDays = 365` sums to Σ lots expiring within a year.
7. `schemes` totals: Σ outstanding over all schemes = Σ `lmb_balance_points` over all members.
8. `schemes/gifts`: eligibleMembers for the cheapest gift ≥ that of every dearer gift.
9. F1 visibility: after an ADJUST +n, `member.unlotted` = n.
10. `adjust` without `approvedBy` → 400 from the DTO, before the database answers.
11. `status` CLOSED with a balance > 0 → refused.
12. `export` (pdf) header carries every filter that was sent.

---

## 11. Decisions

| # | question | recommendation |
|---|---|---|
| D1 | Call `LoyaltyLedgerService` reads, or copy their WHERE | call it for one member (§5.3); copy into set SQL for the lists, with a pointer comment |
| D2 | F1: should OPENING / ADJUST(+) / TRANSFER(+) be lots? | yes, in `lots`, `redeemable`, `next_expiry` and `expiryRun` together, before any opening balance is loaded |
| D3 | `branchId` meaning | Members / Expiring: the member's `lmb_branch_id` (home branch). Scheme summary: `lld_branch_id` (where the movement happened) |
| D4 | Value ₹ for a member holding points on two schemes | price each lot at its own `lld_lsc_id` rate. Exact, and one more join in the lots lateral |
| D5 | Gift `inStock` source | the same current-stock read the sale bill uses for the item and branch. Prathap to name it |
| D6 | CLOSE with a balance | refuse. Allow with `force` + an EXPIRE-style ADJUST to zero, approver required |
| D7 | Keep menu 206 | yes, untouched. It is agent loyalty |

---

## 12. Do not build

- The SMS send, `sms_log` and the Last SMS column. The SMS module will be decided later (§7.3).
- Member merge.
- Anything for agent loyalty (menus 205–207 / 212).
- A grid registration for any of these lists. They are report routes (§3).
- A database function, trigger or view for anything in this plan. The logic is written in NestJS only (§3, F4).
