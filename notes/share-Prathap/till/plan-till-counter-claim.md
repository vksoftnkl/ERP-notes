# Till: how a session gets its counter (counter claim) — 2026-10-08

> **User question, 2026-10-07:** *"how will you map without a device"*. Answer agreed in chat
> 2026-10-08 (why `till_counter` ≠ `device_master`); this file writes the rule down.
>
> **A separate plan, per house rule.** `TILL_DESIGN.md`, `plan-till-review-rev2.md` and
> `plan-till-operator-to-user.md` are not edited. §0 lists what this file overrides.
> **Where they disagree, this file wins.**
>
> **SQL 47/48/49 were edited in place on 2026-10-08** after the user's yes (§6); the originals are in the
> session scratchpad. `47_TABLES_GUIDE.md` was updated to match.

## 0 · What this overrides / fills in

| In | Section | Now |
|---|---|---|
| TILL_DESIGN.md | §7.2 `POST /till/sessions/open` "counter (from device)" | The counter comes from the claim rule in §2: **linked** → that counter; **unlinked** → the counter the cashier picks from the free list. |
| TILL_DESIGN.md | §7.2 `GET /till/sessions/current` | Replaced by `GET /till/sessions/open-check` (§4), which returns everything S1 needs in one call. |
| TILL_DESIGN.md | §7.3 `TILL_SESSION_WRONG_DEVICE` "device not bound to the session's counter" | "device is not the session's device (`tss_device_id`)". See §3. |
| TILL_DESIGN.md | §8 S1 "Counter C01 · Counter 1 (this device)" | The counter line is either fixed (linked) or a picker (unlinked). See §5. |

## 1 · The two tables, one sentence each

- **`fixed.device_master`** = *which machine is talking*. Login security: uid, blocked, last login.
  Its company/branch/user are re-stamped on every login, so they say who used it last, not where it lives.
- **`accounts.till_counter`** = *which drawer the money is in*. Fixed in one branch; owns the float,
  the cash limits, the safe and the never-reset Z number. Every session, count and variance points here.

`tcn_device_id` is the **optional** link between them: "this PC normally sits at this counter".
`ux_tcn_device` already makes it one counter per device.

## 2 · The claim rule

The server decides; the client only shows the result. Evaluated at `open-check` and again inside
`open` (the second time is the one that counts).

| # | The device is… | Then |
|---|---|---|
| 1 | **not registered**, or **blocked** in `device_master` | No session. `TILL_DEVICE_UNKNOWN` / existing blocked-device error. |
| 2 | **`till.require_session = false`** (DEVICE setting) | **Back-office PC.** No session is offered or needed. It can still post bills with no money tender (credit, against-advance); any money tender is refused `TILL_SESSION_REQUIRED` (existing rule). |
| 3 | **linked** to a counter (`tcn_device_id = this dev_id`) | That counter, no choice shown. If the counter is inactive/deleted → `TILL_COUNTER_INACTIVE`. |
| 4 | **unlinked** | The cashier picks one counter from the **free list** (§2.1). One free counter → pre-selected; none → `TILL_NO_FREE_COUNTER`. |

### 2.1 The free list (rule 4)

A counter is offered when **all** hold:

- same company and branch as the login;
- `tcn_is_active` and not deleted;
- `tcn_requires_session = true`;
- `tcn_device_id IS NULL` — a **linked** counter is never offered to another device (its PC
  may be dead; that is §3.3, not a claim);
- **no live session** on it (OPEN / SUSPENDED / COUNTING).

Order: `tcn_sort_order`, then `tcn_code`.

### 2.2 What the session stamps

`tss_counter_id` = the counter from rule 3 or 4. `tss_device_id` = **this device, always**
(linked or not). Nothing is written to `till_counter` or `device_master` by a claim — the claim
lives on the session row and ends when the session closes. The `SESSION_OPEN` event carries
`tev_counter_id`, `tev_device_id` and, in `tev_payload`, `{"claim":"LINKED"|"PICKED"}`.

