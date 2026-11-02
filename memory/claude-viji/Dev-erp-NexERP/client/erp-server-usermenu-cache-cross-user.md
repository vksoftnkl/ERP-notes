---
name: erp-server-usermenu-cache-cross-user
description: "Server HTTP cache keys GET /menu-masters/usermenu by URL only (not user) with a 1s TTL — concurrent users briefly get each other's menu list; client then shows \"You do not have access to this screen\""
metadata:
  node_type: memory
  type: project
  originSessionId: f758b293-d3c1-4595-84b0-6619df03a5d5
  modified: 2026-10-27T05:34:26.841Z
---

Found 2026-10-27 (not fixed — server repo). `HttpCacheInterceptor.trackBy`
(`ERP server/src/common/interceptors/http-cache.interceptor.ts`) builds the key from path + sorted query only;
`MenuMasterController` has class-level `@CacheTTL(1)`, and `getUserMenu()` reads the user from the JWT
(`requestContextService.getUserId()`). So within one second a second user gets the first user's menus
(`x-cache: HIT`). Seen when two verification agents ran as different users: a 108-menu answer replaced a
129-menu one and the client's permission gate ([[erp-menu-permission-gating]]) locked governed screens until reload.

**Why:** any per-user GET with an explicit TTL leaks the same way, for as long as its TTL.
**How to apply:** if a screen flips to "no access" right after a save in a multi-user/agent test, suspect this before
the grant. Fix belongs server-side (user id in the cache key, or skip caching usermenu); user not yet asked to fix.
