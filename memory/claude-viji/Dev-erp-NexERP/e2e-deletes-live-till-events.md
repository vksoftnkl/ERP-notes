---
name: e2e-deletes-live-till-events
description: Prathap's e2e suites run on the live Acme company and their afterAll deletes till_event by business day — real sessions' timelines vanish (notes-115)
metadata:
  type: project
---
Found 2026-10-14: Cockpit timeline (grid 145) empty for real sessions (ZT1-261012-01, VIJI PC-261012-01, ZT1/VIJI PC-261014-01..03). Cause: test/till-money-paths-http.e2e-spec.ts afterAll runs `DELETE FROM accounts.till_event WHERE tev_day_id = <its session's day>` unguarded, and e2e uses COMPANY 019c8ea6 (Acme, the live test company) so it shares the real business day. Other suites guard it with dayCreated/createdDayId but still have a window. Asked in Shared/Prathap/till/notes-115-e2e-deletes-live-till-events.md (fix the cleanup, separate test company, restore if possible).

**Why:** till_event is the audit journal; missing events look like an app bug.

**How to apply:** an empty timeline / missing PIN_FAIL / missing CASH_RECEIVED on a live session is probably this, not our code — check the event time gaps by tev_day_id before debugging. Until notes-115 is done, our own live tests can lose their events the same way.

**2026-10-14 later — notes-115 §1–2 fixed by Prathap, verified live:** a test run after the fix (tdr00237–272, cleaned) left all real 14-10 events. Lost rows NOT restored (§3 still open). The local nexerp-api-ro snapshot is stale (rsync publickey fails), so verify server changes live, not from the source.
