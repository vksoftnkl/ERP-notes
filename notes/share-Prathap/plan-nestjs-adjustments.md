# NestJS — party-balance endpoint + bill-adjustment posting

Target repo: `/home/vk/Projects/ERP/ERP-Server` (NestJS + Prisma, PostgreSQL 18).

## Context

The Qt client now ships open-credit adjustment on the sale bill: an operator can
set off an order advance or a sale-return credit note against an invoice, from
inside the settle dialog or from F4 outside it. It calls two things the server
does not have:

- `GET /transactions/party-balance?partyId&companyId&type=CR|DR` — the credits
  the party holds. **Missing** → the panel degrades to an empty grid, so the
  client ships but the feature is unusable.
- `"adjustments": [...]` nested in the sale bill's save payload → the pair of
  `accounts.acc_bill_adjustment` rows.

What ERP-Server has today: masters, lookups, grid infrastructure, auth, audit.
**No transaction-document endpoints at all** — no `/bills`, `/quotations`,
`/sale-orders`, `/transactions`. `tender-masters` and `tender-type-masters` are
in; `master-lookups/party-credit` (which the client also calls) is not.

**Scope of this plan (decided):** the adjustment slice only — the read endpoint,
plus a `BillAdjustmentService` written to be called from inside a transaction so
`/bills/create` can invoke it unchanged when it lands. The bill-posting pipeline
itself is a separate plan.

**Database state (decided):** the tables exist in the live DB from
`/home/vk/Shared/Prathap/schema/*.sql`; Prisma does not know them. So: models,
no migrations.

---

## 0. The blocker to settle first — customer → ledger

`acc_bill_balance.abl_party_id` is a FK to `accounts.acc_ledger_master.led_id`.
The client holds `sales.customers.cus_id` and sends that as `partyId`.

**There is no bridge between the two.** `prisma/sales/customer.prisma` has no
ledger column; `accountLedgerMasters.prisma` has no party/ref column. Grepping
the whole Prisma folder for either direction returns nothing. Both the SQL
(`grids/adjust_open_credits.sql` §1) and the payload doc (§6.1) flag this — it is
still open, and nothing can be returned until it is closed.

**Recommendation: add `cus_ledger_id uuid` to `sales.customers`**, FK to
`acc_ledger_master(led_id)`, set when the customer's control ledger is created.
One join, indexable, and the same resolution then fills `tdPartyLedgerId`, which
`SaleBillEntry::buildTenderPayload()` currently sends as null with a TODO for
exactly this reason. The reverse shape (`led_party_id` + `led_party_type` on the
ledger) also works but makes the hot path a scan of the ledger master.

Until it exists, the endpoint can be built and unit-tested against a ledger id
passed directly; wire the resolution in one place (§2.3) so there is a single
line to change.

---

## 1. Prisma models — no migration

`prisma/schema.prisma` is **auto-generated** by `scripts/build-prisma-schema.js`
from the per-folder files. Never edit it; add sources and rebuild.

New files:

| File | Model | Table |
|---|---|---|
| `prisma/accounts/AccountBillBalance.prisma` | `AccBillBalance` | `accounts.acc_bill_balance` |
| `prisma/accounts/AccountBillAdjustment.prisma` | `AccBillAdjustment` | `accounts.acc_bill_adjustment` |

Follow `AccountVoucherBills.prisma` for style: camelCase fields with
`@map("snake_case")`, `@default(dbgenerated("uuidv7()"))`, `@@map`, `@@schema("accounts")`.

Four things that differ from the neighbouring models and matter:

1. **`acc_bill_balance` is NOT partitioned** — plain `@id` on `ablId`. That is
   deliberate in the schema: an unpartitioned bill table is what makes
   cross-year settlement a plain foreign key.
2. **`acc_bill_adjustment` IS partitioned** by `abj_acc_year` with a composite
   PK → `@@id([abjId, abjAccYear])`.
3. **Generated columns.** `abl_pending_amount` and `abl_status` are
   `GENERATED ... STORED`. Prisma has no concept for them and will happily try to
   INSERT them. Map them as read fields and **never include them in a
   `create`/`update`**; add a comment saying so. (This slice only reads
   `acc_bill_balance`, so the risk lands on whoever writes `/bills/create`.)
