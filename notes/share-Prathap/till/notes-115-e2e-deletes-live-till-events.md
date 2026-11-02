# Notes 115: an e2e test wipes the whole business day's till events on the live box (2026-10-14)

**STATUS:** §1–2 verified live 2026-10-14: a server test run after the fix (movement docs tdr00237–tdr00272, cleaned up) left every real session's events of the 14-10 day in place (ZT1-261014-04 … -11 and VIJI PC-261014-04/-05 kept all theirs). §3 open: the lost rows (ZT1-261012-01, VIJI PC-261012-01, the 12th's part of VIJI PC-261012-02, ZT1/VIJI PC-261014-01 … -03) are not back — restored or lost for good?

## What we see

The Session Cockpit's timeline (grid 145, `accounts.till_event`) is empty for real sessions that certainly had events:

| Session | Business day | Counts / movements | Events now |
|---|---|---|---|
| ZT1-261012-01 | 12-10 | 3 / 2 | 0 |
| VIJI PC-261012-01 | 12-10 | 2 / 2 | 0 |
| VIJI PC-261012-02 | 12-10 | 7 / 7 | 10, all from the 14th; its opening and billing events of the 12th are gone |
| ZT1-261014-01 … -03 | 14-10 | 4 / 2, 3 / 2, 3 / 2 | 0 |
| VIJI PC-261014-01 … -03 | 14-10 | 7 / 5, 2 / 2, 3 / 2 | 0 |

Other evidence:
- **Bags received with no CASH_RECEIVED event.** `tdr00202`, `tdr00203` and `tdr00204` are RECEIVED (12:52, 13:16, 13:20 today). `movements/receive` journals CASH_RECEIVED in the same transaction, but there is no event for any of the three.
- **The same goes for the PIN_FAIL** from the Cash Office screen test on `tdr00203`.

The pattern by time is the giveaway:

- **14-10 day:** every event before roughly 17:00 is gone. Every event from 17:57 on is there.
- **Events dated the 14th but on the 12-10 day:** VIJI PC-261012-02's close at 11:14 to 11:21, and SESSION_DAY_ENDED at 10:07. These survive, because their `tev_day_id` is the 12th's day.
- **12-10 day:** everything before ZT1-261012-02 (20:19) is gone. That session's events and the 13th's are intact.

So events were deleted **by business day**, at some point on the 12th after 13:31 and on the 14th between 13:20 and 17:57.

## The cause, read from the source

`test/till-money-paths-http.e2e-spec.ts`, `afterAll` (around line 140):

```ts
const [day] = await prisma.$queryRaw`SELECT tss_day_id::text FROM accounts.till_session WHERE tss_id = ${s}::uuid`;
await prisma.$executeRaw`DELETE FROM accounts.till_session WHERE tss_id = ${s}::uuid`;
if (day) {
  await prisma.$executeRaw`DELETE FROM accounts.till_event WHERE tev_day_id = ${day.tss_day_id}::uuid`;   // ← every session's events of the day
  ...
}
```

The test runs on **Acme Foods / Coimbatore** (`helpers/payment-e2e.ts` COMPANY), the same company and branch as the live data. A session there joins the branch's existing business day, so `tev_day_id = <today>` matches every real session's events of that day. Unlike the other suites, this delete is **not** guarded by "the test created the day".

The guarded suites still have a smaller window:
- `expense-voucher`, `till-counter-claim`, `till-session` (`createdDayId`) and `tender-settlement` (`dayCreated`) all guard the delete.
- But if a suite creates the day (it ran first that morning) and a real session opens before its `afterAll`, that real session's events go too.

## Asked

1. **`till-money-paths`:** delete only the test's own events: `tev_session_id = <test session>` or `tev_device_id = <test device>`, as the first line already does. Drop the `tev_day_id` delete, or guard it with "this test created the day" as the other suites do.
2. **The guarded suites:** make the day-level delete also skip events whose session is not the test's. For example:

   ```sql
   tev_day_id = $day AND (tev_session_id IS NULL OR tev_session_id = ANY($testSessions))
   ```

   Or give the e2e tests their own company / branch, so they never share a business day with real counters.
3. **Can the lost rows come back** (a backup, or the PostgreSQL WAL)? The 12-10 and 14-10 sessions above have no history now. If not, say so, and we'll treat their timelines as lost. Their counts, movements, variances and vouchers are all intact; only the journal is missing.

## Why it matters

`till_event` is the audit journal: who opened, counted, recounted, approved, and who typed a wrong PIN. `test/till-event-single-writer.e2e-spec.ts` already forbids the application from ever deleting it. The tests are the only writer that still deletes from it, and they reach live sessions.
