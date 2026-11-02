---
name: declare-strips-dto-decorators
description: "A `declare` field narrowing an optional base DTO field stays unvalidated when omitted — the base's inherited @IsOptional skips the new decorators; TS suggests `declare` for TS2612"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 8740bb9d-bd83-4f4b-ba99-8e34e3872382
  modified: 2026-10-21T05:20:02.883Z
---

**A `declare` field that narrows an optional base field does not make it required.**
Corrected 2026-10-21: TypeScript DOES emit the decorators of a `declare` field
(`dist/.../sale-return-lifecycle.dto.js` has `__decorate([ApiProperty, RequiredUuid],
AmendSaleReturnDto.prototype, "srId")`); it only emits no class field. The field
still goes unvalidated when omitted because class-validator also inherits the
base's `@IsOptional()`, which skips every validator on undefined/null. A
malformed value IS refused. Result: a missing id ends as a 404 "…with id
undefined" (or a 500) instead of a 400 on the field.

It is a trap because **TypeScript itself suggests it**: narrowing an optional
base property to a required one raises TS2612, whose message offers "add a
`declare` modifier" as a fix. That is what produced the `/receipts/amend`
500-on-missing-`avhVoucherId` defect.

**How to narrow a base property safely** — an initializer, and it must be
neither null nor undefined, because the base's `@IsOptional()` is inherited by
class-validator and skips every validator on a null/undefined value:

```ts
@ApiProperty({ format: 'uuid', required: true })  // required: true is NOT optional
@RequiredUuid()
avhVoucherId: string = '';
```

`required: true` must be stated: the Swagger CLI plugin reads the AST and treats
any property with an initializer as optional, so the initializer that fixes the
validator would otherwise re-break the schema. The two halves have opposite
causes.

**Still broken as of 2026-09-18** (verified, not fixed — different module):
`src/modules/stocks/stock-transfer/dto/save-stock-transfer-receive.dto.ts`
— `declare linkSrcDocId` and `declare linkSrcAccYear` are unvalidated and
report `required: false, nullable: true`.

Pinned by `src/modules/accountsModule/receipt/dto/amend-receipt.dto.spec.ts`.
