# Notes 110: Pickup / Top-up are signed at the counter by two people, and a Drop needs a drop safe (2026-10-13)

**STATUS:** DONE + verified live 2026-10-13 on ZT1-261013-01 (VKPOS cashier, zt-approver supervisor): wrong PIN 403 TILL_WITNESS/SUPERVISOR PIN refused, right PIN → PICKUP tdr00171 + TOP_UP tfl00154 (done_by cashier, witness_by supervisor, verified_on set); Drop greyed while ZTS no drop safe, ticked through the Safe popup → DROP tdr00172; supervisor posting from own login 403. Session closed (count 1,202 ACCEPTED, Z 19). ZTS restored to not-a-drop-safe. This **replaces the roles in notes 108**: the PIN check from 108 stays, but the person who types the PIN flips.

## Why (the user's decision, 2026-10-13)

Today a supervisor logs in, opens the session from Till Sessions (273) and posts the Pickup, and the cashier types their PIN on the supervisor's screen. At a real counter that is backwards. The cashier is the one logged in at the till, so the supervisor would have to log the cashier out or pull them to another PC.

The user wants every hand-over of cash to be **done and signed in front of both people**, so nothing is recorded without one of them knowing. In the user's words: *"if supervisor enter without knowing cashier / vice versa means at the day end it will make too many confusion"*.

The rule agreed:

> A hand-over is **two signatures**, not "do + approve". The person logged in at the counter is one signature (the session's cashier). The other person picks their name and types their till PIN on the same screen. Approval (276 rules, decide-variance) stays for exceptions only.

| Kind | Cash goes | Given by | Received by | Who calls the route |
|---|---|---|---|---|
| DROP | till → **drop safe** (sealed bag) | cashier | — (the cash office counts the bag later) | the cashier |
| PAID_IN | outside → till | — | cashier | the cashier |
| PICKUP | till → supervisor | cashier | supervisor (name + PIN) | **the cashier** |
| TOP_UP | safe → supervisor → till | supervisor (name + PIN) | cashier | **the cashier** |

## Asked

### 1. `POST /till/movements/create`: PICKUP / TOP_UP are now cashier calls

1. **The caller is the session's cashier on its device**, checked exactly as DROP / PAID_IN are (Open Till 272 EDIT + `assertOperator`). The 273 OVERRIDE check on the **caller** goes. A caller who is not the session's cashier is refused:
   `422 TILL_MOVEMENT_AT_COUNTER` (field `kind`) "A pickup / top-up is entered at the counter by the session's cashier, with the supervisor there". This is how "no entry from the office" is enforced, so a supervisor can never post one behind the cashier's back.
2. **Two new body fields, both required on PICKUP / TOP_UP:**
   - `supervisorId` (uuid): the other person.
   - `supervisorPin` (string ≤ 20): that person's till PIN (`usr_pin_hash`, notes 95).
3. **What the supervisor needs** (checked before the transaction, like `proveWitness` today, so a wrong PIN stays journalled):
   - an active user, `TILL_MOVEMENT_INVALID` otherwise (field `supervisorId`);
   - not the session's cashier: `422 TILL_SUPERVISOR_SELF` "A pickup is two people";
   - **Till Sessions (273) OVERRIDE held by the supervisor** (their rights, not the caller's): `403 TILL_SUPERVISOR_RIGHT` "<login> has no Till Sessions override right";
   - a PIN: `422 TILL_SUPERVISOR_NO_PIN` if they have none set, `403 TILL_SUPERVISOR_PIN_REQUIRED` if none was typed, and `403 TILL_SUPERVISOR_PIN` if it is wrong (journalled PIN_FAIL, the same as 108).
   All with `field: "supervisorPin"`, except the first three, which use `field: "supervisorId"`.
4. **What is recorded** (no new column):
   - `tcm_done_by` = the cashier (the caller);
   - `tcm_witness_by` = the supervisor;
   - `tcm_witness_verified_on` = when the supervisor's PIN was checked.

   The kind gives the direction: on a PICKUP the cashier gave and the supervisor received; on a TOP_UP the supervisor gave and the cashier received. Please put that in the DTO description.
5. `witnessBy` / `witnessPin` are **no longer read** on PICKUP / TOP_UP. You can drop them from the DTO or ignore them there. The client stops sending them.
6. `/till/movements/change` (counter to counter) and `/till/movements/void` are **unchanged** for now. No screen calls `/change`. Void stays a 273 OVERRIDE correction.

### 2. `GET /till/movements/supervisors`: who can sign

Query: `companyId, branchId, accYear, tssId`. It returns the users who could sign a pickup / top-up on that session:
- active and not deleted;
- holding Till Sessions (273) OVERRIDE for this company / branch;
- **not** the session's cashier.

```json
{ "success": true, "data": [
  { "userId": "…", "loginName": "MANI", "fullName": "Mani S", "hasPin": true }
] }
```
Rights: the session's cashier (Open Till 272 VIEW) or 273 VIEW. `hasPin` lets the screen say "<name> has no till PIN — set it in User Administration (menu 58)" before the cashier counts.

### 3. A DROP needs a drop safe

The user: *"need drop safe"*. A drop is a sealed bag put into a drop box at the counter, with nobody receiving it. If the bag goes into a supervisor's hand instead, it is a Pickup.

1. **SQL** (a CHECK-free column, per the house rule): `ALTER TABLE accounts.till_safe ADD COLUMN IF NOT EXISTS tsf_is_drop_safe boolean NOT NULL DEFAULT false;`
2. **Safe master** (`/till/safes/*` save + get + the list grid): carry `tsfIsDropSafe`.
3. **DROP** on `/movements/create` goes **only to the counter's own safe** (`tcn_safe_id`), with no fallback to the branch default, and that safe must have `tsf_is_drop_safe = true`. Otherwise:
   `422 TILL_NO_DROP_SAFE` (field `kind`) "Counter <code> has no drop safe: give the bag to a supervisor as a Pickup, or set a drop safe on the counter (Till Masters, menu 275)".
4. **`/till/sessions/get`** returns `dropSafeName` (string, or null when the counter has no drop safe). The screen disables the Drop tile from it.

PICKUP / TOP_UP keep today's safe choice (the counter's safe, else the branch default).

### 4. Old rows

Pickups and top-ups posted before this have the roles the other way round (`done_by` = the supervisor, `witness_by` = the cashier). They are all ZT- test data on .106, so nothing needs migrating. Note it if real data ever has some.

## Client side (built 2026-10-13, waits for this)

- Cash Movement (S3): on Pickup / Top-up there is a **Supervisor** picker (filled from §2), with a **"Supervisor's PIN \*"** box beside it. The count comes first and the PIN last, so the PIN signs the counted amount. It sends `supervisorId` + `supervisorPin`. A refusal on `supervisorPin` clears the PIN box and puts the cursor back in it. The summary card reads **Given by / Received by**.
- The Drop tile is disabled when `dropSafeName` is null, and its tooltip says why.
- The "Cash movement" entry on Till Sessions (273): the F5 row action and the cockpit button are **removed**. These moves are only entered at the counter. Open Till's live panel and End Shift still list every pickup / top-up (grid 147).
- The Drop tile is only disabled when `dropSafeName` is **present and null**. Until §3 ships the key is missing, and Drop behaves as it does today.
- Safe master: the "Drop safe" tick (`tsfIsDropSafe`) is added **once §3 is live**. Today the save refuses unknown fields ("should not exist"), so adding it now would break every safe save. The safe list grid also needs `tsf_is_drop_safe` in its SELECT; the tick is filled from the grid row.
