---
name: till-witness-pin-notes-108
description: "Notes 108 built 2026-10-12 — /till/movements/create PICKUP/TOP_UP wants witnessPin, /movements/change wants fromWitnessPin + toWitnessPin; stamps tcm_witness_verified_on; a self top-up stays one-person"
metadata:
  type: project
---

**Superseded on /create by notes 110 (2026-10-13), see [[till-counter-signed-movements-notes-110]]:**
`proveWitness` is gone there (the cashier calls, the supervisor signs). Everything below still holds
for `/movements/change`.

`TillMovementService.proveWitness` ran BEFORE the movement's transaction (as decide-variance judges
`approverPin`), so a wrong PIN is journalled `PIN_FAIL` (tev_user_id = the supervisor, payload
`{action:'movement', kind, witnessBy}`) instead of rolling back. Order: session 404/409 → own-cashier
pickup 422 → inactive witness 422 `witnessBy` → `TILL_WITNESS_NO_PIN` 422 → `_PIN_REQUIRED` 403 →
`TILL_WITNESS_PIN` 403. No lockout exists anywhere (decide-variance has none). Migration
20261012140000 adds `tcm_witness_verified_on` (physically last) + `ck_tcm_witness_verified`.

**Why:** the user asked how a cashier knows a supervisor recorded a pickup without them. A TOP_UP now
defaults its witness to the session's cashier (it had none before). I decided, not the user: a top-up
the session's own cashier brings has no witness and wants no PIN (else a solo operator could never
bring change; a PICKUP has DROP as the one-person path, TOP_UP has none).

**How to apply:** one checker, `checkWitnessPin`, serves both routes; `/movements/change` (REV 2 §2.14)
takes `fromWitnessPin` (always) and `toWitnessPin` (unless the caller is the receiving cashier), added
the same evening at the user's go-ahead. /create PIN check went live on PM2 :3000 at 20:05 on
2026-10-12; the change PINs were built after that deploy. The movements suite sets tester1's and
prathap's PINs and restores them in afterAll. See [[till-variance-decision-notes-105]],
[[till-management-schema]].
