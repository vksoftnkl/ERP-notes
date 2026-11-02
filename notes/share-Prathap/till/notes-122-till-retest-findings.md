# Notes 122: Full till retest 2026-10-15: server findings

**STATUS:** mostly DONE, verified live 2026-10-15 6:50 PM (see the recheck table at the end). Open: §4 269ST not refused; §2 not reproducible (logs asked); §6 still negative (question).

A full retest of the till on the test box, through the real Qt screens, after notes 117–121 went live:
- till masters, Open Till, bills, cash movements, expense, End Shift (recount → decide → close);
- Approve Cash, force close / reopen / X read, slip check;
- Cash Office (count, remit, confirm, Decide);
- Business Day close with exception → store Z → reopen → re-close;
- prints, error boxes, Esc / Enter;
- the sale bill suite.

What follows is the server's part. The Qt fixes are already in.

## 1 · Money taken by a till that is being counted (unlinked counter)

A device on an **unlinked** (picked) counter whose session is COUNTING or PENDING_APPROVAL takes cash with **no session**:
- bil01257 (create, then post) and bil01258 (one call, `post: true`), 59.00 + 118.00, on ZT1-261015-01 while it was PENDING_APPROVAL;
- on both, the header `sb_session_id` is the session, but the cash tender's `td_session_id` is NULL.

`resolveForMoney`:
- `held` looks only at LIVE statuses (OPEN / SUSPENDED), so it is none;
- `counter` needs `tcn_device_id`, and ZT1 has none;
- so the device counts as "not a till" and the answer is null.

