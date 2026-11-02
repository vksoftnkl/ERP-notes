---
name: erp-print-template-save-clears-header
description: "POST /print-templates/create with only {ptlId, versions} NULLS the published pointer and fork lineage — always resend the whole header (open server bug, found 2026-10-13)"
metadata:
  node_type: memory
  type: project
  originSessionId: 7102eab9-29b7-4d7a-be7d-a14231289db7
  modified: 2026-10-13T08:05:35.606Z
---

`print-template.service.ts` decides "key present?" with `hasOwnProperty(dto, …)`
(planHeader + planPointer). The server compiles to ES2022, so the DTO class
instance has EVERY declared field as an own property (undefined) — every key
looks present. A partial update `{ptlId, versions:[…]}` therefore wrote
`ptlPublishedRevId`, `ptlForkedFromId`, `ptlForkedFromRev` (and would write
`ptlCompanyId`/`ptlDescription`) to NULL. Hit it adding SaleInvoiceA5 rev 10;
the template stopped resolving until restored.

**How to apply:** when saving through the API, send the full header copied
from GET `/print-templates/get?ptlId=` (incl. `ptlPublishedRevId`), and diff the
header after. Also: `ptvBody` must be a JSON STRING — an object 400s despite
the DTO comment. Bug not fixed yet (reported to user). Related: [[erp-print-flow-layout]].

**Same root cause, version level (2026-10-15):** a PUBLISHED revision cannot be RETIRED through
the API at all — `planFrozenVersion` refuses on key PRESENCE and every DTO key is present, even
in `{ptvId, ptvStatus:'RETIRED'}` or with identical values. Harmless while the pointer has moved
on (preview and /resolve follow `ptl_published_rev_id`). Publishing a NEW revision: omit
`ptlPublishedRevId` (sending it with a PUBLISHED row is a 400 "not both"); the header survived.

