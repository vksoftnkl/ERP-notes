# Customer / Supplier masters: delete guard, GST type list, last billed, customer ↔ supplier sync

Date: 2026-10-29 / From: Viji / Claude / For: the user, then Prathap

Status: **PLAN. Nothing is built.** No build starts before the user says "go".

This is a new file. It changes nothing in `accounts/notes-81-party-dual-role.md` or `accounts/notes-82-ledger-delete-party-guard.md`. It builds on them: one ledger can belong to a customer and a supplier, the group is not rewritten on a shared ledger, and the ledger is deleted only with the last role.

**Where the facts come from (read 2026-10-29):**
- the live catalog and data (`psql -h localhost -U claude -d ERP`);
- `ERP server` working tree (DEV-v1), including the uncommitted edits of the nestjs-backend agent that is working on `customer.service.ts`, the customer DTOs and `customer-detail.lookup.ts` right now;
- `NexERP` (main);
- `ERP client` (dev).

Row counts are test data. They size the clean-up only; they are not bugs.

**Work going on in parallel (plan around it, do not undo it).** The nestjs-backend agent's uncommitted edits cover:
- F3: a cleared number is stored as 0;
- F4: the price level and branch must exist (`CUSTOMER_PRICE_LEVEL_NOT_FOUND`, `CUSTOMER_BRANCH_NOT_FOUND`);
- F5: a blank name on update is refused;
- F6: the actor comes from the request user;
- F8: a name clash on create gives `CUSTOMER_NAME_EXISTS`;
- F10: the GSTIN must match the state (`CUSTOMER_GSTIN_STATE_MISMATCH`);
- DTO min/max lengths, plus GSTIN, PAN and e-mail patterns;
- the lookup's TCS flag and company check.

The build of this plan starts **after those edits land**, on top of them (§9).

---

## 1. Decisions log

| # | Date | Decision | By |
|---|---|---|---|
| 1 | 2026-10-29 | D1: refuse a customer delete ONLY when the ledger balance ≠ 0. A settled party may be deleted even if it has history. | user |
| 2 | 2026-10-29 | D2: GST type is a fixed list REGULAR, COMPOSITION, SEZ, OVERSEAS, UNREGISTERED (upper case). The server refuses anything else. | user |
| 3 | 2026-10-29 | D3: `cus_billed_date` / `cus_billed_count` mean the REAL last sale bill. Master saves stop touching them. They are set on bill post and corrected on cancel / void, by recomputing from bills (not ±1). | user |
| 4 | 2026-10-29 | D4: "do update supplier detail when customer record changes vice versa". | user |
| 5 | 2026-10-29 | "Balance" = the books' own balance (the Ledger Statement's one definition), taken over every year and checked in every company that has books for the ledger (§3.1). | Claude (proposed) |
| 6 | 2026-10-29 | The same delete guard applies to suppliers. | Claude (proposed, Q1) |
| 7 | 2026-10-29 | SEZ is stored on the ledger as `led_gst_party_reg_type = 'REGULAR'` + `led_is_sez = true`. OVERSEAS is stored as `'UNREGISTERED'`, with the state set to 96. The ledger CHECK is not widened. | Claude (proposed, Q5) |
| 8 | 2026-10-29 | The new `ck_cus_gst_type` / `ck_sup_gst_type` CHECKs land in phase 2, after every site runs the new build (offline sync, §3.3). | Claude |
| 9 | 2026-10-29 | The fields that sync and the fields that stay with one role are listed in §3.4. Company and branch go to the user (Q7). | Claude (proposed) |
| 10 | 2026-10-29 | D3: recommend computing "last billed" when it is read, instead of storing it (Q9). §3.6 gives both ways of building it. | Claude (proposed) |

## 2. Master register

Every item asked for, with its phase. P1 = this build; P2 = the follow-up once P1 is live.

