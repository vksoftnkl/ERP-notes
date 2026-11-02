# Notes 100: a re-tender's new rows carry no session (2026-10-09)

**STATUS:** done. Checked live at 18:30 on 2026-10-09:
- A re-tender in the bill's own session (-15) stamps its new rows with that session.
- A re-tender of a closed session's bill from a new session (-16) stamps both the cash row and tnc00113 with -16.
- -15 accepted 2,090 and -16 accepted 2,500. Cash In Hand nets to 0 in each.
- The rows of the earlier test (bil01099, session -13) were not backfilled, so the +180 stays on the box.

**Order:** after notes 99.

## What was run

As VKPOS on device VKPOS1, counter ZT1:

1. Session ZT1-261009-12: bill bil01099 was paid CARD 1,180. The card batch was counted 1,000. The session closed with the 180 card variance PENDING, as notes 99 §1 designed.
2. Session ZT1-261009-13 was opened on the same device. Then `/bills/retender` on bil01099:
   - voids: the CARD 1,180 row (KEYED_WRONG)
   - tenders: CARD 1,000 + CASH 180

## What the box shows

| What | Session |
|---|---|
| RETENDER event, 1,180 | ZT1-261009-13 ✔ |
| tnc00107: Dr Kvb 1,000 · Dr **Cash In Hand 180** · Cr Kvb 1,180 | ZT1-261009-13 ✔ |
| New tender row CARD 1,000 | `td_session_id` **NULL** |
| New tender row CASH 180 | `td_session_id` **NULL** |

Session -13's `expected()` counts tender rows by `td_session_id`, so it never sees the 180:

- The drawer counted 2,180 and got RECOUNT_REQUIRED.
- The drawer counted 2,000 and got ACCEPTED, then closed.
- The close handed over 2,000, so **Cash In Hand keeps +180** across the two sessions. tnc00107 put it there, and no session's expected figure or handover takes it off.

The README says the RETENDER is logged "in the session the money moved in". The rows the money moved on should carry that session too.

**Asked:** stamp each row `/bills/retender` adds with the session the RETENDER event names (the device's live session, else the bill's own when that is a till session). Then the drawer's expected figure includes the cash taken at the re-tender.

The card 1,000 row has the same gap. Today a card row with no session falls out of every slip check and every batch total.

## Box state left

- Cash In Hand: +180 from tnc00107. This is evidence, not something to fix by hand.
- bil01100 (UPI 118, session -14): written off as SUSPENSE through Settlement 278 (tva00027). The write-off route itself was the test.
