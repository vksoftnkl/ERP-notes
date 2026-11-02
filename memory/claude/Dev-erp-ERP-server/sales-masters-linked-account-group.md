---
name: sales-masters-linked-account-group
description: "state/city/area masters each share a PK with a linked account_groups row that must be kept in sync on create, update, AND softDelete"
metadata: 
  node_type: memory
  type: project
  originSessionId: d2efb5a2-3f81-4035-8bbf-cf327f65d203
---

The sales masters **state** (`stm*`), **city** (`ctm*`), and **area** (`arm*`) each create a row in `accounts.account_groups` whose `accGroupId` **equals the master's PK** (`stmId`/`ctmId`/`armId`) — the master id is set explicitly to the freshly-created group's id, not the model's `uuidv7()` default. The group is created under the fixed parent group `019f081c-6764-73b0-b397-3f30a6efe73e` (`STATES_ACCOUNT_GROUP_ID` / `*_ACCOUNT_GROUP_PARENT_ID`).

The mirrored field subset is: `accGroupName←name`, `accGroupShort←short`, `accGroupDescription←description`, `accGroupSort←order/sort`, `accGroupIsActive←!isDeleted`, `accGroupIsDeleted←isDeleted` (+ modified audit). **Alias is NOT mirrored**, and area's `armDistanceKm`/`armCollectionDays` are NOT mirrored.

Any write path that mutates a master's mirrored fields must update the linked group in the SAME `$transaction` via `tx.accountGroup.updateMany({ where: { accGroupId }, ... })` (updateMany is a deliberate no-op for legacy rows with no linked group). This applies to create, **update**, and **softDelete** (softDelete sets the group `isActive:false/isDeleted:true`).

Type/width mismatches to guard when mirroring: `accGroupSort` is `Int?` but master order/sort are `Decimal` → `Math.trunc(toNumber(...))`; `accGroupDescription` is `VarChar(250)` but master description is unbounded `Text` → `.slice(0, 250)`. Skipping these aborts the whole transaction (P2000 / non-integer Int).

The HTTP write endpoint is a single `POST <module>/create` that dispatches on PK presence: id present → `service.save()` → `updateX` (returns bare payload); absent → `createXMaster` (returns `{ master, accGroupId }`). All three controllers (state/city/area) follow this; see [[prisma-schema-is-generated]] for schema-fragment editing.
