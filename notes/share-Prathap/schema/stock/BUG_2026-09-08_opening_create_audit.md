# BUG — 2026-09-08 — `POST /api/v1/stock/opening` (create) is unreachable

> **RESOLVED 2026-09-08 ~10:30.** The same payload now returns
> `Opening stock created successfully`. Two DRAFT vouchers exist in
> `stock.stock_voucher_2026_2027` (`…/1` from this repro, `…/2` a
> batch-tracked variant). Left here for the record; see the "Asks"
> section — items 2 and 3 may still be worth a look.

**Severity: blocker.** No opening stock document can be created through the API
by any client. The update branch is fine; only create is affected. Found while
smoke-testing the endpoint from Swagger/curl against
`https://192.168.0.106:3011`, company `019cc885-d0f4-771b-a7d1-7c98f9ff3ac1`
(login `VKPOS`). `stock.stock_voucher_2026_2027` is still empty — nothing was
written by any of the attempts below.

## Symptom

A body that passes every DTO rule is rejected by a later audit check:

```
HTTP 400
{"success":false,"message":"Validation failed",
 "errors":[{"field":"request",
   "message":"originalRecord and modifiedRecord are required for update audit log"}]}
```

## Why it is unreachable — the two layers contradict each other

| # | What was sent | Response |
|---|---|---|
| 1 | Valid body, no `originalRecord`/`modifiedRecord` | 400 `originalRecord and modifiedRecord are required for update audit log` |
| 2 | Same body **with** both properties at the top level | 400 `property originalRecord should not exist` + `property modifiedRecord should not exist` |
| 3 | Same body, both passed as **query params** | 400, audit message again (params ignored) |
| 4 | Same body, both passed as headers `x-original-record` / `x-modified-record` | 400, audit message again (headers ignored) |

So the audit guard demands two properties that `forbidNonWhitelisted` on
`SaveOpeningStockVoucherDto` rejects, and there is no third channel that
satisfies it. `grep originalRecord` over the published spec
(`/api/docs-json`) returns **0 hits** — no DTO in the whole API declares
either property, so no client can be written that supplies them.

## The check does not fire on the update branch

Identical body plus `header.svhId` set to a non-existent UUID:

```
HTTP 404
{"success":false,"message":"Opening stock not found",
 "errors":[{"field":"svhId",
   "message":"No OPENING voucher 019f8500-0000-7000-8000-000000000001 in 2026-2027."}]}
```

It reached the lookup, so the audit guard was satisfied on that path. That
narrows it to the **create** branch of the service calling the update-audit
helper (presumably a shared `writeAuditLog(..., {originalRecord,
modifiedRecord})`) without the record pair — on a create there is no original
record to pass, so the call should be the insert-audit variant, or the pair
should be optional when the operation is an insert.

## Ordering, for reference

DTO validation runs first and the audit guard second:

- `{}` → `lines must be an array` (DTO), no audit error.
- `{"header":{},"lines":[]}` → ten per-field DTO errors, no audit error.
- Full valid body → **only** the audit error, i.e. every field passed.

`POST /stock/physical` and `POST /stock/transfer` with `{}` return ordinary
field validation, so they do not reach this guard — whether they hit it with a
complete body is untested.

## Repro

```bash
TOK=$(curl -sk -X POST https://192.168.0.106:3011/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"usrLoginName":"VKPOS","usrPassword":"VKPOS"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

curl -sk -X POST https://192.168.0.106:3011/api/v1/stock/opening \
  -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d @opening.json -w '\nHTTP %{http_code}\n'
```

`opening.json` sits beside this note in `schema/stock/` — run the curl from
that directory. Every id in it is a live row on this DB (branch "saw mill" is the
only branch of the company, device "ravi" the only device on it, godown
dharmapuri, and the two items are the only non-service items with unit
conversions; both base-factor 1, neither batch- nor expiry-tracked):

```json
{
  "header": {
    "accYear": "2026-2027",
    "companyId": "019cc885-d0f4-771b-a7d1-7c98f9ff3ac1",
    "branchId": "019d9b39-3dbd-702a-8c7a-90aa1bee679a",
    "deviceId": "019e7282-3e94-77e7-a7f5-7a63f12c42d4",
    "docDate": "2026-04-01",
    "toGodownId": "019db036-36c7-7104-b998-73150eb86a21",
    "rateSource": "MANUAL",
    "lineCount": 2, "totalQty": 30, "totalValue": 1300, "totalValueWot": 1300,
    "remarks": "Swagger test opening"
  },
  "lines": [
    { "lineNo": 1, "itemId": "019f84c2-7767-7055-a2a6-e07212104e72",
      "uomId": "019f84c2-776f-7221-b596-a758410b7390",
      "baseUomId": "019f84c2-776f-7221-b596-a758410b7390",
      "toBaseFactor": 1, "godownId": "019db036-36c7-7104-b998-73150eb86a21",
      "bucket": "SALEABLE", "qty": 10, "baseQty": 10, "costRate": 100, "taxPerc": 0 },
    { "lineNo": 2, "itemId": "019e8384-0695-76cf-a9bc-30e1f0460e54",
      "uomId": "019f22be-f112-7790-ab2b-1f7a3b26d59a",
      "baseUomId": "019f22be-f112-7790-ab2b-1f7a3b26d59a",
      "toBaseFactor": 1, "godownId": "019db036-36c7-7104-b998-73150eb86a21",
      "bucket": "SALEABLE", "qty": 20, "baseQty": 20, "costRate": 15, "taxPerc": 0 }
  ]
}
```

## Working for comparison

`GET /api/v1/stock/opening?companyId=…&branchId=…&accYear=2026-2027` returns
`200 {"items":[],"meta":{"limit":50,"offset":0,"count":0}}`. Auth is fine.

## Asks

1. Fix the create branch so it does not require the update-audit record pair.
2. Whichever way it is fixed, the fields must not become part of the request
   contract — the Qt client sends no audit payload on any route
   (`src/core/api/api_client.cpp` sets only Content-Type, Accept,
   Authorization and X-Device-Id), and nothing in the spec documents them.
3. Worth a check of the same helper on `/stock/physical` and
   `/stock/transfer` create paths before those are smoke-tested.

Once it is in, the payload above is ready to re-run and I will carry on with
post / validate / cancel.
