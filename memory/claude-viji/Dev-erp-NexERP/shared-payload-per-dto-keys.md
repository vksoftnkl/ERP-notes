---
name: shared-payload-per-dto-keys
description: A shared payload builder (ChargeGridController::collectJson) feeds screens with DIFFERENT NestJS DTOs; forbidNonWhitelisted makes any key one DTO lacks a hard 400, so optional keys are emitted only when they carry a value
metadata:
  type: feedback
---

The API validates every body with `whitelist + forbidNonWhitelisted` (main.ts).
`ChargeGridController::collectJson()` is shared by the sale bill AND the
quotation, but only the bill's `SaveChargeDetailDto` knows the order-carry keys
(`cdSrcCdId` / `cdSrcAccYear` / `cdCarryBasis`); `SaveQuotationChargeDto` does
not. Sending them as null on every row broke the quotation save on 2026-09-23
("unable to save quotation entry" = 400 "property cdSrcCdId should not exist").

**Why:** the bill service reads them with `?? null`, so an absent key and a null
key mean the same thing there; the quotation DTO rejects the key outright.

**How to apply:** when adding a field to a shared builder, emit it only when it
has a value, and grep every `*.dto.ts` that receives that array before
sending a new key unconditionally. Related: [[verify-against-artefact]],
[[charge-auto-apply]], [[sale-bill-upgrade]].
