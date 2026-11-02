# Posting ledger map — backend plan (NestJS)

**For Prathap. Written 2026-09-18.** Read off the schema at 192.168.0.106/ERP
and the conventions the accounts routes already follow. That box is a TEST
database: its schema is authoritative, its rows are not.

**The whole module in one line:** every posting engine turns a ROLE
(`DISCOUNT_ALLOWED`, `OUTPUT_CGST`, `ROUND_OFF`, …) into a ledger through
`accounts.acc_ledger_map`, and there is no way to see or change that mapping
except SQL on the box.

---

## 1. What exists, what does not

| | state |
|---|---|
| `accounts.acc_ledger_map` | deployed, **28 rows, every role mapped** |
| scope of those rows | ALL shared — `alm_company_id` and `alm_branch_id` are NULL on all 28 |
| the ledgers they point at | all shared too (`led_company_id` NULL on all 28) |
| companies on the box | 5 |
| routes | **NONE.** Nothing under `/ledger-map`, `/posting-*` or anything like it |
| client | none |
| grids | 104 POPUP - POSTING ROLES, 105 POPUP - LEDGERS FOR ROLE, 106 SETUP - POSTING LEDGER MAP — **already registered**, unused |

So nothing is broken today: the seed (`26_ledger_map_seed.sql`) filled every
role and posting works. What is missing is any way to CHANGE it, and that
matters because a wrong mapping does not fail — it posts money to the wrong
account and says nothing.

**Scope decision, taken by the user 2026-09-18: the SHARED set only.** One
ledger per role, the same for every company. `alm_company_id` / `alm_branch_id`
stay NULL and the per-company override the unique index allows stays in the
schema, unbuilt, until somebody actually needs it. **Do not build the override
now** — §7.

---

## 2. Schema prerequisites

**None.** The table is deployed and correctly shaped. Two things worth knowing
before you write against it:

```
ux_alm_role UNIQUE (alm_company_id, alm_branch_id, alm_role, alm_supply_nature)
            NULLS NOT DISTINCT WHERE alm_is_deleted = false
```

`NULLS NOT DISTINCT` is what makes the shared row unique — without it Postgres
would allow any number of rows with NULL company. Keep it.

`alm_supply_nature` (varchar 5) is NULL on all 28 rows today. It exists for the
GST roles, where INTRA and INTER resolve to different ledgers. **It is out of
scope here** — carry it, never set it, and let the unique index keep the NULL
row unique.

### 2.1 One guard that does not exist and should

There is no CHECK and no trigger stopping a role being mapped to a ledger that
cannot hold it — `DISCOUNT_ALLOWED` pointed at a bank account is accepted
today. That is the single most valuable thing this module can add, and it
belongs in the SERVICE rather than the schema because the rule is per-role:

| role family | the ledger must be |
|---|---|
| `INPUT_*`, `OUTPUT_*`, `TDS_*`, `TCS_*` | ledger type `TAX` |
| `SALES`, `SALES_RETURN`, `INTEREST_INCOME`, `SURCHARGE_RECOVERED`, `BOUNCE_CHARGES_RECOVERED` | `INCOME` |
| `PURCHASE`, `PURCHASE_RETURN`, `DISCOUNT_ALLOWED`, `WRITE_OFF`, `BANK_CHARGES`, `CLAIMS_ALLOWED` | `EXPENSE` |
| `ROUND_OFF`, `OPENING_DIFFERENCE`, `RETAINED_EARNINGS` | `GENERAL` or `EXPENSE` — a round-off goes either way |
| `ADVANCE_RECEIVED` | `GENERAL` (it is a liability head, not a party) |

Refuse with the ledger's own type named, the way
`/account-ledger-masters/create` already refuses a bad `ledLedgerType`. A
mapping is set up once and lived with for years; catching it at the moment it
is typed is worth far more than catching it at the first posting.

---

## 3. Module layout

`src/modules/accounts/ledger-map/` beside `receipts/`, same shape: controller,
service, dto/. Nothing new to learn.

---

## 4. The endpoints

Three. All under `/api/v1/ledger-map`.

### 4.1 `GET /ledger-map/roles`

Every role the product knows, whether mapped or not — because the screen's job
is to show a role with NOTHING against it just as loudly as one with a ledger.