### 2.3 Two devices claim the same free counter at once

Both are on the same store server, so `ux_tss_counter_live` stops the second insert. The server
maps the unique violation to **`TILL_COUNTER_BUSY`**; S1 reloads the free list and says
"C03 was just taken — pick another". No lock table, no reservation step.

## 3 · After the session is open

### 3.1 The session belongs to its device

Every money post checks `assertLive(sessionId, user, device)`: the device must equal
`tss_device_id`, else **`TILL_SESSION_WRONG_DEVICE`**. This is what stops a second tablet from
billing into someone else's drawer.

### 3.2 Login / resume on the same device

`open-check` finds a live session with `tss_device_id = this device`:

| Session's operator | S1 shows |
|---|---|
| this user | "Resume session C01-261008-02, opened 09:02?" → **Resume** |
| another user | "Counter C01 is in use by RAVI K." → **Supervisor force-close** (existing FORCE_CLOSE) or **Not now**. Not a new session on the same drawer. |

The same user with a live session on **another** device gets "You have session C02-… open on
<device name>" and no new session (`TILL_OPERATOR_BUSY`, online check only, as in 47's header).

### 3.3 Dead PC on a linked counter

The replacement PC is a new `device_master` row, so it is **unlinked** and C01 is not on its free
list. Two ways back:

1. **Counter master** (S10): edit C01, set Device = the new PC. Normal master right.
2. **From S1, supervisor PIN:** "C01 is linked to <old device> — **Re-link to this device**".
   Writes `tcn_device_id`, logs `COUNTER_RELINK` with old/new device in the payload, needs the
   `COUNTER_RELINK` approval (shipped ALWAYS, SUPERVISOR, COUNTER channel).

A live session on the old device moves with it: if C01 has a live session, re-link also re-stamps
that session's `tss_device_id` (same approval), so the cashier carries on from the new PC with the
same drawer, float and bills.

### 3.4 Moving a session to another device (unlinked counters)

Same as 3.3 step 2 but without a link change: supervisor PIN → re-stamp `tss_device_id`,
event `SESSION_DEVICE_MOVE`, approval `COUNTER_RELINK`. Used when a tablet's battery dies mid-shift.
**TRAY** counters keep their existing rule (suspend on A, resume on B, `tss_counter_id` re-pointed).

## 4 · Route: `GET /till/sessions/open-check`

Query: `companyId · branchId · accYear · deviceId · userId`. One call, everything S1 needs:

```json
{
  "requireSession": true,
  "businessDay": { "dayId": "…", "date": "2026-10-08", "status": "OPEN" },
  "linkedCounter": { "counterId": "…", "code": "C01", "name": "Counter 1",
                     "liveSessionId": null, "inactive": false },
  "freeCounters":  [ { "counterId": "…", "code": "C03", "name": "Counter 3", "defaultFloat": 2000 } ],
  "deviceSession": null,
  "userSessionElsewhere": null,
  "carriedFrom":   { "sessionId": "…", "sessionNo": "C01-261008-01", "floatLeft": 2000.00 }
}
```

- `linkedCounter` and `freeCounters` are mutually exclusive: linked → `freeCounters` is `[]`.
- `deviceSession` / `userSessionElsewhere` carry `{sessionId, sessionNo, counterCode, operatorName, openedOn, status}`.
- `carriedFrom` is per counter; for an unlinked device it is filled after the pick (call again
  with `counterId=`), or returned inside each `freeCounters` row — **server's choice, say which.**
- Read-only. `POST /till/sessions/open` re-checks everything; `open-check` is advice, never a lock.

`POST /till/sessions/open` gains `counterId` (required when unlinked, ignored-and-checked when
linked: a mismatch → `TILL_COUNTER_NOT_YOURS`).

### 4.1 New error codes

