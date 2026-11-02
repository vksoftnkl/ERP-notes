# Plan — Transaction audit: what every save changed (header + lines)

**Date:** 2026-10-03 · **Owner:** Qt client (Viji / Claude) + Prathap (server) · **Mockup:** `txn_audit_history_mockup.png`
**Server ask:** `notes-89-txn-line-audit.md` (this folder)

## 1. Goal

An admin opens **History (Ctrl+H)** on any transaction and sees, **per save**: who, when, which
counter, which header fields changed (before → after), and which lines were **added**, **dropped**
or **changed** (field by field). They can also open the document **as it stood at any revision**,
or compare any two revisions.

Example (the user's): opening stock saved with 3 items; later lines are changed, dropped and added,
and the admin must see exactly that.

## 2. What exists today (live, checked 2026-10-03)

| Piece | State |
|---|---|
| `txn_status_log` + grid 126 + `TxnHistoryDialog` (Ctrl+H on every txn list, opening, physical) | status steps only: CREATED / EDITED / POSTED / CANCELLED… with no content |
| `audit.audit_log` header row per save | `log_action = update` with `log_changed_fields = NULL` (no from/to on edits) |
| `audit.audit_log` line row per save, stock vouchers | `{svhId, lineCount}`, *"Opening stock lines replaced"*: **no line content** (`stock-voucher.service.ts` ~1293, logged as `insert` because the old lines are never read back) |
| sale bill / quotation lines | full line on insert, nothing on edit |
| line save model | stock vouchers: **delete all + re-insert** (new ids every save). Bill / quotation: upsert by `sbiId` / `sqiId` |
| client line ids | every screen **keeps** the line id in a hidden grid column on load (`Cols::Id`, `SalesItemCols::Id`), but **no screen sends it back** |
| `/audit-log/list?screen_id&record_pk` | already returns `log_original_record` + `log_modified_record`, so **no new route is needed** |
| `audit.audit_screen` | 18 `transaction` screens registered (67 Opening Stock, 85 Sale Bill, …); `screen_audit_sql` snapshot templates are **disabled** in code |

## 3. Design

### 3.1 One audit row per save = full BEFORE and AFTER document

Inside the save transaction, the service:
1. reads the document **as it is now** (before touching anything), in the **same shape as its own
   `/get` response** (header + `items[]`, with names: item, unit, godown);
2. saves as today;
3. reads it again (after);
4. writes **one** `audit.audit_log` row:

| Column | Value |
|---|---|
| `log_action` | `insert` for create, `update` for edit, `cancel` / `delete` as today |
| `log_table_name` | the header table (`stock_voucher`, `sale_bill`, …) |
| `log_pk` and `log_entity_id` | the document id |
| `log_original_record` | BEFORE (NULL on create) |
| `log_modified_record` | AFTER |
| `log_changed_fields` | header fields only, `{field: {from, to}}`, as masters do (cheap to compute; lets the revisions list show "hdr 2") |
| `log_notes` | `rev N` (or a `log_rev_no` column; see notes 89 Q1) |
| `log_device_name` | the counter / device (mockup shows it) |

The separate per-line audit rows ("lines replaced", "Bill item created") **stop**. The single row
carries everything.

**Why the `/get` shape:** *View at Rev N* can fill the real form straight from the snapshot. That
reuses the screen's own load code, not a second renderer.

**Why snapshots, not stored diffs:** one generic helper covers all 18 transaction screens with no
per-screen field lists. It also gives "document as at revision N" (needed for posted amendments).
Saving stays fast because the diff runs only when History opens. Storage is small: jsonb is
TOAST-compressed, and a 500-line document saved 10 times is a few hundred KB.

### 3.2 Stable line ids

The diff matches a line to itself by id:
- **Client:** every save sends each existing line's id (`sviId`, `sbiId`, `sqiId`, `soiId`, `sdiId`, …).
  New lines send none.
- **Server:** a line with an id **updates** that row; a line without one **inserts**; ids missing from
  the payload are **deleted**. Bill and quotation already work this way. Stock vouchers (opening,
  physical, adjustment, transfer) move from delete-all + re-insert to this upsert.
- **Fallback** (old data, or a screen not yet sending ids): the client diff matches unmatched lines on
  item + batch/lot + godown (+ bucket), then pairs any remaining ones by line number.

### 3.3 Client — `TxnHistoryDialog` grows into the mockup

- **Left: revisions.** `txn_status_log` steps (grid 126, as now) merged with the audit rows from
  `/audit-log/list?screen_id=<audit screen>&record_pk=<doc id>`, joined on revision / time. Each row
  shows rev, when, user, device, event pill, and change counts (+ − ~ hdr).
- **Right: "Rev N — what changed".**
  - Filter chips (All / Added / Dropped / Changed / Header) and a *show unchanged* toggle.
  - Header table: field, before (struck through), after.
  - Lines table: + ADDED (green), − DROPPED (red, struck through), ~ CHANGED (amber; only the changed
    cells highlighted, `40 → 36`).
  - Totals strip: lines, qty, value, net change.
- **View at Rev N** opens the screen's own entry form **read-only**, filled from that AFTER snapshot.
- **Compare…** diffs any two revisions (e.g. Rev 1 against latest).
- **Export** writes the shown changes to CSV / PDF.
- **One shared diff engine** (`shared/ui/txn/txn_doc_diff.{h,cpp}`), driven by a small per-doc-type
  descriptor in `TxnDocTypes`: the line array key, the line id key, the fallback match keys, the
  columns to show, and the header fields to ignore (timestamps, sync dates, `*_modified_on`).

### 3.4 Order of work

| Step | Who | What | Done when |
|---|---|---|---|
| 1 | Prathap | snapshot helper + one-row-per-save in **opening stock** (screen 67) + stable `sviId` upsert | an edit of an OS doc writes one row with before and after |
| 2 | Qt | opening stock sends `sviId`; diff engine; History right panel; View at Rev | the mockup works on opening stock, live |
| 3 | Prathap | same helper in the other 17 transaction services | each writes before and after |
| 4 | Qt | send line ids on every screen; descriptors per doc type | History shows changes on all |
| 5 | both | tests: `tests/txnaudit` (live: create 3 lines → edit: change 1, drop 1, add 1 → History shows +1 −1 ~1) | green |

Opening stock first: it's the user's example, it has the "lines replaced" gap, and it's the smallest
line model.

## 4. Out of scope / later

- Thinning old snapshots to diffs (only if storage ever matters).
- Field-level audit of GST portal calls (`gst_api_log` already covers them).
- Masters: they already have their own audit (`AuditHistoryDialog`), unchanged.
