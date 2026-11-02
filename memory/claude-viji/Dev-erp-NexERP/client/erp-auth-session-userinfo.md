---
name: erp-auth-session-userinfo
description: Token refresh blanked the user block (userType, then deviceId) — refresh response is partial; setAuthSession AND the authSlice reducer merge per field, deviceId falls back to the JWT `device_id` claim (3rd round 2026-09-29)
metadata:
  type: project
---

`lib/auth/session.ts` `setAuthSession(token, userId, refreshToken, userInfo?)` writes the
**persisted** auth blob (`sessionStorage["erp_client_redux_state"]`) that a reload rehydrates
from. Until 2026-08-31 it did `userInfo ?? null`, and BOTH token-refresh paths
(`hooks/useApi.ts`, `store/api/baseApi.ts`) call it with three arguments — so the first refresh
blanked `userInfo` (redux kept it in-tab, because the reducer only applies keys that are
present), and the next page reload came up with `userType: null` **for the rest of the session**.

Nothing read `userType` until the SUPER-ADMIN gate on "Save as Default Template"
([[erp-customer-form-defaults]]), which is how it stayed invisible — the button simply was not
there, on a screen where it had worked an hour earlier.

Fixed: `setAuthSession` keeps the stored block unless the caller supplies one with content
(an all-null `extractUserInfo({})` counts as nothing), sign-out still goes through
`clearAuthSession`, and both refresh paths now pass `extractUserInfo(response.data)` — the
refresh response reports `user_type`, so a stale block also self-heals. `lib/auth/session.test.ts`
locks it (stub `window` + fake sessionStorage, re-import per case); the two "keeps the user
block" cases fail on the old line.

**Second round, same file (2026-08-31):** keeping the block "unless the caller supplies one with
content" was still too coarse. The refresh response is PARTIAL — it reports `user_name` /
`user_type` and sends every device field as `null` (the server carries the real device inside the
token; see `auth.service.ts` refresh) — so it counted as content and overwrote `deviceId` with
null a quarter of an hour into every session. That is the id `txn_hold` needs
([[erp-txn-hold-contract]]), so F9 started answering "this session has no registered device" on
sessions that had merely lived long enough to refresh. `setAuthSession` now merges **per field**
(`mergeUserInfo`): a null keeps what is stored, unless the incoming block names a different
`userName`, which inherits nothing. Two more cases in `lib/auth/session.test.ts` lock it.

**Third round (2026-09-29) — the Redux copy.** Round two fixed only `setAuthSession`. Both refresh
paths (`store/api/baseApi.ts`, `hooks/useApi.ts`) then dispatched `authSessionChanged` with the RAW
partial block, and `applyAuthSession` replaced `state.userInfo` wholesale → Redux lost `deviceId`,
and the store's 250 ms persistence subscriber (`store/store.ts`) wrote that over the merged
sessionStorage copy, so the next reload lost it for good. Fix: the reducer merges with the exported
`mergeUserInfo`; refresh + login dispatch `getUserInfo()` (the stored, merged block); and
`setAuthSession` fills a missing `deviceId` from the access token's `device_id` claim (server signs
the device_master id in at login and copies it forward on every refresh — the token is the
authority, the refresh BODY always says null). Tests: `store/slices/authSlice.test.ts` + a token
case in `lib/auth/session.test.ts`. Any NEW writer of `userInfo` must go through one of these two
merges.

**Symptom to recognise:** a role-gated control that disappears after a refresh/reload and comes
back only on re-login. Check
`JSON.parse(sessionStorage.erp_client_redux_state).auth.userInfo` first.