```jsonc
{ "data": [
  { "role": "DISCOUNT_ALLOWED",
    "label": "Discount allowed",              // for the screen; see below
    "expectedLedgerType": "EXPENSE",          // drives §2.1 and the picker's filter
    "usedBy": ["RECEIPT", "SALE_BILL"],       // which documents post it
    "almId": "01a0…",                         // null when unmapped
    "ledgerId": "01a0…",                      // null when unmapped
    "ledgerName": "Discount Allowed",
    "isActive": true,
    "remarks": null }
] }
```

**The role list is the SERVER's, not the client's.** Today the client has its
own copy in `ReceiptRoles` and the receipt screen offers eight of them; that
list and yours have already disagreed once (`ROUND_OFF` is in
`acc_ledger_map` and in the client, and `/receipts/post` rejects it — notes
34). One list, served, ends that class of bug.

`label` is yours too: an operator should not have to read `BOUNCE_CHARGES_RECOVERED`.

### 4.2 `POST /ledger-map/create`

Create-or-update, the house way — the presence of `almId` selects.

```jsonc
{ "almId": "01a0…",            // omit to create
  "role": "DISCOUNT_ALLOWED",
  "ledgerId": "01a0…",
  "isActive": true,
  "remarks": null }
```

- `alm_company_id` / `alm_branch_id` / `alm_supply_nature` are **not accepted**
  and are written NULL. Sending one is a 400 naming the field — the shape is
  deliberately narrow so the day we DO add overrides, nothing already depends
  on a half-built one.
- Refuses an unknown role (name the list), a ledger that does not exist, a
  deleted or inactive ledger, and a ledger whose type does not fit §2.1.
- A second row for a role that is already mapped is the same 409 shape as
  everywhere else, not a silent second row — `ux_alm_role` will refuse it
  anyway, so catch it and say which role.

### 4.3 `DELETE /ledger-map/delete?almId=`

Soft. **And it must refuse when the role is one a deployed engine resolves**,
because an unmapped role is not a blank screen, it is a posting that fails at
the moment money is being taken. Refuse with the documents that use it named:

> `DISCOUNT_ALLOWED is used by RECEIPT and SALE_BILL. Point it at a different
> ledger instead of removing it.`

Unmapping a role nothing posts yet (`PURCHASE_RETURN`, say) is fine.

---

## 5. Service logic

Thin. The only real work is §2.1's type check and §4.3's in-use check.

**One thing to get right:** the roles list in §4.1 is the union of what the
engines resolve, NOT `SELECT DISTINCT alm_role FROM acc_ledger_map`. A role
with no row is exactly the case the screen exists to show.

---

## 6. Rules the DATABASE enforces

- `ux_alm_role` — one row per (company, branch, role, supply nature), NULLs
  not distinct, soft-deleted rows excluded.
- `alm_ledger_id` NOT NULL — a mapping without a ledger is not a mapping.

Everything else is the service's.

---

## 7. Do not build

- **Per-company and per-branch overrides.** Taken 2026-09-18: the shared set
  only. The schema keeps the door open; the API does not open it yet. When it
  does, the resolution order the index implies is branch → company → shared,
  and `GET /roles` grows a `from` field saying which level answered.
- **`alm_supply_nature`.** Carried, never set.
- A grid/list route: grid 106 already exists and the client reads it through
  `/configured-grid-sql/run` like every other list.
- Any change to how the ENGINES resolve a role. They read the table today and
  will read the same table after this.

---

## 8. Test checklist

1. `GET /roles` returns every role, including ones with no row, with
   `ledgerId` null on those.
2. Create a mapping for an unmapped role → 201, and it appears on `/roles`.
3. Re-point `DISCOUNT_ALLOWED` at another EXPENSE ledger → 201, one row still.
4. Point `DISCOUNT_ALLOWED` at a BANK ledger → **400 naming the type**.
5. Point `OUTPUT_CGST` at an INCOME ledger → 400.
6. Send `almCompanyId` → 400, "not accepted".
7. Delete `DISCOUNT_ALLOWED` → **409 naming RECEIPT**.
8. Delete a role no engine posts → 200, and `/roles` shows it unmapped.
9. After every one of the above, `SELECT count(*) FROM acc_ledger_map WHERE
   NOT alm_is_deleted` is 28 or 28±the rows the test meant to change — no
   duplicates, no orphans.
