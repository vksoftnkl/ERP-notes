# Notes 108: a Pickup / Top-up needs the cashier's PIN (2026-10-12)

**STATUS:** DONE + verified live 2026-10-12 20:20 (after notes 109 redeploy). On ZT1-261012-02, supervisor zt-approver, witness VKPOS: no PIN → 403 TILL_WITNESS_PIN_REQUIRED; wrong PIN → 403 TILL_WITNESS_PIN "VKPOS's till PIN is wrong" (also through the Qt Cash Movement screen: message shown, PIN box cleared); right PIN through the screen → PICKUP tdr00141 posted, tcm_witness_verified_on stamped. Then voided and the session closed (count 2,000 ACCEPTED, Z 18).

## The gap (the user found it)

A Pickup is two people: a supervisor takes cash from the drawer and the session's cashier witnesses it. Today `/till/movements/create` takes only `witnessBy` (a user id, defaulting to the session's cashier), with **no proof the cashier was there**. A supervisor with 273 Override can record a pickup while the cashier is away, and the cashier is named as the witness anyway.

The user asked: *"how does the cashier know the supervisor recorded a pickup… if the supervisor records it without him knowing, what to do?"*

The blind close count catches a pickup that took no cash (the drawer comes out over). But it is caught late, and the cashier's name is already on the slip.

## Asked

On `/till/movements/create` for **PICKUP** and **TOP_UP**:

1. A new field **`witnessPin`**, **required**. It is the till PIN of the witness: `witnessBy`, else the session's cashier. Check it exactly as `/till/sessions/decide-variance` checks `approverPin`, with the same hashing and the same lockout if there is one.
2. Refusals, with `field: "witnessPin"`:
   - `TILL_WITNESS_PIN_REQUIRED`
   - `TILL_WITNESS_PIN` (wrong PIN)
   - `TILL_WITNESS_NO_PIN` (the witness has no till PIN set; say who must set one)
3. Record that it was verified: a `tcm_witness_verified_on` timestamp, or the `till_event` row with the witness and the time. Then the movement slip and the cockpit can show "witnessed (PIN)".
4. Keep the existing rule: the witness cannot be the caller (422 "A pickup is two people").

DROP and PAID_IN are unchanged: the cashier does them alone.

## Client side

- **Done now (2026-10-12), before this ships:** the cashier sees every Pickup and Top-up on their session, with the time, the amount and who recorded it. This shows on Open Till's live panel and at the top of End Shift, read from grid 147. So a pickup they never handed over is questioned before they count.
- **When this ships:** Cash Movement gets a **"Cashier's PIN \*"** box on Pickup / Top-up. The cashier types it on the supervisor's screen, and it is sent as `witnessPin`.
