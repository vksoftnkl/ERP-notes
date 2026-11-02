---
name: redis-usage-audit
description: "State of Redis in the API as audited 2026-10-07 — HTTP cache key ignores user/company, sessions have no TTL, BullMQ dead, what must never move to Redis"
metadata:
  node_type: memory
  type: project
  originSessionId: f85553be-872d-485d-8fdd-7dd5a8f3555f
  modified: 2026-10-07T06:20:05.759Z
---

Audit of 2026-10-07 (no code changed yet):

- Three Redis clients coexist: hand-rolled RESP `RedisCacheService` (auth sessions + health, no command timeout), `cache-manager-redis-store` (redis v4) behind Keyv for the HTTP cache, ioredis via BullMQ. `redis` v5 dep is unused.
- HTTP cache key = path + query only. Routes that read token context leak across users when cached: `/menu-masters/usermenu` (HIGH), `/print-template-assignments/resolve`, `/stock-track-presets/get`, `rights` block of `/bills|/delivery-challans|/sale-returns|/dc-returns/get`. Do not raise any TTL before the key carries scope.
- Invalidation = `KEYS` on the first URL segment only, not awaited; stock/price writes never bust `item-stock-balance` or `master-lookups/item-price` (TTL 60, carries stock).
- `auth:session:<sid>` is SET with no EX (2,588 keys locally); with `noeviction` (BullMQ needs it) they grow forever.
- QueueService has zero callers; QueueModule connects even when REDIS_ENABLED=false.
- Out of scope but found: `user-administration` and `device-list-master` controllers are `@Public()` at class level.

**Why:** the next Redis task should start from these findings, not re-audit.
**How to apply:** voucher numbering stays on pg advisory locks ([[stock-voucher-numbering-lock-is-a-noop]]); never Redis INCR, since a rollback would leave gaps in invoice numbers. Keep stock and money in Postgres; cache only reference data.