| code | status | when |
|---|---|---|
| `TILL_DEVICE_UNKNOWN` | 409 | device id not in `device_master` for this company |
| `TILL_NO_FREE_COUNTER` | 409 | unlinked device, free list empty |
| `TILL_COUNTER_INACTIVE` | 409 | linked counter inactive/deleted |
| `TILL_COUNTER_NOT_YOURS` | 409 | `counterId` sent is linked to another device, or not on the free list |

## 5 · S1 change (Qt)

The mockup `till_open_session_ui_mockup.png` changes only its first line:

| Case | Counter line |
|---|---|
| linked | `Counter  C01 · Counter 1  (this device)` — read-only, as drawn |
| unlinked, one free | `Counter  [C03 · Counter 3 ▾]` pre-selected |
| unlinked, several | same combo, empty; F10 disabled until picked |
| unlinked, none | message "No free counter in this branch" + list of busy counters and who holds them; only **Not now** |
| back-office (`require_session=false`) | S1 never opens |

Picking a counter fills Float / Carried from that counter (re-call `open-check` with `counterId`).
`AppSession::counterId()` comes from the **session**, never from the device.

## 6 · Schema changes (APPLIED to 47/48/49 on 2026-10-08, user said yes; validated on a scratch PG 18 — not live)

1. `till_session.tss_device_id` → **NOT NULL**. Every session is opened from a device; force-close
   from the cockpit writes `tss_closed_by`, not the device.
2. Add `COUNTER_RELINK` to `ck_tar_event` and `ck_tap_event`; add `COUNTER_RELINK` and
   `SESSION_DEVICE_MOVE` to `ck_tev_code`; seed the `COUNTER_RELINK` rule (ALWAYS · SUPERVISOR · COUNTER).
   48 and 49 restate all three lists, so the codes went into **47, 48 and 49**; leaving them out of
   48 would make its `ADD CONSTRAINT` fail on the seeded rule row.

No new table, no new column on `device_master`.

## 7 · Decided (user, 2026-10-08: "agree all three")

All three recommendations below are taken.

1. **Web devices.** Live `device_master` has two web uid styles: `WEB-<uuid>` (per browser) and
   `web:<usr_id>` (per **user**). A per-user id cannot be linked to a counter — it would follow the
   person to any browser. **Recommendation:** web devices may only **claim** (rule 4), never be
   linked; the counter master's Device dropdown lists Desktop/Mobile only.
2. **Default for a new device.** `till.require_session` is GLOBAL `true`, so a newly registered
   office PC gets S1 until someone sets it `false` for that device. **Recommendation:** keep `true`
   (money never moves without a session by accident); setting office PCs is a one-time admin job.
3. **Should an unlinked device be allowed at all for a DRAWER counter?** A shop wanting "this PC,
   this drawer, always" can link every counter and then rule 4 never fires. Recommendation: allow
   both; linking is per counter.

## 8 · Tests (when built)

| # | Case | Expect |
|---|---|---|
| 1 | linked PC opens | counter = linked, `claim=LINKED` |
| 2 | unlinked tablet, 2 free counters | both listed, ordered; open with one → `claim=PICKED` |
| 3 | unlinked tablet sends a linked counter's id | `TILL_COUNTER_NOT_YOURS` |
| 4 | two tablets open the same free counter | second → `TILL_COUNTER_BUSY` |
| 5 | bill from device B on device A's session | `TILL_SESSION_WRONG_DEVICE` |
| 6 | same user, live session elsewhere | `TILL_OPERATOR_BUSY` |
| 7 | other user on a device with a live session | no new session; force-close path offered |
| 8 | re-link with supervisor PIN, live session on C01 | link + session device both moved; event + approval rows |
| 9 | office PC (`require_session=false`) | no S1; credit bill posts; cash bill → `TILL_SESSION_REQUIRED` |
| 10 | free list excludes inactive, `requires_session=false`, other branch, live-session counters | yes |
