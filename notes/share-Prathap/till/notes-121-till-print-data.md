# Notes 121: Till print data: what the Z and X prints still need (2026-10-15)

**STATUS:** DONE, live 2026-10-15 (Prathap deployed; verified by rendering previews): 12-hour times, variance_treatment, cash_expected_counted, safe_*. The three layouts were republished the same day for it (till_z_x_templates.py: tm() now cuts the hh:mm AM text — the old 24-hour rebuild printed 7:58 PM as 07:58 AM). Was: asked. Server only (`modules/till/services/till-print.service.ts`, providers `till.session.*` / `till.day.*`). The three layouts are built and published (`TillSessionZ80`, `TillSessionX80`, `TillDayZ80`, made by `Shared/Prathap/print/till_z_x_templates.py`). Each fix here lets a workaround in them go.

This gathers in one place the four gaps from notes 117's follow-up (10-14) and the safe figures notes 118 left for the print. `till-print.service.ts` has not changed since 10-14.

## 1 · Times in 12-hour form

`timeText()` sends `'DD-MM-YYYY HH:mm'`, which is 24-hour. The user's rule is AM/PM everywhere a person reads a time, and the designer's `date` transform cannot parse this text. Today the templates rebuild AM/PM by cutting the string apart.

**Ask:** send every time as `'DD-MM-YYYY hh:mm AM'` (IST), the same as the grids' `HH12:MI AM`. Or keep the field and add a `*_text` field beside it. This covers `opened_on`, `closed_on`, `received_on`, `printed_on`, the movements' `created_on`, and every other `timeText()` field.

## 2 · How the session's cash difference was booked

`till.session.header` has `variance_status` but not the treatment. The Z should say how the gap was booked.

**Ask:** add `variance_treatment`: the session's CLOSE-stage cash variance treatment (EXPENSE · RECOVER · SUSPENSE · WITHIN_TOLERANCE), null when there is no gap.

## 3 · The day's "cash expected" counts tills not yet counted

On an open day, `till.day.header` `cash_expected` sums every session's expected, including live ones that have no count. Example from 10-14: expected 40,480, counted 38,550, variance +70. The 2,000 gap was ZT1-261014-15, still open, so the day X looked 1,930 short.

**Ask:** add `cash_expected_counted`, the expected of the counted sessions only (what `cash_counted` and `cash_variance` compare against). Keep `cash_expected` as it is.

## 4 · The safe on the day Z (notes 118)

`days/get` already carries the day's safe figures (`safe.opening / expected / counted / remitted`, from `tbd_safe_*`). The day Z print data does not.

**Ask:** add to `till.day.header`:

| Field | From |
|---|---|
| `safe_opening` | `tbd_safe_opening`; null until the day closes |
| `safe_expected` | `tbd_safe_expected` |
| `safe_counted` | `tbd_safe_counted`; null when not counted that day |
| `safe_difference` | counted − expected, when counted |
| `safe_remitted` | `tbd_remitted` |
| `safe_in_transit` | the CIT remittances still IN_TRANSIT at print time (amount) |

## After this

I update the three layouts (12-hour times straight from the field, the treatment line on the session Z, the counted-only expected on the day X, a SAFE block on the day Z), re-publish them, and test-print from Business Day and the Cockpit.
