# Notes 90 — Temp credit has no history of its own

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-05 · **For:** Prathap
**Client side:** done, no server change needed for it (below). This note asks for the part only the server can do.

## What the user saw

Temp Credits (menu 257) → History (Ctrl+H) always said "No history is recorded for this document".

**Why:** History reads `public.txn_status_log` by the row's id. A temp credit's id (`atc_id`) is never written there
(0 rows, no `TEMP%` doc type). The bill's own steps are logged under the **bill** id, the follow-ups go to
`audit.audit_log`, and the money goes to `acc_bill_adjustment`. Nothing joined them.

## What the client now does (live, grid 132)

New grid **132 "POPUP - TEMP CREDIT HISTORY"** (`grids/temp_credit_history_grid.{sql,json}`). Params:
`iatc_id` / `iacc_year` / `icompany_id`. Sort column: `th_sort`. It is one UNION ALL over the four places the
story lives, oldest first:

| th_kind  | from | event |
|----------|------|-------|
| BILL     | `txn_status_log` by `atc_src_doc_id` | Bill created / posted / retendered / cancelled |
| CREDIT   | `acc_temp_credit` itself | Credit given (name, mobile, days, due) |
| FOLLOWUP | `audit_log` (`acc_temp_credit`, `log_pk = atc_id`) | Follow-up (remark, promise date) |
| SETTLE   | `acc_bill_adjustment` on `atc_abl_id` | Received / Written off / Credit note / Advance set-off / Reversed |
| COUNTER  | the same, where `abj_tender_id IS NOT NULL` | Paid with the bill (the bill's own cash; the balance row is the whole bill) |

Nothing to do for this. It is here so the SQL has an owner if a table changes.

## The ask

### A. Log the temp credit's own life in `txn_status_log`

Status changes are **silent** today:
- `bill-lifecycle.service.ts` ~1396 creates it OPEN, and ~1790 sets it CANCELLED. `bill-retender.service.ts` ~201 sets it CANCELLED.
- `bill-balance-recompute.service.ts` ~276 moves it OPEN → PARTIAL → SETTLED (and back on a reversal) with a raw `UPDATE`, without logging anything.

Please call `appendTxnStatusLog` for each, with a new doc type **`TEMP_CREDIT`** keyed by `atc_id` and the event
`CREATED` / `PARTIAL` / `SETTLED` / `REOPENED` (when a reversal moves it back) / `CANCELLED`. The from → to status,
the amount in remarks ("received 10.00, balance 200.00"), and the receipt refno where there is one.

The recompute `UPDATE` would need `RETURNING t.atc_id, old status, new status` (or a before-read) to know which
rows actually changed. Log only real transitions, not every recompute.

Once this exists, grid 132 gains a fifth branch for these rows. The client needs no change beyond the SQL.

### B. Audit the create

`audit_log` has only `update` rows for `acc_temp_credit` (the follow-ups). The create (name / mobile / place / days)
has no `insert` row, so an edit of the WHO before the bill is posted is untraceable. Please write an `insert` audit
row with `log_entity_id = atc_id` when the bill posts it. `log_entity_id` is NULL on the follow-up rows too
(same gap as notes 80).

### C. Question: WRITTEN_OFF is never set

The recompute keeps `WRITTEN_OFF` if it is already there, but no code path sets it. A write-off of the bill's balance
drops pending to 0, so the credit becomes **SETTLED**. Is that intended? If not, set WRITTEN_OFF when the
last movement that zeroed the bill was a `WRITEOFF` adjustment. The list's status filter already offers it.
