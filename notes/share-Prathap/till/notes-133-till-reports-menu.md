# Notes 133: a menu for Till Reports (2026-10-17)

**STATUS:** done — menu **288** added 2026-10-17 12:26; Qt registers TillReportsEntry under 288. §4 questions still open.

## 1 · The ask: one new menu row

The Till Reports screen is built on our side (Qt). It needs a menu so it can be opened and granted.

| Field | Value |
|---|---|
| Parent | **271** Till |
| Name | **Till Reports** |
| Position | 2.50 (after Till Sessions 273) |
| Visible | yes |
| Verbs | `{VIEW, PRINT, EXPORT}` |

Please tell us the id you give it; we register the screen under it.

**No new route is needed.** The screen only reads registered grids:
- **159–169** (new, registered 2026-10-17 from `grids/till_reports_registration.py`);
- **157** (remittances), **133** / **136** (counter and safe pickers) and dropdown **61** (branches).

## 2 · What the screen shows

It is one read-only screen with four tabs. The mockups are in `till/till_report_*_ui_mockup.png`.

| Tab | Grids | What |
|---|---|---|
| Cashier variance | 159, 160 | Per cashier: sessions, hours, net sales, shorts and excesses, recounts, force closes, card / UPI gaps, pending, treatments. Below it, that cashier's sessions. |
| Exceptions | 161, 162, 163 | Per person: pickups, top-ups, recounts, force closes, reopens, wrong PINs, voided movements, re-tenders, suspends, sessions over `till.session_max_hours`. Then a by-hour strip and the events themselves. |
| Approver activity | 164, 165, 166 | Per approver: approved / recount / rejected, amounts, decisions on their own session, median wait. Then the decisions, and the requests nobody decided. |
| Safe book | 167, 168, 157 | One safe, one row per day: opening, in, out, book closing, the day's safe count and its difference. Then that day's lines with a running balance, and the remittance register. |

## 3 · For information: grid 158 (Cash Office safe book) was counting till transfers twice

Since notes 128, a till voucher (TFlt / TDrp / TRcv / CRem) carries a CASH tender row on the safe's ledger. Grid 158 picked those rows up as "money in / out" **as well as** the movement itself.

For example, the float **tfl00001** (17-10) showed twice in the Cash Office's safe book.

Your `flowsOf()` already leaves these rows out (`td_src_doc_type <> 'VOUCHER'`). We made grid 158 do the same on 2026-10-17, so it matches GET `/till/safes/balance` again. Nothing is needed from you.

This is one of the readers notes 129 §3 asked about. Please still answer §3 for the server's own readers (Cockpit expected, day Z, Business Day, 40A(3)).

## 4 · Two small questions (no rush)

1. **Refused approvals are not journalled.** When a self-approval or a no-grant approval is refused, nothing is written. The Approver tab therefore cannot count those attempts. Would you journal a `till_event` (for example `APPROVAL_REFUSED`, with the approver and the reason code)? This is optional.
2. **Recount decisions.** These are stored as `tap_status = 'REJECTED'` with `tap_rule_snapshot->>'decision' = 'RECOUNT'`. The report reads them that way. Please tell us if you plan a separate status, so we can change the report with it.
