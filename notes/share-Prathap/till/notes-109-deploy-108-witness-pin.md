# Notes 109: notes 108 is in the code but not on the server (2026-10-12)

**STATUS:** DONE 2026-10-12 ~20:18: the API on .106 now accepts witnessPin and /api/docs lists it.

## What is wrong

Notes 108 (the cashier's PIN on a Pickup / Top-up) is **written**:
- `src/modules/till/dto/till-session.dto.ts` (saved 19:49): `witnessPin` is on `CreateTillMovementDto`;
- `src/modules/till/services/till-movement.service.ts` (saved 19:50): the three refusals and the stamp;
- the DB column `accounts.till_cash_movement.tcm_witness_verified_on` exists.

But the API that answers on **https://192.168.0.106:3011** is still the old build. Checked four times between 19:56 and 20:20:

```
POST /api/v1/till/movements/create
  { ..., "kind": "PICKUP", "amount": 1, "witnessPin": "0000" }
→ 400  { "message": "Validation failed",
         "errors": [ { "field": "request", "message": "property witnessPin should not exist" } ] }

GET /api/docs-json   → the word "witnessPin" appears 0 times
```

"should not exist" is the ValidationPipe whitelist refusing a property the **running** DTO does not have. So the process on port 3011 was not built from the 19:49 source.

## What to change

1. **Build** on .106 from the current source: `npm run build` (or your build script), so `dist/` is regenerated. Restarting without a build starts the old `dist/` again.
2. **Restart** the service that serves port 3011 (pm2 / systemd / docker, whichever you use).
3. **Make sure no old instance still holds the port:** `ss -ltnp | grep 3011` and `pm2 list`. If a second, older process answers on 3011, stop it.
4. If it runs in **docker**, rebuild the image (`docker compose build` then `up -d`). A plain container restart keeps the old image.

## How to check it is live

Any one of these:
- `https://192.168.0.106:3011/api/docs` → `/till/movements/create` lists **witnessPin**.
- The same POST as above, on a CLOSED session (e.g. `ZT1-261012-01`), answers `TILL_SESSION_NOT_OPEN` instead of "property witnessPin should not exist". The field is accepted and the session rule answers first. Nothing is written.

## Then

Tell Viji. The Qt side is already built and waiting: Cash Movement shows "<cashier>'s PIN \*" on Pickup / Top-up and sends `witnessPin`. Claude then tests it live on the ZT test counter: no PIN, a wrong PIN, the right PIN, and the `tcm_witness_verified_on` stamp.

**Until this is deployed, nobody can post a Pickup or Top-up from the new Qt build**, because the server refuses the PIN field. Drop and Paid in are unaffected.
