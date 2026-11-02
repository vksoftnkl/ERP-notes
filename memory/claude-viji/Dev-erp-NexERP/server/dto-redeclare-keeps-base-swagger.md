---
name: dto-redeclare-keeps-base-swagger
description: Redecorating an inherited DTO property loosens class-validator but NOT Swagger — nestjs/swagger merges base options (e.g. minimum 0) up the prototype chain; use OmitType and redeclare
metadata:
  type: project
---

Redeclaring a property in a DTO subclass (e.g. `@OptionalNumber()` over the base's `@OptionalNumber(0)`)
DOES replace the base's validators in class-validator 0.14 (it drops inherited metadata of the same
`type`, so `customValidation` IsNumber+Min go, `conditionalValidation` IsOptional stays). But
`@nestjs/swagger`'s createPropertyDecorator reads `Reflect.getMetadata` (prototype chain) and merges
the new options OVER the base's, so `minimum: 0` stays in /api/docs; `minimum: undefined` is stripped
by pickBy and cannot remove it. Also target ES2022 → TS2612 on a redeclare without initializer.

**Why:** found 2026-09-30 fixing the stock adjustment header's signed totals (notes 65 §2).

**How to apply:** to change an inherited property's rules, `extends OmitType(BaseDto, ['prop'] as const)`
and declare it fresh (precedent: SaveStockAdjustmentHeaderDto, save-item-composite.dto.ts). Probe
Swagger with `Reflect.getMetadata('swagger/apiModelProperties', Dto.prototype, 'prop')` in a jest
spec (the `@nestjs/swagger/dist/constants` import is blocked by package exports). Never `declare`
the field — see [[declare-strips-dto-decorators]].
