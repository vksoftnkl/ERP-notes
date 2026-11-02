---
name: quotation-save-500
description: Why "quotation not saving" was a 500 — sqSalesmanId is one uuid on the DTO but uuid[] in the DB, so only an ABSENT key saves; the client omits it (notes 44) and the order/bill now send one-element arrays; plus the four-key delete shapes
metadata:
  type: project
---

Found 2026-09-23 by driving the quotation screen's own payload through
`tests/quotation/dryrun.sh` and bisecting the header keys with curl.

**The cause:** `sq_salesman_id` became uuid[] on 2026-09-21
([[salesman-array-decision]]) but `save-quotation.dto.ts` still validates
`sqSalesmanId` as ONE uuid. Prisma refuses the scalar → 500 "Internal server
error" for null, "" and a uuid alike; an array is a 400 from the DTO. Only an
absent key saves. So the client OMITS `sqSalesmanId` and warns on save when a
salesman was picked; restore `idArray(...)` when the DTO becomes
`NullableUuidArray` (asked as notes 44).

**Fixed alongside:** the order sends `soSalesmanId` / `soPackedId` and the
bill `sbSalesmanId` / `sbLoadmanId` / `sbPackedId` as one-element arrays
(`idArray`), and every load reads the first element (`firstId`) — those
routes already had `@NullableUuidArray()`. And the deletes: `/quotations/delete`
and `/sale-orders/delete` need all FOUR keys (`deleteById(GetParams)`),
`/bills/delete` is a POST with `BillKey.toJson()` in the body.

**Why:** the first "not saving" report (the charge-carry keys, see
[[shared-payload-per-dto-keys]]) was real but not the only break; a 500 with
no detail is only found by bisecting the payload against the live route.

**How to apply:** when a create route answers a bare "Internal server error",
dump the screen's payload (`PAYLOAD_OUT=`), start from the DTO's required keys
and bisect — the test scripts under `tests/quotation/` and `tests/dc/` are the
template. Delete what a probe creates with the route's FULL key set, and read
the count back.

**Still client-side:** `validateSaveInputs()` refuses a save when the
customer's freight flag is on and no FREIGHT charge line exists ("Freight of
X is calculated on the items, but the charges grid has no Freight line") —
that one is a rule, not a bug; the operator adds the charge or unticks freight.
