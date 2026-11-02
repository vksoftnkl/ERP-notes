# Opening stock — closing test plan

Drives every row of `stock.stock_track_preset` through the document's whole life.
Run each case in the UI; after **every** step run the checker and keep its output:

```
psql -h 192.168.0.106 -U claude -d ERP \
     -v refno="'OS/26-27/0001'" -v year="'2026-2027'" \
     -f 19t_verify_opening.sql
```

Anything not `PASS`/`SKIP` is a finding. `NOTE` is a known discrepancy, listed at the end.

---

## 0. Blockers — the run fails at step 1 without these

| # | Blocker | Why it stops the test | Fix |
|---|---|---|---|
| B1 | `ui_table` 27 has no column 38 (`Cost / Unit`) | `CostRate` is now derived and read-only, and the column that replaces it is hidden — **no cost can be keyed at all**, so every Post is refused with *"an inward with no cost rate"* | POST `grids/ui_table_27_cost_per_unit.json` |
| B2 | No sample item for 4 of the 8 presets | half the matrix is unrunnable | §1 |
| B3 | `fn_svh_cancel` raises on non-POSTED | if the route delegates to it, case **F2** (draft → cancel) fails with `restrict_violation` | confirm the route branches for DRAFT |

Grid 100 scoping (`supplier_popup_grid_100_scoping.json`) is **not** a blocker, but until
it is applied the supplier picker lists every company's suppliers — so in case P8 pick a
supplier that really belongs to the test company, or the lot gets a foreign one.

## 1. Sample items to create

One item per preset, so the matrix is one item wide. Set **Tracked as** on the item (or its
group) to the preset — that is what writes the `stock_track_policy` row.

Give each **two unit conversions** (e.g. `PCS` base, `BOX` = 12) — a factor of 1 cannot
catch the cost-conversion bug, which is the whole point of case C2.

| Preset | Sig | Existing item | To create |
|---|---|---|---|
| NONE | `N` | `001` (Chandra) | — |
| MRP_SELLING | `M` | — | **TEST M — MRP only** |
| SP_ONLY | `S` | — | **TEST S — selling price** |
| BATCH | `B` | — | **TEST B — batch only** |
| BATCH_EXPIRY | `BE` | `TEST OPENING ITEM 01` | — |
| FMCG_MRP | `BME` | `TEST OPENING ITEM 02` | — |
| PHARMA | `BMEP` | `Sakthi Manjal Powder` (Acme) | one on the test company |
| SERIAL | `R` | — | **TEST R — serial** |

Keep them in ONE company/branch so a single opening can carry all eight lines.

## 2. What each preset must show on the line

The delegate greys what the item does not track. Assert the greying, not just the save —
a cell that opens when it should not is how a value reaches a lot that discards it.

| Preset | Batch | Mfg/Expiry | MRP | Sale price | Serial | Supplier |
|---|---|---|---|---|---|---|
| NONE | grey | grey | grey | grey | grey | grey |
| MRP_SELLING | grey | grey | **open** | grey | grey | grey |
| SP_ONLY | grey | grey | grey | **open** | grey | grey |
| BATCH | **open** | grey | grey | grey | grey | grey |
| BATCH_EXPIRY | **open** | **open** | grey | grey | grey | grey |
| FMCG_MRP | **open** | **open** | **open** | grey | grey | grey |
| PHARMA | **open** | **open** | **open** | grey | grey | **open** |
| SERIAL | grey | grey | grey | grey | **open** | grey |

"Tracked by" must read the same facets in words. Checks 26/27 assert the lot agrees.

## 3. Flows

Each flow is run per preset. `→ verify` = run the checker.

- **F1 draft → post → cancel** — the main line.
  1. Key the line, **Save** → verify (10, 11 PASS: no ledger, no lot)
  2. F8, reopen the draft → the greying must survive the reload (this is where a
     reconstructed signature used to read `N` and lock the batch cell)
  3. **Save & Post** → verify (20–29)
  4. **Cancel**, pick a preset reason → verify (40–43)
  5. Re-open the same holding → must be **accepted** (43 proved it is free)
