# Posting ledger map — Qt client plan

**For Viji (me). Written 2026-09-18, to be coded while Prathap builds
`plan-backend-ledger-map.md`.** Written against the client as it is on `main`
at `78f2dbe`. **No code yet — this is the plan.**

**The house rules in `../openingbal/plan-qt-opening-balance.md` §0 apply
unchanged** — own `.ui` on `NexDialog`, `NexTable` + a delegate emitting
`cellEdited`, width and visibility owned by `ui_tables`, every `ApiClient`
callback capturing a `QPointer`, never gating on `ctx.canCreate` directly.
Two of them bite here specifically and are called out in §4.

---

## 1. What the screen is for

Every posting engine turns a ROLE into a ledger through `acc_ledger_map`.
Today the only way to see or change that is SQL. A wrong mapping does not
fail — it posts money to the wrong account silently, and the first anyone
knows is a trial balance that will not explain itself.

So the screen has one job and it is not data entry: **make the mapping
legible, and make a wrong one hard to save.** Twenty-eight rows, set up once,
lived with for years.

## 2. The shape

ONE screen, no list page, no entry dialog. A `NexTable` of the roles with the
ledger editable in the row — because the whole set is 28 rows and an operator
comparing SALES against SALES_RETURN wants both on screen, not one modal at a
time.

```
Posting Ledger Map                                    [ 3 unmapped ]

ROLE                        LEDGER                      USED BY
────────────────────────────────────────────────────────────────────────
Discount allowed            Discount Allowed       […]  Receipt, Sale bill
Round off                   Round Off              […]  Receipt, Sale bill
TDS receivable              TDS Receivable         […]  Receipt
Write off                   Bad Debts Written Off  […]  Receipt
Sales                       Sales Account          […]  Sale bill
Output CGST                 — not mapped —         […]  Sale bill        ⚠
…

                                     [ Save ]  [ Close ]
```

- **ROLE is the server's label**, not the code. `BOUNCE_CHARGES_RECOVERED` is
  not something to make somebody read.
- **USED BY is why this screen is not a settings page.** It says which
  documents break if the row is wrong, and it is the column that turns an
  abstract list into a consequence.
- **An unmapped role is loud** — the count in the header, the em-dash in the
  cell, a warning tint on the row. That is the failure this screen exists to
  prevent.

## 3. Grids and ids

| | |
|---|---|
| grid | **106** `SETUP - POSTING LEDGER MAP` — already registered |
| ledger picker | **105** `POPUP - LEDGERS FOR ROLE` — already registered |
| roles popup | **104** `POPUP - POSTING ROLES` — registered, and probably NOT needed: the rows come from `GET /ledger-map/roles`, which is the whole list. Leave it alone rather than delete it. |
| ui_table | none yet — register one for the column widths, AFTER the columns are settled |
| menu | needs an id under Accounts setup |

**Check grid 106 and 105 actually have live columns before trusting them.** A
grid can carry SQL and zero live columns, answer 200, and render nothing —
that is exactly what bit grid 112 (notes 33), and 104/105/106 were registered
in the same era.

## 4. The two house rules that bite here

**The ledger picker must be filtered by `expectedLedgerType`.** `GET /roles`
says what type the role needs; grid 105 takes a parameter for it. Offering
every ledger and letting the server refuse is a worse screen than offering the
ones that can be right — and the server refusing is the backstop, not the
design.

**Width and visibility belong to `ui_tables`.** Seed the header text and the
hidden flag once, set no width and no resize mode, and register the layout
when the columns stop moving. Registering it early and then adding a column is
the index-shift trap from the receipt (`[[receipt-screen-revision]]`) —
`fixed.ui_table_columns` is keyed by column NUMBER, so inserting one mid-grid
silently applies every stored width to the column next door.

## 5. What it loads, and when

`GET /ledger-map/roles` on open. That is the whole screen — one call, no
paging, no search. Twenty-eight rows.

The ledger name comes back on the same call, so the grid paints without a
second round trip and without the client holding its own copy of the ledger
list.

## 6. What Save sends

One `POST /ledger-map/create` per CHANGED row, not per row. The screen tracks
which rows the operator touched and sends those; an untouched row is not
re-sent, because a create-or-update that rewrites all 28 makes every audit row
say everything changed on a day one thing did.

If several rows changed, they go one after another and the screen reports what
landed and what did not — there is no batch route and this is not frequent
enough to want one.

## 7. What it must refuse before the server does

- An unmapped role left unmapped is **allowed** to be saved: the screen is not
  a wizard, and half-finishing a setup is legitimate. It just stays loud.
- A ledger whose type does not fit the role — refuse locally, naming the type,
  because the picker should not have offered it (§4) and if it did, that is a
  bug worth seeing in the client rather than as a 400.
- Removing a mapping a deployed engine uses — the server refuses with the
  documents named (`plan-backend-ledger-map.md` §4.3); show its sentence.

## 8. Files

```
src/modules/accounts/master/ledger_map/
    ledger_map_entry.h / .cpp / .ui      NexDialog + QStackedWidget? NO — one page
    ledger_map_delegate.h / .cpp         the ledger cell, filtered by role type
    ledger_map_entity.h                  column enum + formats()
src/core/api/endpoints/accounts_end_points.h   namespace LedgerMap
```

`ReceiptRoles` in `receipt_entity.h` becomes redundant the day `GET /roles`
lands — the receipt's Type combo should offer what the SERVER says a receipt
may post, not a second hand-maintained list. **That is a follow-up, not part
of this screen**, but it is the reason the roles list is a route at all.

## 9. Test checklist

`tests/ledgermap/run.sh`, same arrangement as the others (link ninja's
objects, `QT_QPA_PLATFORM=offscreen`, live server).

1. The screen opens and paints all 28 roles.
2. An unmapped role paints as unmapped, and the header count matches.
3. The ledger picker on `DISCOUNT_ALLOWED` offers EXPENSE ledgers and not the
   bank.
4. Changing one row and saving sends ONE create, not 28.
5. A type mismatch is refused locally, with the cursor on the cell.
6. Deleting `DISCOUNT_ALLOWED` shows the server's refusal naming RECEIPT.
7. After a save, `GET /roles` agrees with the screen — read it back rather
   than trusting the 201.
8. Grid 106 **has live columns** (`NOT grid_column_is_deleted`), not just SQL.