A **linked** counter refuses correctly (the user's screenshot on VIJI PC: TILL_SESSION_NOT_OPEN).

**Ask:** a device that holds a session not yet CLOSED (COUNTING / PENDING_APPROVAL) is still a till device. At the least, judge a named `sbSessionId` even when `held` and `counter` are null.

## 2 · 500 on bills whose items carry the "Plain stock" preset

Every one-call post of a bill whose items have `stock_track_preset` **NONE / Plain stock** (`01a07198-6903-77da-a3ef-78ee70c26a5d`) answers **500 INTERNAL_ERROR**. These are the ZT items of the sale bill suite:
- the same bill without `sbiBillQty` posts;
- a bill with an item that has no preset posts;
- the preset's own description says choosing it is "the same answer" as having no policy.

Seen 3 times in one run and once in the next, so it may depend on stock data.

**Repro body:** `Shared/Prathap/sales/repro-500-tracked-item-bill.json`. POST it to `/bills/create` as VKPOS.

## 3 · An unknown `sbSessionId` → 500

Until today the Qt sale bill / sale order sent the **login** session id as `sb/soSessionId` when there was no till (fixed in Qt: now null).
- On a one-call post the server answered 500.
- On create-then-post it stored the bogus id (bil01256).

**Ask:** an id that names no till session → a coded refusal (e.g. TILL_SESSION_NOT_FOUND on `sbSessionId`), never a 500.

## 4 · 269ST not refused

bil01256 (2,12,400.00 cash, a ZT customer) and bil01271 **posted**. The sale bill suite expects a cash receipt of 2,00,000 or more from one person to be refused under section 269ST. Please check whether the 269ST guard still runs on `/bills/create` with `post: true`.

## 5 · Cash out of a drawer that does not hold it

Expense vouchers paid 3 × 1,180.00 in cash from ZT1-261015-01 while its expected cash was 2,418.00. Expected cash went to **−1,122.00**.

**Ask:** refuse a cash payment beyond the drawer's expected cash, with a code (e.g. TILL_CASH_SHORT_IN_DRAWER: "The drawer does not hold this much cash"). It could be a setting if some stores want it allowed.

## 6 · A past day's safe opening

The store Z of 13-10, closed today (15-10), prints **SAFE Opening −8,326.00**.

`tbd_safe_opening` for a past day seems to be worked out backwards from today's figures, which gives an impossible opening for a day closed late.

**Ask:** freeze the opening when the day opens, or work it out "as of" the day. And never print a negative safe.

## 7 · A missing slip found after close leaves nothing open

The supervisor's slip check on ZT1-261015-01 (card bil01242) recorded "1 with no slip". Yet the close-check shows **SLIP_GAPS GREEN**, and no NONCASH variance was written.

**Question:** should a slip the supervisor could not find raise a slip gap, or a NONCASH variance to decide?

## 8 · Small points

- **"No PIN" worded as "Wrong PIN".** A user with no till PIN who types one gets TILL_APPROVER_PIN "tester1 has no till PIN". The catalogue words that as "Wrong PIN · PIN is empty or wrong". A separate code (e.g. TILL_APPROVER_NO_PIN, as the witness / receiver / supervisor already have) would let it say "You have no PIN · Call manager".
- **Dropdown 68 shows blank names.** TILL APPROVER USERS uses `COALESCE(usr_full_name, usr_display_name, usr_login_name)`, and an empty string is not NULL, so zt-approver and Zt-till-ui-80288 list with blank names. Please use `COALESCE(NULLIF(usr_full_name,''), NULLIF(usr_display_name,''), usr_login_name)`. Qt now falls back to the login name in the day-close popup, but the dropdown itself (Cash Office witness, the safe Decide second person) still shows blanks.

## For the record: tested and fine

- notes 117: force close, session reopen, X read, close-check, close with exception, store Z, day reopen (keeps the Z), re-close (keeps the Z, prints REOPENED ×n);
- notes 118: balance, blind count, Decide with two signatures, CIT remit + confirm, refused remit;
- notes 119 / 120: codes and `user` words seen live;
- notes 121: 12-hour times, treatment, counted-tills expected, safe block.

## Follow-up (2026-10-15, later): re-probed before any server change

- **§2:** the saved repro body **posted** this time (201), where it answered 500 four times earlier. So the 500 is not the payload alone. It may be the stock state of those items, or something running at the same moment (the earlier 500s came while other test runs were posting). Server logs for the earlier 500s (around 12:30 – 13:10 IST, POST /bills/create) would show the real cause.
- **§3:** a one-call bill with a bogus `sbSessionId` was **accepted and stored** (201), with no refusal and no 500 this time. The ask stands: an id that names no till session should be refused with a code.

## Recheck after your changes (2026-10-15, 6:50 PM): verified live

| § | Result |
|---|---|
| 1 | **Fixed.** A COUNTING session on the unlinked ZT1 refuses both paths: create → post (bil01293 stays a draft) and one-call, with "Cannot take payment · Your till is being counted." A bill made BEFORE billing stopped is still taken as a late arrival (bil01294), as designed. |
| 2 | **Not reproducible now.** The saved repro posts (201). Please look at the server log for the earlier 500s (12:30 – 13:10 IST). |
| 3 | **Fixed.** A bogus `sbSessionId` is stored as NULL (bil01291). |
| 4 | **On hold → notes 123 (user: never block the counter; later, warn only)** (the law step in bill posting is commented out). No change seen. The 269ST limit row exists (shared, 2,00,000, REFUSE), yet 2,12,400-cash bills posted (bil01256, bil01271). |
| 5 | **Fixed.** An expense beyond the drawer is refused: "Not enough cash · The drawer does not hold this much cash. · Pay less in cash, or pay by bank." |
| 6 | **Improved, still negative.** A re-close of 13-10 froze opening −1,426.00 (it was −8,326.00), now "as of" the day. It is negative because on 13-10 the test safe issued floats before those sessions' hand-overs were approved (approved on a later day). Should the Z show a negative safe as it is, or flag it? |
| 7 | **Fixed.** SLIP_GAPS AMBER: "1 slip(s) the slip check could not match: ZT1-261015-01 (reported, not enforced)". |
| 8 | **Fixed.** TILL_APPROVER_NO_PIN: "No PIN · You have no PIN. · Call manager.", and dropdown 68 lists every name. |