- **F2 draft → cancel** — no ledger row ever exists. 40 passes trivially, 41 nets 0,
  10/11 must STILL pass after the cancel. See blocker B3.
- **F3 draft → update → post** — reopen a saved draft, change qty and cost, Save again.
  Lines are a FULL REPLACE, so check 2 must still balance and no orphan
  `stock_voucher_item` rows may survive (`count(*)` = lines on screen).
- **F4 post → cancel → re-open → post** — proves a reversal really frees the holding
  and the second posting builds its own ledger rows rather than reusing the first's.

## 4. Cases that are not about tracking

| # | Case | Expect |
|---|---|---|
| C1 | Cost keyed against `BOX` (factor 12), 240 | line cost rate **20**, value = base qty × 20. Check 23. The bug it guards: 240 left per-piece values the line twelvefold |
| C2 | Same item, switch the line to `PCS` | the base rate must not change meaning — re-key and confirm |
| C3 | Free qty only, qty 0 | line saves; value counts the free base qty (`svi_value` is generated over `base + free`) |
| C4 | Split: one line, three batches | 3 ledger rows, 1 distinct `svi_line_no`. Check 3 reports `NOTE` — see §6 |
| C5 | Two lines, same item, two godowns | two holdings, both accepted |
| C6 | Same holding twice in ONE document | `/validate` must refuse before Post |
| C7 | Second opening for a holding already opened | refused: *"already has an opening in this year"* |
| C8 | Tracked item, batch left blank | `/validate` refuses. **PHARMA with no supplier does NOT** — see §6 |
| C9 | SERIAL preset, qty 2 on one serial | the preset is `BLOCK` on negative; confirm what one serial + qty 2 does |
| C10 | Expiry typed `31-03-2027` | reaches the wire as `2027-03-31`; reopened it reads `31-03-2027` |
| C11 | Expiry typed `31-02-2027` | refused at save by name, not silently dropped |
| C12 | Post, then cancel after selling from the stock | must fail 409 — the fix is an adjustment, not a retry |

## 5. Screen-state cases

| # | Case | Expect |
|---|---|---|
| S1 | Save | message names lines still missing a tracked facet; screen clears; **godown, date and rate source carry forward** |
| S2 | Post | clears; POSTED badge is **green, row not dimmed** in the list |
| S3 | Cancel | clears |
| S4 | New | blank *including* the godown |
| S5 | Post refused by `/validate` | the grid must still be **editable** — this regressed once |
| S6 | Open a POSTED voucher from the list with **Enter** | opens read-only; lines readable and scrollable |
| S7 | Pick an item with no godown chosen | refused, cursor to the godown |
| S8 | Choose the godown after keying lines | blank line godowns back-fill |

## 6. Known discrepancies — expect these, they are not new findings

1. **`svh_line_count` counts ROWS, not lines.** `fn_svh_recompute` is `COUNT(*)` over
   `stock_voucher_item`, so a 1-line/3-split document reports 3. The client and
   `grids/opening_stock_list_grid.sql` both describe it as a count of LINES. Check 3
   reports `NOTE`. Decide which is right and make the other agree.
2. **`/validate` does not check supplier or sale price**, though both are tracked facets.
   A `BMEP` line with no supplier posts, and `fn_slt_resolve` normalises the blank to the
   zero-uuid sentinel — check **27** is what catches it. Same for `S`.
3. **`fn_svh_post` enforces none of the identity rules itself** — quantity, duplicate
   holding and cost only. `/validate` is a client-called guard, so `/stock/opening/import`
   bypasses all of it.
4. **No `trackSignature` on `OpeningStockLineDto`**, so the screen re-resolves it per
   distinct item after every repaint.