4. **No Prisma enums.** `AccVoucherBill` declares `VoucherBillDrCr` etc., but the
   house rule for these tables is `varchar + CHECK, no enum types`, and
   `abj_dr_cr` is `character(2)`. Use `String @db.Char(2)` / `String @db.VarChar(20)`
   with a TS union type (`type BillDrCr = 'DR' | 'CR'`) rather than a Prisma enum
   — a Prisma enum over a CHECK column silently breaks the moment a value is added.

Then:

```bash
node scripts/build-prisma-schema.js && npx prisma generate
npx prisma db pull --print | diff -  # sanity-check the models against the live DB
```

---

## 2. `GET /transactions/party-balance`

New module, standard shape (`master-lookup` and `tenderMaster` are the templates):

```
src/modules/transactions/party-balance/
  party-balance.module.ts
  party-balance.controller.ts
  party-balance.service.ts
  party-balance-exception.filter.ts
  dto/list-party-balance-query.dto.ts
  dto/party-balance-response.dto.ts
  types/party-balance-api.types.ts
```

### 2.1 Controller

`@Controller('transactions')`, `@Version('1')`, `@ApiTags('Transactions')`,
`@ApiBearerAuth('access-token')`, method `@Get('party-balance')`.

Returns the house envelope:

```ts
return { success: true, message: 'Party adjustable credits fetched successfully', data };
```

**Do not add `@CacheTTL`.** `master-lookups` caches for a day because masters
barely move; these are live balances that another counter can spend mid-sale.
A cached credit list is how two tills adjust the same 2,000 twice.

### 2.2 Query DTO

```ts
export enum PartyBalanceType { CR = 'CR', DR = 'DR' }

export class ListPartyBalanceQueryDto {
  @IsUUID() partyId!: string;
  @IsUUID() companyId!: string;
  @IsEnum(PartyBalanceType) type: PartyBalanceType = PartyBalanceType.CR;
}
```

No `branchId`, no `accYear` — deliberately. Outstanding is company-wide (an
advance taken at one branch must be adjustable at another), and
`acc_bill_balance` is unpartitioned so a March credit settles an April invoice.
Reject them if sent rather than silently ignoring.

### 2.3 Service

Not a configured grid — there is no grid id for this list, so it does not go
through `ConfiguredGridSqlService`. A parameterised `$queryRaw` against
`acc_bill_balance`, which is `grids/adjust_open_credits.sql` §1 verbatim:

```
WHERE abl_party_id = $ledgerId AND abl_company_id = $companyId
  AND abl_is_deleted = false AND abl_is_active = true
  AND abl_dr_cr = $type                                  -- 'CR' | 'DR'
  AND abl_bill_type IN (...)                             -- see below
  AND abl_pending_amount > 0
ORDER BY abl_doc_date, abl_doc_refno                      -- FIFO, stable tie-break
```

`abl_bill_type` by direction: `CR` → `('ADVANCE','SALES_RETURN')`; `DR` →
`('SALES')`. `OPENING` and `JOURNAL` are deliberately not offered — which of
`ADVANCE_ADJUST`/`NOTE_ADJUST` an opening credit should post as is an accounting
decision, not one to guess.

**Resolve the ledger here, in one place** (§0): `cus_id → cus_ledger_id`, 404 if
the customer has no control ledger. This is the single line that changes when
the bridge lands.

Response rows, camelCase — these key names are what the Qt client reads, so they
are contract:

```
billId · billAccYear · billType · docRefno · docDate · billAmount
pendingAmount · status · srcModule · srcDocType · srcDocId · srcAccYear
narration · adjType · settlementMode
```

`adjType`/`settlementMode` are **derived in the SELECT** from `abl_bill_type`
(the same CASE the INSERT uses) so the client never computes them:

```sql
CASE abl_bill_type WHEN 'ADVANCE' THEN 'ADVANCE_ADJUST'
                   WHEN 'SALES_RETURN' THEN 'NOTE_ADJUST' END AS "adjType"
```

`pendingAmount` is the generated column — return it as-is. The client no longer
computes a balance and must not have to.

