---
name: dto-hasownproperty-always-true
description: With target ES2022 every declared DTO field is an OWN property (undefined when not sent), so hasOwnProperty guards nothing; an update must never write `dto.x ?? null` / `?? []`
metadata:
  type: project
---

tsconfig targets ES2022 (useDefineForClassFields on), so `new SaveXDto()` — what the ValidationPipe
builds — defines every declared property as an own key holding `undefined`. `hasOwnProperty(dto, 'x')`
is therefore ALWAYS true, and `applyPresentFields` / the `hasOwnProperty` guards in the Inventory
services do not tell "sent" from "omitted". What actually keeps an omitted field untouched is that its
value is `undefined`, which Prisma skips.

**Why:** item master notes 50 #1 / 67 — `itemCompanyId: dto.item_company_id ?? null`,
`itemBaseUnitId ?? null`, `itemPackingItemIds ?? []` and the price row's `ipmGodownId ?? null` wiped
stored values on every update from a client that did not echo them. Fixed 2026-09-30.

**How to apply:** on an update write `if (dto.x !== undefined) data.x = dto.x ?? null;` — null clears,
absent keeps. Same trap for actor columns (`resolveRecordActor(dto.x_by)` returns null when not sent).
Related: [[declare-strips-dto-decorators]], [[dto-redeclare-keeps-base-swagger]].
