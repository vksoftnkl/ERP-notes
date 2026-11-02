---
name: erp-auth-endpoint-verification
description: How to call authenticated ERP server endpoints locally for verification (mint JWT + Redis session)
metadata: 
  node_type: memory
  type: reference
  originSessionId: f0e4ef84-196d-4387-ace6-145a47300a81
  modified: 2026-10-03T10:45:28.146Z
---

To hit a Bearer-protected ERP server endpoint locally (e.g. verify `configured-grid-sql/columns`):

1. **Secret:** use the **second** `JWT_SECRET` line in `ERP server/.env` — there are two and the later one is the value the running server uses (`sed -n '25p'`, not line 23).
2. **Mint token:** HS256 JWT, header `{alg:HS256,typ:JWT}`, payload `{sub,user_name,sid,iat,exp,typ:"access",user_type,company_id}` (`user_type`/`company_id` were added later, both nullable — `null` works fine for local verification), signature = `HMAC-SHA256(secret, header.payload)` base64url. See `ERP server/src/modules/auth/token.service.ts`.
2b. **Make `sid` a real UUID** (`crypto.randomUUID()`), not a readable label. GETs and the
   guard accept anything, so a `"verify-123"` sid looks fine — until a transaction SAVE 400s
   with `sqSessionId must be a valid UUID`: the quotation DTO validates the session id the
   server copies out of the JWT. Cost a debugging cycle on a save that looked like a client bug.
3. **Redis session is required** — the guard checks `auth:session:{sid}` in Redis. Write `{sid,sub,user_name,token_hash,refresh_token_hash:"",iat,exp}` where `token_hash = sha256_hex(token)` (plain SHA-256, **not** HMAC — see `auth-session.service.ts` `hashToken`). `redis-cli set "auth:session:$sid" '<json>' EX 600` — pass the JSON as an argument; `redis-cli -x set key EX 600` appends stdin AFTER `600`, the set fails, and every call 401s "Access token is no longer active". A real user for `sub`/`user_name`: `public.user_master` (`usr_id`, `usr_login_name`).

DB for lookups: `postgres://...@localhost:5432/ERP`, grid tables live in the `fixed` schema (`fixed.grid_details`, `fixed.grid_columns`). Related: [[erp-workspace-layout]], [[erp-grid-columns-config]].