Return `numeric` as a JS number (not Prisma `Decimal`), and dates as ISO
`yyyy-MM-dd` — `NexJson::date` reads the first 10 chars, so a full timestamp is
tolerated but pointless.

---

## 3. `BillAdjustmentService.postAdjustments()`

```
src/modules/accountsModule/billAdjustment/
  bill-adjustment.module.ts
  bill-adjustment.service.ts
  dto/bill-adjustment.dto.ts
  types/bill-adjustment.types.ts
```

Follows the established composable-service idiom in this repo — the same
`Prisma.TransactionClient | PrismaService` union `TenderMasterService` and
`LedgerShippingAddressService` already use, so `/bills/create` can call it inside
its own `$transaction` with no refactor:

```ts
type BillAdjustmentWriteClient = Prisma.TransactionClient | PrismaService;

export interface BillAdjustmentContext {
  billId: string;        // the invoice's acc_bill_balance.abl_id — created by the caller
  companyId: string;
  branchId: string;      // the branch that SETTLED it
  tenantId?: string;
  accYear: string;       // FY of the ADJUSTMENT — the bill's year, never the credit's
  partyId: string;       // ledger id
  billDate: Date;
  userId: string;
  sessionId?: string;
  createdBy: string;
}

async postAdjustments(
  client: BillAdjustmentWriteClient,
  ctx: BillAdjustmentContext,
  entries: BillAdjustmentDto[],
): Promise<void>
```

`BillAdjustmentDto`: `againstBillId` (uuid), `amount` (number > 0),
`remarks?`. The client also echoes `billType`/`adjType`/`settlementMode` and
`againstBillAccYear` — **accept and ignore them for posting**. They exist so a
stored payload can be reconciled against the table without a join; deriving
instead of trusting is what stops a mislabelled advance posting as a note.

### Steps

1. **Skip empties.** `amount <= 0` never reaches the insert — `ck_abj_amount`
   refuses zero anyway.
2. **Lock and re-read each credit** in the same transaction:
   `SELECT ... FROM accounts.acc_bill_balance WHERE abl_id = ANY($ids) FOR UPDATE`,
   re-applying every predicate §2.3 used plus `abl_pending_amount >= amount`.
   The row lock is what serialises two tills racing for the same credit; without
   it they both pass the check and the loser dies later on `ck_abl_settled`,
   which is a 500 instead of a clean 409.
3. **Derive** per credit — never from the payload:

   | `abl_bill_type` | `abj_adj_type` | `abj_settlement_mode` | `abj_settlement_ledger_id` |
   |---|---|---|---|
   | `ADVANCE` | `ADVANCE_ADJUST` | `ADVANCE` | `sale_order.so_advance_ledger_id` |
   | `SALES_RETURN` | `NOTE_ADJUST` | `CREDIT_NOTE` | NULL (no column for it yet) |

4. **Insert two rows per entry** — the invoice side `CR`, the credit side `DR`,
   pointing at each other through `abj_against_bill_id`. Two rows because each
   row moves exactly one balance: `fn_abj_refresh_balance()` recomputes
   `NEW.abj_bill_id` and never touches the against-bill.
   `abj_row_no` = ordinal per `abj_bill_id`.
5. **Stop.** Do not update `abl_alloc_amount` — it is trigger-maintained and
   *recomputed*, not incremented.

`grids/adjust_open_credits.sql` §2 is the whole insert as one statement
(`jsonb_to_recordset` + `CROSS JOIN LATERAL (VALUES …) AS side`). Either run it
as `$executeRaw` — least drift from the reviewed SQL, and it re-validates in the
same statement — or build the pairs in TS and `createMany`. **Prefer the raw
statement**; the derivation and the re-check then live in one place that the
backend owner already signed off.

### Errors

Per-module exception filter, as every module here has:

- credit not found / not open / spent since → `409 Conflict`, naming the
  document and its current balance so the client can say which one.
- `ck_abl_settled` violation → also `409`, not a 500.
- customer has no control ledger → `422`.

### Explicitly not in this service

The **`ALLOCATION`** rows. Cash/UPI/card tenders also write
`acc_bill_adjustment`, but with `abj_adj_type = 'ALLOCATION'`, no against-bill,
and `abj_tender_id` set. That belongs to bill posting, next to the
`acc_tender_detail` insert. Two rules for whoever writes it:

