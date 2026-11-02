---
name: acc-group-master-contract
description: "what /account-groups create+delete accept and refuse, and the parent-inheritance rule the ledger form depends on"
metadata: 
  node_type: memory
  type: project
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T12:09:41.390Z
---

All live-tested 2026-09-17 (`tests/ledger/run.sh`). The endpoint is solid — no
defects found.

**Create**
- `accGroupParentId` is **required** (400 "Parent account group is required").
  There is no way to make a new root group through the API.
- **Nature, type and profile are NOT accepted on create — they are inherited
  from the parent** (`accGroupNature`, `accGroupType`, `accLedgerProfile`).
  This is what the ledger form leans on. Legacy Feb-2026 rows on the box predate
  it and carry junk types (`AG`/`BR`/`OF`/`ST`) with a NULL nature.
- `accGroupPrimaryName` is **not** inherited — stays NULL on every custom group.
  The one gap worth raising; harmless for the ledger form.
- Duplicate name → 409. Blank name → 400. Missing parent → 400 "does not exist".

**Create-with-id is the update, and it MERGES**: keys left out keep their stored
value (short name, sort survived). Reparenting **re-derives** nature/type/
profile — silently re-shaping the form for every ledger already under the group,
with no warning and no block.

Refused: self-parent, cycle ("Parent cannot be a child of the same account
group").

**Delete** is soft, and refuses: a reserved group, a group with active children,
a group with active ledgers ("is used by N ledger(s)"). A deleted group then
GETs 404.

Backend gap found on the LEDGER side: an invalid `ledLedgerType` (not in the
`chk_led_ledger_type` list) escapes as a **500**, not a 400 with a field error.

See [[ledger-group-tests]], [[ledger-type-derivation]].
