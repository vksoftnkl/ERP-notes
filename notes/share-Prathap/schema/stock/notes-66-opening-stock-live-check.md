# notes (66): Opening Stock (menu 44), live check 2026-09-30

The Qt screen was re-checked against the live server after the 2026-09-28 engine port.

`tests/openingstock/run-live.sh` covered:
- pick → item-lookup;
- save a draft → grid 99 → reopen → edit → re-save the same document → cancel the draft;
- save & post → lot resolved, holding +4, Stock Journal voucher (2530, 200/200) → reopen read-only → cancel.

Every check passed. The cancel reversed both the stock and the voucher. Grid 99 now carries company, branch and the date range, so the old gap is closed; thank you.

## 1. "Already opened" ignores the bucket (question)

The preflight's `already_opened` matches company, branch, year, **godown** and the **lot identity**. It does not match `sml_bucket`.

The message calls this a "holding". In the engine, a holding is godown × lot × bucket (stock_balance's grain, and the physical count's).

Consequence:
- A shop that opened 40 SALEABLE of an item and later keys the 5 crushed ones as a second opening in DAMAGED is refused: "this holding already has an opening in this year".
- Keying both on one document works, because `sml_src_doc_id <> keyed.svi_voucher_id` excludes it.

Found live: ZT-ITEM-A was opened SALEABLE in Coimbatore (OPN0067); a QUARANTINE opening for it was refused.

**Ask:** decide which is meant.
- (a) Add `AND sml.sml_bucket = keyed.svi_bucket`, so the rule is per holding as the message says.
- (b) Keep it per lot, and change the message to "this lot already has an opening in this godown this year". The fix for crushed stock is then an adjustment.

(a) matches the wording and the physical count. Either is fine for the screen.

## 2. `trackSignature` on /stock/opening/get lines (still open)

The loaded lines still carry no `trackSignature`. The screen re-runs /stock/opening/item-lookup once per distinct item after every load, only to learn which identity cells may open.

**Ask:** return it on each line, as item-lookup does. The screen then drops those calls.

## Verified live 2026-09-30 (after Prathap's fix)

1. Option (a) was taken: the rule is now per bucket. A QUARANTINE opening of ZT-ITEM-A in Coimbatore, where SALEABLE is already opened, validates clean. The probe draft was cancelled.
2. `trackSignature` is on every /stock/opening/get line, and on /stock/adjustment lines too. Both screens now read it from the line: the per-item item-lookup calls after a load are gone.

The live suite passes 26/26.