- **A `CREDIT` tender must not produce an `ALLOCATION` row.** It is not money
  received; it is the declaration that the remainder stays receivable — which
  the `acc_bill_balance` row already represents. Allocating it would settle the
  debt with the debt and zero out a real outstanding.
- The `ADJUST` tender type (7) never appears in `tenders[]` at all; the client
  drops it. Its money is the `adjustments` array.

---

## 4. Wiring

- `PartyBalanceModule` and `BillAdjustmentModule` into `src/app.module.ts`
  imports, alphabetically near the other accounts modules.
- `BillAdjustmentModule` `export`s its service so the future `BillsModule` can
  import it.
- Both controllers under the existing global `v1` versioning + auth guard, so the
  path resolves as `/api/v1/transactions/party-balance` — which is what
  `AppConfig::baseUrl` + the client's bare constant produce.

---

## 5. Tests

Unit (`*.service.spec.ts`, mirroring the existing specs):

- the derivation table — ADVANCE → ADVANCE_ADJUST/ADVANCE, SALES_RETURN →
  NOTE_ADJUST/CREDIT_NOTE.
- payload-supplied `adjType`/`settlementMode` are ignored: feed a deliberately
  wrong pair and assert the derived values are written.
- `amount > pendingAmount` → 409, nothing inserted.
- `type=CR` excludes `SALES`; `type=DR` excludes `ADVANCE`/`SALES_RETURN`.

Integration — run the worked example end to end:

> bill 5,000; advance 2,000 adjusted; cash 500 + UPI 1,000 + credit 1,500.
> Assert: 2 `ADVANCE_ADJUST` rows; invoice `abl_pending_amount` 3,000 after the
> adjustment alone (5,000 − 2,000) and 1,500 once the two ALLOCATION rows land;
> advance `abl_pending_amount` 0, `abl_status` CLOSED; and **no ALLOCATION row
> for the credit tender**.

Concurrency: two parallel `postAdjustments` for the same credit — one succeeds,
one gets a clean 409.

---

## 6. Verification

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:3011/api/v1/transactions/party-balance?partyId=<cus_id>&companyId=<id>&type=CR"
```

Expect the envelope with `data[]` FIFO by `docDate`, `pendingAmount` > 0 on every
row, and `adjType`/`settlementMode` populated.

Then the client, with no rebuild: open Sales Bill, pick that customer, press
**F4**. The grid should list the credits with `pendingAmount` as Balance. Key an
amount over the balance → refused. Key one under → the Adjusted box on the bill
tracks it, and F5 asks for `total − adjusted`.

---

## Open items to close with the client/DB owner

1. **`cus_ledger_id`** (§0) — nothing works without it.
2. **`sbAdvanceAmt` vs `adjustments`.** Both describe the ADVANCE settlement. If
   bill posting turns *both* into rows, the advance posts twice. Recommended:
   `adjustments` is authoritative, `sbAdvanceAmt` is a header cache. The client
   already guarantees `sbAdvanceAmt == Σ(adjustments where billType='ADVANCE')`.
3. **Absent ≠ empty on re-save.** The client omits the `adjustments` key entirely
   when it could not read what a bill was settled with. `[]` must mean "reverse
   everything"; absent must mean "leave them alone". Edits write reversal rows
   (same `abj_adj_type`, negative amount, `abj_reversal_of_id`) — never UPDATE or
   DELETE.
4. **Schema drift.** The live DB carries `abj_bill_acc_year NOT NULL` and
   `abj_against_bill_acc_year`, neither of which is in
   `schema/09_acc_bill_balance.sql`. The client sends `againstBillAccYear` so the
   second can be filled; the first is NOT NULL with no default and carries no
   information. Drop both or update the file — do not leave them disagreeing.
5. **`abj_dr_cr` direction** is read off the balance sheet, not off a stated rule
   in file 09. Confirm before money moves.
6. **Credit-note liability ledger** — no column exists, so
   `abj_settlement_ledger_id` goes NULL for `NOTE_ADJUST` rather than borrowing
   the advance ledger and mixing two liabilities.