| ID | Item | Phase |
|---|---|---|
| D1-1 | `DELETE /customers/delete` refuses when the ledger's balance ≠ 0 in any company | P1 |
| D1-2 | Same guard on `DELETE /suppliers/delete` | P1 (Q1) |
| D1-3 | A role delete that leaves the ledger alive (the other role stays) skips the balance guard | P1 (Q2) |
| D1-4 | `GET /customers/delete-check` + `GET /suppliers/delete-check`: balance per company + warnings (drafts, open bills, cheques in hand), so the confirm box can say why | P1 |
| D1-5 | Cheque in hand (PDC HELD/DEPOSITED, posted on clearing) refuses the delete | P1 (Q3) |
| D1-6 | Delete and delete-check check the menu `delete` right (10 customers, 22 suppliers) | P1 |
| D1-7 | Ledger Statement can open a deleted party (read-only); its picker gets "Show deleted" | P1 (Q4) |
| D1-8 | Reconciliation grid "Deleted parties with a balance" (catches the offline race in §3.3) | P2 |
| D1-9 | Restore a deleted customer / supplier | ✗ not in this plan (not asked; the user can ask for it later) |
| D2-1 | Server accepts only the 5 codes. Old spellings (`Regular`, `Unregistered`, …) are mapped. Anything else → `*_GST_TYPE_INVALID` | P1 |
| D2-2 | GSTIN required for REGULAR / COMPOSITION / SEZ; OVERSEAS has no GSTIN and state 96 | P1 (Q5, Q6) |
| D2-3 | Ledger fields: `led_gst_party_reg_type` + `led_is_sez` from the type, for customers AND suppliers (today a supplier's type never reaches its ledger) | P1 |
| D2-4 | Data clean-up of `cus_gst_type`, `sup_gst_type`, and the ledgers' reg type / SEZ flag (§3.2) | P1 |
| D2-5 | Seed state code 96 "Other Country" in `fixed.state_codes` | P1 |
| D2-6 | Qt: Customer, Supplier, Quick-add and party-role-link send codes (combo shows words, saves codes) | P1 |
| D2-7 | React: customer, supplier and quick-add use the 5 codes | P1 |
| D2-8 | CHECK `ck_cus_gst_type` / `ck_sup_gst_type`; `cus_gst_type` NOT NULL DEFAULT 'UNREGISTERED' | P2 |
| D2-9 | Tax treatment of SEZ / OVERSEAS on the sale bill (zero-rated / export) and of an OVERSEAS supplier on the purchase invoice (import) | ✗ not in this plan. Today `bill-lifecycle` never reads the type, and `purchase-invoice.service.ts:436` treats anything other than COMP/UNREG as REGULAR. Needs its own plan. |
| D2-10 | Posted documents keep the type they were saved with (`sb_cust_gst_type`, `gdr_party_gst_type` …) | P1: no change (history) |
| D3-1 | Customer master saves stop writing `cus_billed_date` / `cus_billed_count` (create, update, restore-on-link) | P1 |
| D3-2 | "Last billed" = latest POSTED sale bill; "count" = number of POSTED sale bills | P1 (Q9 picks stored vs read-time) |
| D3-3 | Index `ix_sb_cust_posted` on `sales.sale_bill` | P1 |
| D3-4 | Back-fill existing rows from bills | P1 |
| D3-5 | `sup_billed_date`: supplier saves stop writing it (same bug); real "last purchase invoice" later | P1 stop writing / P2 fill (Q10) |
| D3-6 | DTOs keep `cusBilledDate` / `cusBilledCount` / `supBilledDate` for one release, ignored (the global pipe has `forbidNonWhitelisted`: removing them 400s the React form, which sends them today) | P1, removed P2 |
| D4-1 | Shared party-detail sync: customer edit → supplier + ledger; supplier edit → customer + ledger; one transaction | P1 |
| D4-2 | One field map (`cus_*` ↔ `sup_*` ↔ `led_*`) in one file, used by both services (§3.4) | P1 |
| D4-3 | Company / branch rule on a shared party | P1 (Q7) |
| D4-4 | Audit row for each row written (customer, supplier, ledger) | P1 |
| D4-5 | Supplier GSTIN must match its state (new; the customer has it from F10). Needed because a supplier edit now writes the customer. | P1 |
| D4-6 | Save response carries `partySync` (which other rows were changed) | P1 |
| D4-7 | Qt + React: a note on a shared party's form ("Also changes the supplier record") and a toast after save | P1 |
| D4-8 | Data-change event for the other resource too (`suppliers` / `customers` / ledger), so open lists refresh | P1 |
| D4-9 | Reconciliation grid "Shared parties out of step" (catches offline last-push-wins, §3.3) | P2 |
| D4-10 | Rights: a user who may edit customers but not suppliers still changes the supplier's party details | P1 (Q8) |
| X-1 | Today the customer and supplier routes check no menu right at all (only the access token) | ✗ only delete / delete-check get a right here (D1-6). Save is left as it is; raise it separately if wanted. |
| X-2 | `sales.customers` stores `cusCollectionDays` in a column named `sup_collection_days` | noted only, no change |

## 3. Design and DB

### 3.1 D1: what "balance" means, and the delete guard

**One definition, the books' own.** The Ledger Statement (`reports/ledger-statement/ledger-statement.service.ts`, header comment §4) and `reports/common/voucher-books.sql.ts` already define a balance:

> balance(Y, asOf) = the year's opening set (`acc_opening_balance`, 'D' +, 'C' −) + Σ `av_signed_amount` of live legs whose header is POSTED or CANCELLED (a cancel leaves a POSTED mirror, so the pair nets to zero). DRAFT never counts.

The delete guard reuses `countedLegsSql` / `openingSetSql` and does **not** write its own sum. It only adds the walk across years.

**Lifetime balance of ledger L in company C** (new `PartyBalanceService.balances(tx, ledId)`):

1. Companies = every `av_company_id` with a leg for L, plus every `op_company_id` with an opening row for L.
2. For each company C, walk its `public.fiscal_years` (`comp_id = C`, not deleted) in `fy_begin_date` order, up to the last year that has a leg or an opening row:
   - `running` = the opening set of (C, Y, L, all branches) **when such a row exists and is not stale** (`op_is_stale = false`). An opening row replaces the carried figure, which is what a carry-forward is.
   - Otherwise `running` carries over from last year's closing. This way a year whose carry-forward was not run, or is stale, still counts.
   - `running += Σ counted legs` of (C, Y, L, all branches, all dates).
3. balance(C) = `running` after the last year, in Decimal.

The delete is refused when **any** company's balance ≠ 0 (exact Decimal compare, no epsilon).

**A shared ledger (`led_company_id IS NULL`)** is checked in every company that has books for it, and the refusal names the first company with a balance. Live today, no NULL-company ledger has legs in more than one company. The rule still has to hold.

**What else counts, and what does not:**

| Item | Treatment | Why |
|---|---|---|
| Opening balance (`acc_opening_balance`) | counts (step 2) | it is the start of the books |
| Posted bills, receipts, payments, journals, returns | count (legs) | they are the balance |
| Advance received / paid | counts | an advance is a credit (or debit) leg, so balance ≠ 0 and the delete is refused |
| Temp credit (`acc_temp_credit`) | counts | TEMP_CR posts like CREDIT, so the bill stays open on the party's ledger and the balance ≠ 0 |
| Open bills whose net is 0 (e.g. a ₹500 bill open + ₹500 on account, not allocated) | **allowed**, with a warning | the user's rule is the balance; bill-wise allocation that was not done is tidy-up, not money |
| Unposted drafts (sale bill / order / quotation DRAFT, receipt / payment / voucher DRAFT, purchase invoice DRAFT) | **allowed**, with a warning | a draft moves no money. After the delete, posting the draft is refused by the existing party check (`receipt.guards.ts loadParty` → `VCH_PARTY_NOT_FOUND`; sale / purchase party checks) |
| Cheque in hand: `acc_pdc_register` `apd_status IN ('HELD','DEPOSITED')` and `apd_posting_mode = 'ON_CLEARING'` | **refused** (recommended, Q3) | the money is not in the balance yet, but clearing it posts a receipt against a deleted party and the cheque gets stuck. An `ON_RECEIPT` cheque is already in the balance. |

**Delete flow (customer, and the same for the supplier):**

1. Check the `delete` right on menu 10 (22 for the supplier) with `assertMenuRight` from `common/posting/rights.ts`.
2. Lock the ledger row (`SELECT … FOR UPDATE`), so a bill cannot post in between online.
3. Ask whether the other role is live:
   - It is live (Q2, recommended): the ledger is not deleted, so its balance stays with the other role. Skip the balance guard and delete the role row only, as today.
   - It is not: run the balance check. Any company ≠ 0 → 409 `CUSTOMER_HAS_BALANCE`. Then run the cheque check → 409 `CUSTOMER_HAS_CHEQUE_IN_HAND`.
4. Soft-delete as today (customer row; the ledger only if this is the last role) and write the audit row.

**History screens after a delete (Q4):**

| Screen | Today | After |
|---|---|---|
| Receipts list (grid 108), Payments list (123), Bills list (86), Voucher Register (117), Purchase Invoice list (189) | show the party name (their SQL has no `led_is_deleted` filter: checked) | no change |
| Receipt `/get` | loads `ledger.ledName` without a deleted filter | no change |
| New receipt / payment / bill for a deleted party | refused (`VCH_PARTY_NOT_FOUND`, pickers 79/100 hide deleted parties) | no change. That is right. |
| Ledger Statement | the picker hides deleted ledgers (`ledgers`, line ~147). The statement refuses them as `LEDGER_NOT_FOUND` (line ~421). | `GET /reports/ledger-statement/ledgers?includeDeleted=true` lists them with `isDeleted: true`. Every statement route accepts a deleted ledger (read-only report). The menu-258 right is unchanged. |
| Party Outstanding | a deleted party has nothing outstanding (balance 0) | no change |

### 3.2 D2: the GST type list and the clean-up

**The list.** `PartyGstType = REGULAR | COMPOSITION | SEZ | OVERSEAS | UNREGISTERED`. It lives in the new `src/common/party/party-gst-type.ts`. The function `toLedgerGstPartyRegType` (customer.service.ts:198) moves there.

| Party type | GSTIN | State | `led_gst_party_reg_type` | `led_is_sez` |
|---|---|---|---|---|
| REGULAR | required | any; must match the GSTIN | REGULAR | false |
| COMPOSITION | required | must match the GSTIN | COMPOSITION | false |
| SEZ | required | must match the GSTIN | REGULAR | **true** (`chk_led_sez_consistency` requires REGULAR + a GSTIN) |
| OVERSEAS | must be blank | **96 Other Country** (Q6) | UNREGISTERED | false |
| UNREGISTERED | blank (Q5: refuse one that is typed) | any | UNREGISTERED | false |

The ledger's CHECK `chk_led_gst_party_reg_type` stays REGULAR | COMPOSITION | UNREGISTERED. Widening it is not needed: SEZ is a flag on the ledger, and Overseas is "no GSTIN, foreign state". `led_gst_party_type` has no reader in the code and is left alone.

**Normalising old spellings (offline-safe).** Before validating, the server maps known legacy spellings: trim, upper-case, strip spaces, `_` and `-`. `Regular` → REGULAR, `Unregistered` → UNREGISTERED, `Composition` → COMPOSITION, `Sez`/`SEZ Unit` → SEZ, `Overseas` → OVERSEAS. This way an old Qt build that still sends "Regular" keeps working until it is updated. Anything else → 400 `CUSTOMER_GST_TYPE_INVALID` / `PURCHASE_SUPPLIER_GST_TYPE_INVALID`.

On create a blank type means UNREGISTERED when there is no GSTIN; when a GSTIN is given it is refused as `*_GST_TYPE_REQUIRED` (the supplier already refuses a blank type).

**Who reads the strings (checked):**
- The sale documents copy the customer's string into their own column when the party is chosen (`bill.service.ts:1116` `sbCustGstType = cus.cusGstType`; also `sq_`, `so_`, `sdc_`, `sr_`). Posting copies it on into `gdr_party_gst_type` (`bill-lifecycle.service.ts:2290`, `doc-register.service.ts:117`). Nothing compares it: neither the sale tax engine nor e-invoice reads it (no GSTR module exists yet). The print header shows it (`sale-bill-header.provider.ts:63`).
- Purchase reads `sup_gst_type`:
  - `purchase-invoice.service.ts:436`: upper-case, `COMP*` / `UNREG*` / else REGULAR. SEZ → REGULAR is right. OVERSEAS → REGULAR is wrong, see D2-9.
  - `purchase-rcm.ts:42`: the regex `/unregist/i` keeps working.
  - `purchase-lookup.ts:113` and the Qt `purchase_context.cpp:117` chip show it raw. Qt must show the word, not the code.
- Posted documents are history and are not rewritten (D2-10). The register holds `Unregistered` 14, `Regular` 12, `UNREGISTERED` 4. A future GSTR reader must upper-case.

**Clean-up (one data migration, db-schema; dry-run first in a rolled-back transaction).** Live counts (all rows / not deleted):

`sales.customers.cus_gst_type`:

| Now | Rows (all / live) | → New | Note |
|---|---|---|---|
| NULL | 18 / 4 | UNREGISTERED | none has a GSTIN |
| `Unregistered` | 16 / 7 | UNREGISTERED | 1 live row (MADHAVAN) has a GSTIN: listed below |
| `UNREGISTERED` | 8 / 6 | UNREGISTERED | 1 deleted row has a junk GSTIN `21WSQS` |
| `COMPOSITION` | 5 / 3 | COMPOSITION | 2 live rows (pors, uopmm) have no GSTIN: listed below |
| `Regular` | 5 / 5 | REGULAR | all have a GSTIN |
| `REGULAR` | 4 / 3 | REGULAR | all have a GSTIN |
| `test` | 2 / 0 | UNREGISTERED | the one with a GSTIN has `WDSDS232`, not a GSTIN |
| `test21` | 2 / 0 | 1 REGULAR (valid GSTIN `33AAACP1234A1Z5`) + 1 UNREGISTERED | both deleted |

`purchase.suppliers.sup_gst_type` (NOT NULL already):

| Now | Rows (all / live) | → New | Note |
|---|---|---|---|
| `Unregistered` | 26 / 4 | UNREGISTERED | |
| `Regular` | 6 / 6 | REGULAR | |
| `COMPOSITION` | 2 / 2 | COMPOSITION | neither has a GSTIN: listed below |
| `REGULAR` | 1 / 1 | REGULAR | |

The rule in SQL: `upper(btrim(x))` in the 3 known words → that word. Anything else or NULL → `REGULAR` if `x_gst_no ~ '^[0-9]{2}[0-9A-Z]{10}[0-9A-Z]{3}$'`, else `UNREGISTERED`.

**Contradictions the clean-up leaves alone** (the type is what the operator picked; the new rule applies at their next save). The list goes in the migration's report and to the user:
- customer MADHAVAN (`019f659c-3942-…`) is UNREGISTERED with GSTIN 33AABCA8398K1ZA;
- customers pors, uopmm and suppliers ccscdsssss, testfddfd are COMPOSITION with no GSTIN.

**Ledgers.** For every ledger that backs a customer or a supplier, set `led_gst_party_reg_type` (and `led_is_sez`) from the cleaned type:
- On a shared ledger where the two roles disagree, the non-NULL one wins, then the customer.
- Live today: 25 customer ledgers have a NULL reg type although the customer has a type (18 more have none on either side), and 34 of 35 supplier ledgers have NULL. Supplier types never reached the ledger before.
- Ledgers with `led_is_sez = true` but no party (3, test data that breaks the NOT VALID `chk_led_sez_consistency`) are not touched.

**Seed:** `fixed.state_codes` row `('96', 'Other Country', …)`, the GST portal's code for a foreign party. Without it an OVERSEAS customer cannot pass `ensureStateCodeExists`, and `sb_pos_stcd`'s FK refuses the bill.

### 3.3 Offline checks (every item)

| Question | Answer |
|---|---|
| Two offline branches both delete / edit the same party? | A delete is a soft delete (an UPDATE). Last push wins, as for every master today (cloud-sync upserts the whole row by PK, under `session_replication_role = replica`). |
| Branch A deletes (its local balance is 0) while branch B bills the same party offline? | Both sync. The cloud then holds a deleted party with a balance. It cannot be stopped at the sync without refusing a real sale, so it is detected instead: grid D1-8 "Deleted parties with a balance" (P2). |
| Balance computed on a device sees only that device's books | Same as above. The guard is right online; offline it is best-effort, and D1-8 catches the rest. |
| CHECK on GST type | Single-row arithmetic: allowed by `offline-sync-invariants`. **But** CHECKs still run under replica mode, so an old-build device that saved `Regular` locally would fail its push. That is why the CHECK is P2, after every site runs the new build. P1 relies on server-side normalising. |
| UNIQUE | None added. Name uniqueness stays the ledger's existing per-company index. |
| D3 stored last-billed (option A) | **Hazard:** every bill post dirties the `customers` row, and sync pushes the WHOLE row. A branch's bill can then overwrite a credit-limit edit made at HO since that branch last synced. In addition, each device counts only its own bills, so the cloud shows whichever device pushed last. Option A therefore needs a cloud-side recompute after each sync run (§3.6). Option B has neither problem. → Q9. |
| D4 sync offline | Device X edits the customer's phone while device Y edits the supplier's address of the same party, both offline. Each device keeps its own three rows consistent. On push, the last pusher wins each row, so the three rows can disagree on the cloud. This cannot be prevented without field-level merge, which the sync does not have. It is detected instead (grid D4-9, P2), and re-saving either role puts the three rows back in step. |
| Numbering | No document numbers are involved. |

### 3.4 D4: the party-detail map (cus ↔ sup ↔ led), from the live catalog

"Sync" = a save of either role writes the value to the other role's row and to the ledger, when the payload carries that key (same `hasOwnProperty` rule as today). A max length is the narrowest of the three columns. The DTO `@MaxLength` the other agent is adding must use these widths for the synced keys, or a long address saved on one side fails on the other.

| Party detail | `sales.customers` | `purchase.suppliers` | `accounts.acc_ledger_master` | Sync | Max |
|---|---|---|---|---|---|
| Name | cus_name varchar(200) NULL | sup_name varchar(200) NOT NULL | led_name varchar(200) NOT NULL | all 3 | 200 |
| Short name | cus_short (50) | sup_short (50) | led_short (50) | all 3 | 50 |
| Address 1–3 | cus_addr1..3 (250) | sup_addr1..3 (250) | led_addr1..3 (**200**) | all 3 | 200 |
| City | cus_city (250) | sup_city (250) | led_city (**100**) | all 3 | 100 |
| District | cus_district (250) | sup_district (250) | led_district (**100**) | all 3 | 100 |
| State name | cus_state_name (100) NOT NULL | sup_state_name (100) NOT NULL | led_state_name (100) | all 3 | 100 |
| State code | cus_state_code char(2) NOT NULL | sup_state_code char(2) NOT NULL | led_state_code char(2) | all 3 | 2 |
| Country | cus_country (60) | sup_country (60) | led_country (60) | all 3 | 60 |
| PIN | cus_pin (10) | **sup_pincode** (10) | led_pin (10) | all 3 | 10 |
| Telephone | cus_tel (20) | sup_tel (20) | led_tel (20) | all 3 | 20 |
| Mobile | cus_phone1 (20) | **sup_phone** (20) | led_phone1 (20) | all 3 | 20 |
| Phone 2 | cus_phone2 (20) | (none) | led_phone2 (20) | cus ↔ led | 20 |
| WhatsApp | cus_whatsapp_no (20) | sup_whatsapp_no (20) | led_whatsapp_no (20) | all 3 | 20 |
| E-mail | cus_email (120) | **sup_mail_id** (120) | led_email (150) | all 3 | 120 |
| Contact person | cus_contact_person (150) | (none) | led_contact_person (150) | cus ↔ led | 150 |
| Aadhaar | cus_aadhar_no (12) | (none) | led_aadhar_no (20) | cus ↔ led | 12 |
| GSTIN | cus_gst_no (15) | sup_gst_no (15) | led_gstin_no (15) | all 3 | 15 |
| PAN | cus_pan_no (10) | sup_pan_no (10) | led_pan_no (10) | all 3 | 10 |
| GST type | cus_gst_type (30) | sup_gst_type (30) NOT NULL | led_gst_party_reg_type (20) + led_is_sez | all 3 (§3.2 mapping) | code |
| e-Commerce GSTIN | cus_ecommerce_gstin (15) | (none) | led_ecommerce_gstin (15) | cus ↔ led | 15 |
| Delivery (region) name | cus_region_name (200) | sup_region_name (200) | led_region_name text | all 3 | 200 |
| Delivery address 1–3 | cus_region_addr1..3 (250) | sup_region_addr1..3 (250) | led_region_addr1..3 (**200**) | all 3 | 200 |
| Delivery city / district | cus_region_city / _district (250) | sup_region_city / _district (250) | led_region_city / _district (**100**) | all 3 | 100 |
| Delivery state name | cus_region_state_name (100) | sup_region_state_name (100) | led_region_state_name (100) | all 3 | 100 |
| Delivery country | cus_region_country (60) | sup_region_country (60) | led_region_country (60) | all 3 | 60 |
| Company | cus_company_id uuid | sup_company_id uuid | led_company_id uuid | **Q7** | |
| Branch | cus_branch_id uuid | sup_branch_id uuid | led_branch_id uuid | **Q7** | |

**Stays with its role (never synced):**
- Customer: title, code, landmark; credit allowed, bill limit, amount limit, credit days, debit balance, grace days; discount %; SMS, overdue and promotion / loyalty / discount flags; sort order; birth / marriage date; transport, freight / loading / unloading; TCS / IT-collection pair; geo location; collection days; default salesman; price level; area; customer group; notes; active; billed date / count.
- Supplier: supplier group; purchase type; website; cheque name; notes; credit days; cash discount %; collection days; CST; drug licence + expiry; MSME four; FSSAI + valid-to; default godown; show notes; import; sort order; active; billed date.
- Ledger-owned (unchanged rules): `led_group_id` (notes 81: no role edit moves a shared ledger's group); bank accounts (`acc_ledger_bank_accounts`, already one set per ledger); TDS pair; Tally fields; `led_allow_sms` / `led_sort_order` / TCS-TDS flags (customer-only today, unchanged); `led_is_active` = customer active OR supplier active (as today).
- `led_remarks`: today BOTH roles write their notes into it, so the last save wins. On a shared ledger neither role writes it any more (notes are per role). On a single-role ledger it stays as today.

**Name.** Name syncs. A rename of either role renames the ledger and the other role. Uniqueness:
- the ledger's per-company index is the guard (`rethrowLedgerNameClash` → `CUSTOMER_NAME_EXISTS` / `PURCHASE_SUPPLIER_NAME_EXISTS`);
- in addition, the supplier's own `ensureNameIsUnique` runs when a customer edit renames the supplier.

**Company and branch: the concern, and three ways to handle it (Q7).** Today a customer edit writes `led_company_id` / `led_branch_id`. On a shared ledger that moves the supplier's books' ledger to another company: the receipt and payment `loadParty` then hides it from the old company (`OR ledCompanyId = companyId`).
- (a) **Recommended.** Company and branch are party details and sync to all three, so they never disagree. A **company** change is refused, for any party, shared or not, when the ledger has books (a leg or an opening row) in its current company: 409 `CUSTOMER_COMPANY_HAS_BOOKS` / `PURCHASE_SUPPLIER_COMPANY_HAS_BOOKS`. Branch changes are free (the ledger branch is informational).
- (b) On a shared party, company and branch are locked. Changing them is refused until one role is deleted.
- (c) They stay per role and the ledger keeps its own when shared. This allows a customer in company A whose supplier side is in company B, which the one-ledger design cannot really hold. Not recommended.

### 3.5 D4: one transaction, audit, rights, response

New `src/common/party/party-sync.service.ts` (+ `party-detail.map.ts` holding the table above as data, and `party.module.ts`). Both services call it inside their existing `$transaction`. It replaces `CUSTOMER_TO_LEDGER_FIELD_MAP` / `SUPPLIER_TO_LEDGER_FIELD_MAP` for the synced keys; the role-only ledger keys stay in each service.

The order inside the one transaction, e.g. a customer save:
1. `SELECT … FROM acc_ledger_master WHERE led_id = $1 FOR UPDATE`. This serialises a customer save and a supplier save of the same party online.
2. Validate everything first:
   - GST type + GSTIN + state (§3.2);
   - the GSTIN state for both sides (D4-5);
   - the company rule (Q7);
   - the name uniqueness on both role tables.

   Nothing is written before every check passes.
3. Write the customer row (as today).
4. `updateLedgerWithinTx`, as today. It already writes the ledger's own audit row (`account-ledger-masters.service.ts:443`).
5. When a live supplier row exists, write its synced columns, but only those whose value actually changes (`IS DISTINCT FROM`), so an unchanged row is not dirtied for sync. Also set `sup_modified_on` / `sup_modified_by` = the request user.
6. Audit rows:
   - the customer's row as today;
   - plus one supplier row: `tableName 'suppliers'`, `screenName 'Supplier Master'`, `action 'update'`, before/after payloads, `notes 'Party details updated from Customer Master'`.

   Each table that changed gets its own row.
7. Publish the data-change event for `suppliers` (and the ledger resource) as well, so an open supplier list refreshes (D4-8). The HTTP cache purge is already `'*'`.

The supplier save is the mirror image (`'Party details updated from Supplier Master'`).

**Rights (Q8).** The sync writes the other role even if the user has no edit right on the other master. Recommended: yes, because it is one party; the audit row names the user and the screen.

**Save response, added key** (`data.partySync`, both masters; everything else unchanged):

```json
"partySync": { "otherRole": "SUPPLIER", "otherRoleUpdated": true, "fields": ["supAddr1", "supPhone"] }
```

`otherRole` is `"SUPPLIER" | "CUSTOMER" | null`; `fields` uses the other role's own keys, empty when nothing changed.

### 3.6 D3: last billed

**What it means.** Last billed = `max(sb_bill_date)`; count = `count(*)` over `sales.sale_bill` where `sb_cust_id = cus_id`, `sb_status = 'POSTED'` and `sb_is_deleted = false`, across all companies, branches and years.
- A cancelled or voided bill is excluded: a void is a cancel, through `BillLifecycleService.cancelIn`, `bill-void.service.ts:235`.
- An amend cancels the old bill and posts a new one, so it counts once.
- Sale returns do not reduce the count.

**Index (db-schema):** `ix_sb_cust_posted ON sales.sale_bill (sb_cust_id, sb_bill_date DESC) WHERE sb_status = 'POSTED' AND sb_is_deleted = false`, created the house way for a LIST-partitioned table (the parent plus each `sale_bill_YYYY_YYYY` partition). There is no index on `sb_cust_id` today.

**D3-1, either way:** remove `cusBilledDate: now` / `cusBilledCount: 1` from create, `cusBilledDate: now` / `increment: 1` from update, and `cusBilledCount: { increment }` from the restore-on-link path (`customer.service.ts:414, 517–522, 683–686`). Do the same for `supBilledDate: now` (`suppliers.service.ts:258, 448`). The DTO keys stay one release, ignored (D3-6).

**Option A (as decided): stored, recomputed.**
- New `PartyBilledService.recomputeCustomers(tx, cusIds[])`. It is one set-based `UPDATE sales.customers c SET cus_billed_date = x.d, cus_billed_count = x.n FROM (…GROUP BY sb_cust_id…)`, plus a second UPDATE that zeroes the ids that have no bill left (the `GROUP BY` trap from the loyalty work).
  - It writes only `IS DISTINCT FROM` rows.
  - It does not touch `cus_modified_on` / `_by` (a bill is not a master edit).
- Called from `BillLifecycleService.postCore` (post and amend) and `cancelIn` (cancel, void, and amend's cancel), with the old AND the new `sb_cust_id` on an amend that changes the customer.
- Offline: after each cloud-sync run, the run calls the same recompute against the cloud connection, for the customers that appear in that run's pushed `sale_bill` rows. Otherwise the cloud shows the last pusher's local count (`cloud-sync.service.ts`, after all tables).
- The whole-row overwrite hazard of §3.3 remains.

**Option B (recommended, Q9): computed when read, never stored.**
- `customer-detail.lookup.ts` (the "n days : dd/MM/yy" text, `billed_date`) and `GET /customers/get` (`cusBilledDate`, `cusBilledCount`) run the one indexed query above. The JSON keys stay the same, so nothing changes in the clients.
- The columns stay (no DDL); nothing writes them; drop them in P2.
- It is right on every device and on the cloud, it adds no write to the bill post, and it never touches the customer row.

**Back-fill (D3-4).**
- Under A: one UPDATE from the same query. Live, 63 customers: 4 have posted bills, 2 already agree, and 57 carry a "billed" date that came from master saves with no bill behind it. Those become NULL / 0.
- Under B: no back-fill; set the columns to NULL / 0 once, so a stray reader cannot show a fake date.

**Readers found:**
- server: `customer-detail.lookup.ts:132` (`billed_date`), `GET /customers/get` payload;
- React: quotation header `header-blocks.tsx:179` "· last billed …" (via the lookup); the customer form holds `cusBilledDate` / `cusBilledCount` as form values and SENDS them (`page.tsx:3076–3077`);
- Qt: only a TODO in `quotation_entry.cpp:2021`; no screen shows it;
- grids: none (no registered grid reads `cus_billed_*` / `sup_billed_*`).

## 4. API

The menu rights are 10 = Customers and 22 = Suppliers. Every error is `errors[].code` + the existing envelope. Wording goes in `prisma/seed/Error_Messages.json` → `fixed.error_message` (`erm_title / erm_line / erm_todo`); amounts are shown in ₹ with Dr/Cr.

### 4.1 Routes

| Route | Change | Right checked |
|---|---|---|
| `POST /customers/create` (create + update by `cusId`) | GST type list and GSTIN rules; billed keys ignored; party sync; `data.partySync` added | none today (X-1) |
| `GET /customers/get?cusId=` | under Q9-B `cusBilledDate` / `cusBilledCount` are computed from bills | none today |
| `DELETE /customers/delete?cusId=` | balance and cheque guard (§3.1) | menu 10 `delete` |
| `GET /customers/delete-check?cusId=` | **new** | menu 10 `delete` |
| `POST /suppliers/create` | GST type list + GSTIN + GSTIN-state; type now reaches the ledger; party sync; `data.partySync` | none today |
| `DELETE /suppliers/delete?supId=` | same guard | menu 22 `delete` |
| `GET /suppliers/delete-check?supId=` | **new** | menu 22 `delete` |
| `GET /reports/ledger-statement/ledgers` | `includeDeleted` (bool, default false); item gets `isDeleted` | menu 258 view (unchanged) |
| `GET /reports/ledger-statement/header \| vouchers \| voucher-legs \| daily \| monthly \| export` | a deleted ledger is accepted (read-only) | unchanged |

`delete-check` answers 200 even when the delete would be refused. It is a question, not an action:

```json
{ "success": true, "data": {
  "partyId": "uuid", "role": "CUSTOMER", "otherRoleLive": false, "ledgerWouldBeDeleted": true,
  "canDelete": false, "refusalCode": "CUSTOMER_HAS_BALANCE",
  "balances": [ { "companyId": "uuid", "companyName": "Acme Foods", "amount": "1250.00", "drCr": "DR" } ],
  "warnings": { "draftDocs": 2, "openBills": 0, "chequesInHand": 0 }
} }
```

`balances` lists only the companies whose balance is not zero; it is empty when every company's balance is 0. `amount` is a string, 2 decimals, always positive, with the side in `drCr`.

### 4.2 Refusals (new codes)

| Code | HTTP | When | Title / what happened / what to do |
|---|---|---|---|
| `CUSTOMER_HAS_BALANCE` | 409 | delete, the ledger balance ≠ 0 in a company | Cannot delete customer / This customer still has ₹{amount} {Dr/Cr} in {company}. / Settle it to zero first, then delete. |
| `PURCHASE_SUPPLIER_HAS_BALANCE` | 409 | same for the supplier | Cannot delete supplier / This supplier still has ₹{amount} {Dr/Cr} in {company}. / Settle it to zero first, then delete. |
| `CUSTOMER_HAS_CHEQUE_IN_HAND` | 409 | delete, a received cheque not yet cleared (Q3) | Cannot delete customer / A cheque from this customer is not cleared yet. / Clear or return the cheque first. |
| `PURCHASE_SUPPLIER_HAS_CHEQUE_IN_HAND` | 409 | delete, an issued cheque not yet cleared (Q3) | Cannot delete supplier / A cheque to this supplier is not cleared yet. / Clear or cancel the cheque first. |
| `CUSTOMER_GST_TYPE_INVALID` | 400 | type not one of the 5 | Wrong GST type / GST type must be Regular, Composition, SEZ, Overseas or Unregistered. / Pick one from the list. |
| `PURCHASE_SUPPLIER_GST_TYPE_INVALID` | 400 | same | (same wording, "supplier") |
| `CUSTOMER_GST_TYPE_REQUIRED` | 400 | blank type with a GSTIN | GST type missing / This customer has a GST No but no GST type. / Pick the GST type. |
| `CUSTOMER_GSTIN_REQUIRED` | 400 | REGULAR / COMPOSITION / SEZ without a GSTIN | GST No needed / A {type} customer must have a GST No. / Type the GST No, or choose Unregistered. |
| `PURCHASE_SUPPLIER_GSTIN_REQUIRED` | 400 | same | (same, "supplier") |
| `CUSTOMER_GSTIN_NOT_ALLOWED` | 400 | UNREGISTERED or OVERSEAS with a GSTIN (Q5) | GST No not allowed / An {type} customer has no GST No. / Clear the GST No, or choose Regular. |
| `PURCHASE_SUPPLIER_GSTIN_NOT_ALLOWED` | 400 | same | (same, "supplier") |
| `CUSTOMER_OVERSEAS_STATE` | 400 | OVERSEAS with a state other than 96 (Q6) | Wrong state / An overseas customer's state must be "Other Country". / Choose Other Country as the state. |
| `PURCHASE_SUPPLIER_OVERSEAS_STATE` | 400 | same | (same, "supplier") |
| `PURCHASE_SUPPLIER_GSTIN_STATE_MISMATCH` | 400 | supplier GSTIN's first 2 digits ≠ state code (D4-5) | GST No and state differ / This GST No is from state {nn}, but the state is {mm}. / Correct the GST No or the state. |
| `CUSTOMER_COMPANY_HAS_BOOKS` | 409 | company change on a party with books in its current company (Q7-a) | Cannot change company / This party has entries in {company}. / Keep the company. Make a new party for the other company. |
| `PURCHASE_SUPPLIER_COMPANY_HAS_BOOKS` | 409 | same | (same) |
| `SALES_CUSTOMER_*` / existing codes | | unchanged | |

Existing codes reused:
- `CUSTOMER_NOT_FOUND` / `PURCHASE_SUPPLIER_NOT_FOUND`;
- `CUSTOMER_NAME_EXISTS` / `PURCHASE_SUPPLIER_NAME_EXISTS`, now also for a clash caused by the sync rename;
- `CUSTOMER_GSTIN_STATE_MISMATCH` (F10);
- the rights refusal of `assertMenuRight`.

The existing CUSTOMER_* / PURCHASE_SUPPLIER_* codes have **no** wording rows in `fixed.error_message` today. db-schema adds the new ones above. Wording for the old ones is a separate small job (notes-119 style); it is not in this plan.

## 5. Qt screens (NexERP)

These are existing screens on `NexCrudForm` / the NexCrud list, so TxnKeys does not apply: they are masters, not txns. No new menu, grid or ui_table in P1. **No mockup PNG:** nothing is laid out anew. There is one info line, combo items, and two message boxes, described exactly below; ask if a PNG is wanted.

- **Customer Entry** (menu 10, `src/modules/sales/master/customer/customer_entry.cpp`):
  - line 277: `F::staticCombo` → `F::staticComboCoded("cusGstType", "GST Type", {{"REGULAR","Regular"},{"COMPOSITION","Composition"},{"SEZ","SEZ"},{"OVERSEAS","Overseas"},{"UNREGISTERED","Unregistered"}})`.makeRequired(). The combo shows words and saves codes; on edit it prefills by value, and old rows are clean after §3.2;
  - line 62: `gst.regTypeCoded = true`, so the GSTIN fetch fills the code;
  - line 387 `extraValidate`: compare the combo's **data** (code), not `currentText()`. Add OVERSEAS → state 96 and no GSTIN.
- **Supplier Entry** (menu 22, `src/modules/purchase/master/supplier/supplier_entry.cpp`): the same three changes (lines 26, 82, 186–202).
- **Customer quick-add** (`src/modules/sales/common/customer_quick_add_dialog.cpp:21–23, 162`): send the code (`currentData()`), not the text.
- **Party role link** (`src/shared/party/party_role_link.cpp:156–161`): prefill the code (`REGULAR` …, plus `SEZ` when `ledIsSez`), not the word.
- **Purchase context chip** (`src/modules/purchase/common/purchase_context.cpp:117`): show the word for the code.
- **Shared-party note (D4-7).** On a SAVED form, `PartyRoleLink` already probes the other master (404 = none). When the other role answers 200, show one muted info line under the form heading. The style is a theme class, not a colour:
  - Customer: "This party is also a supplier. Name, address, phones, GST No, PAN and GST type also change on the supplier."
  - Supplier: "This party is also a customer. Name, address, phones, GST No, PAN and GST type also change on the customer."
  - After save, when `partySync.otherRoleUpdated`, the status / toast reads "Saved. Supplier record updated too." (or "Customer").
- **Delete from the Customer / Supplier list:**
  - Before the confirm, call `GET /customers/delete-check` (or `/suppliers/…`). This is the same hook pattern as `PartyRoleLink::guardLedgerDelete`; the new one is `guardPartyDelete`.
  - When `canDelete` is false, show the server's wording (notes-120 `user` block) and stop.
  - When there are warnings, the confirm text adds: "It has {n} unposted drafts; they cannot be posted after the delete." The open-bills warning reads the same way.
  - The DELETE refusal is still shown in case of a race.
- **Ledger Statement** (menu 258): the ledger picker gets a "Show deleted" tick. It sends `includeDeleted=true`, and a deleted row is labelled "(deleted)".
- **Locked when:** nothing new. The company / branch fields are locked on a shared party only if the user picks Q7-b.

## 6. React (ERP client)

The same routes, keys and behaviour as Qt; nothing should differ.
- `features/masters/sales/customer/customer-master.constants.ts:179` `GST_TYPE_OPTIONS` → the 5 codes; labels Regular / Composition / SEZ / Overseas / Unregistered.
- `features/masters/purchase/suppliers/constants.ts:84` `SUPPLIER_GST_TYPE_OPTIONS` → codes (today it saves the words); `SUPPLIER_UNREGISTERED_GST_TYPE = "UNREGISTERED"`; `transformers.ts:108/311` keep the case-insensitive load.
- `features/sales/salebill/salebill.quick-add.ts:31` `QUICK_ADD_GST_TYPES` → add SEZ / OVERSEAS only if the quick-add should offer them (recommended: no, the counter keeps 3).
- The customer form stops sending `cusBilledDate` / `cusBilledCount` (`page.tsx:3076–3077`), and the supplier form stops sending `supBilledDate` (`form-builder.ts:104`). Where shown, they are read-only.
- Shared-party note, toast, delete-check and the Ledger Statement "Show deleted" work as in Qt. `features/masters/shared/party-role-link.ts` already probes the other role.

## 7. Tests

**API (api-tester, `ZT-` data, VKPOS on Acme Foods / Coimbatore; cancel what you post).**

D1:
1. A customer with no books: delete-check gives `canDelete:true`, then the delete → 200 and the ledger is soft-deleted.
2. Post a ZT credit bill: delete-check gives `canDelete:false`, `CUSTOMER_HAS_BALANCE` with the amount + DR. The delete → 409 with the same code, and nothing is changed.
3. Receive the full amount: the balance is 0 → the delete is allowed (history kept).
4. An advance receipt only: the balance is Cr → refused.
5. Cancel the bill: the pair nets to 0 → allowed.
6. A MANUAL opening balance ≠ 0 and no legs → refused; set the opening to 0 → allowed.
7. A shared (NULL-company) ledger with a balance in company B only → refused, naming B.
8. A cheque HELD ON_CLEARING → `CUSTOMER_HAS_CHEQUE_IN_HAND` (if Q3 = refuse).
9. A draft receipt → allowed with `warnings.draftDocs = 1`; posting that draft afterwards → `VCH_PARTY_NOT_FOUND`.
10. A dual party with a balance: delete the customer role → allowed, the ledger and the supplier stay (if Q2 = yes).
11. A user without the menu-10 `delete` right → the rights refusal.
12. Repeat 1, 2 and 10 for the supplier.

D2:
13. Each of the 5 codes saves, and the ledger reg type + `led_is_sez` are right (SEZ → REGULAR + true).
14. `Regular` / `unregistered` / ` sez ` → stored as the code.
15. `test` → `CUSTOMER_GST_TYPE_INVALID`.
16. REGULAR without a GSTIN → `*_GSTIN_REQUIRED`.
17. UNREGISTERED with a GSTIN → `*_GSTIN_NOT_ALLOWED` (Q5).
18. OVERSEAS with state 33 → `*_OVERSEAS_STATE`; with state 96 → 200 and the ledger UNREGISTERED.
19. Supplier GSTIN 33… with state 29 → `PURCHASE_SUPPLIER_GSTIN_STATE_MISMATCH`.
20. A sale bill on an SEZ customer: `sb_cust_gst_type = 'SEZ'`.
21. A purchase invoice on a COMPOSITION supplier: the tax mode is unchanged.

D3:
22. Create and update a customer: `cusBilledDate` / `cusBilledCount` are unchanged.
23. Post bill 1: the count is 1 and the date is the bill date.
24. Post bill 2 on an earlier date: the date stays the later one and the count is 2.
25. Cancel bill 2: the count is 1.
26. Void bill 1: the count is 0 and the date is NULL.
27. Amend a bill to another customer: both customers are right.
28. The lookup's `billed_date` text agrees.

D4:
29. Dual party: the customer edits phone1 + addr1 → `sup_phone`, `sup_addr1`, `led_phone1`, `led_addr1` are all changed; `partySync.fields` lists them; 3 audit rows (customer, supplier, ledger).
30. The supplier edits the GSTIN → the customer and the ledger follow.
31. The customer edits its credit limit → the supplier row is not written (`sup_modified_on` is unchanged).
32. A rename to a name another ledger holds → `CUSTOMER_NAME_EXISTS`, and none of the 3 rows is changed (rollback).
33. A 250-char address → refused by the DTO at 200.
34. A company change on a party with legs → `*_COMPANY_HAS_BOOKS` (Q7-a); on a party with no books → all 3 rows move.
35. A single-role customer edit → no supplier write, and `partySync.otherRole = null`.
36. The notes of a shared party do not reach `led_remarks`.

Clean-up migration:
37. After the dry-run, no `cus_gst_type` / `sup_gst_type` outside the 5 codes.
38. The per-word counts match §3.2.
39. The listed contradictions are reported, not changed.

**Qt (qt-tester, offscreen, live :3011):**
- the Customer and Supplier GST Type combos show 5 words and save codes; an old row prefills;
- the GSTIN fetch fills the code;
- Unregistered skips the GSTIN check; Overseas forces state 96;
- the shared-party note shows only for a dual party; the toast after save;
- delete from the list: refused with the balance wording; warnings in the confirm; allowed when settled;
- quick-add sends a code;
- the Ledger Statement "Show deleted" opens a deleted party's statement;
- the party-role-link prefill uses codes.

**Web (web-tester):** the same list on the React customer and supplier masters, quick-add and the Ledger Statement. The customer save payload carries no `cusBilledDate`.

## 8. Open questions for the user (business only)

1. **Supplier delete:** should the same rule hold? *Recommend: yes, refuse while the balance ≠ 0.*
2. **Dual party:** when you delete only the customer side and the supplier stays, may it go through even with a balance? The ledger and its balance stay with the supplier. *Recommend: yes.*
3. **Cheque not yet cleared** (held or deposited, posted on clearing): should it block the delete? It is outside "balance ≠ 0", but clearing it later would post to a deleted party and get stuck. *Recommend: yes, block it.*
4. **After a delete,** should the Ledger Statement still open that party (read-only, "Show deleted" tick)? *Recommend: yes.*
5. **Unregistered or Overseas with a GST No typed in:** refuse the save? (1 live customer, MADHAVAN, is like this.) *Recommend: refuse; the clean-up leaves the old row as it is until its next save.*
6. **Overseas party:** state must be 96 "Other Country", and no GST No? *Recommend: yes (the GST portal's rule for exports and imports).*
7. **Company / branch on a party that is both customer and supplier:** (a) keep them the same on both and refuse a company change once the party has entries in its company; (b) lock them on a shared party; (c) let each side differ. *Recommend: (a).*
8. **A user who may edit customers but not suppliers:** their edit still changes the supplier's name, address and GST details. OK? *Recommend: yes, it is one party, and the history shows who did it.*
9. **Last billed:** you decided "set on bill post, fixed on cancel". Storing it means every bill re-saves the customer record. With offline branches, that can overwrite an office edit of the same customer (credit limit, phone) at sync, and each branch counts only its own bills. Working it out from the bills each time it is shown gives the same answer with neither problem. Which? *Recommend: work it out when shown (option B).*
10. **Supplier "last billed date":** make it the last posted purchase invoice the same way, later? *Recommend: yes, in phase 2; for now supplier saves just stop changing it.*

## 9. Build order (agents)

0. **Wait** for the running nestjs-backend fix batch (customer.service.ts, customer DTOs, customer-detail.lookup.ts) to land. Re-read those files. The new code keeps F3–F10 and the DTO limits, and sets the synced keys' `@MaxLength` to §3.4's widths.
1. **db-schema.** One migration, dry-run shown, applied only after the user's OK:
   - D2-4 data clean-up + ledger reg type / SEZ, with the contradiction report;
   - D2-5 seed state 96;
   - D3-3 index `ix_sb_cust_posted` (parent + partitions);
   - D3-4 back-fill (A) or one-time NULL / 0 (B);
   - the §4.2 wording rows in `Error_Messages.json` (version bump).

   There is no `db_table_master` row, no menu, grid, ui_table or dropdown in P1. P2 (later): `ck_cus_gst_type` / `ck_sup_gst_type` + `cus_gst_type` NOT NULL DEFAULT 'UNREGISTERED`; grids D1-8 / D4-9 at the next free grid ids (196 / 197 free today; re-query at build, parallel workstreams take ids) and a menu id from 297 up.
2. **nestjs-backend:**
   - new `src/common/party/` (`party-gst-type.ts`, `party-detail.map.ts`, `party-sync.service.ts`, `party-balance.service.ts`, and `party-billed.service.ts` only under A, `party.module.ts`);
   - edits to `sales/customer/{customer.service.ts, customer.controller.ts, types/customer-error-codes.ts, types/customer-api.types.ts, dto/customer-response.dto.ts}`;
   - edits to `purchase/suppliers/{suppliers.service.ts, suppliers.controller.ts, types/supplier-error-codes.ts, types/supplier-api.types.ts, dto/supplier-response.dto.ts}`;
   - `master-lookup/lookups/customer-detail.lookup.ts` (B), or `sales/bill/bill-lifecycle.service.ts` postCore / cancelIn hooks + `cloud-sync/cloud-sync.service.ts` post-run recompute (A);
   - `reports/ledger-statement/{ledger-statement.service.ts, dto/ledger-statement-query.dto.ts}`.

   It proves every route live on :3011 and returns the exact keys of `delete-check` and `partySync`.
3. **qt-screen** and **react-screen**, in parallel, against the live keys (§5, §6).
4. **api-tester** (§7 API, 1–39), then **qt-tester** and **web-tester**.
5. **reviewer** over the diff, against this plan: offline items §3.3; that the field map is the only map; that no save writes a billed column.
6. **docs-notes**: notes file for Prathap, memory notes (party-dual-role, customer-creates-ledger), session-resume.
