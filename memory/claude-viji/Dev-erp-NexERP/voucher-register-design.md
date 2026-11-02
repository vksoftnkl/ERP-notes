---
name: voucher-register-design
description: "Voucher Register — ONE file share/accounts/voucher_register.md (rev 2, 2026-09-25, nothing run): one register for all accountant vouchers + ONE posting routine Receipt/Payment also call; decisions A–E approved; the rev 1 set was deleted"
metadata:
  node_type: memory
  type: project
  originSessionId: d02cda15-8dc1-468c-85eb-fb56e4c24d96
  modified: 2026-09-25T12:46:23.916Z
---

**Single source:** `/home/vk/Shared/Prathap/accounts/voucher_register.md`, with its screen `voucher_register_ui_mockup_rev2.png/.py`.

The rev 1 set (register_flow.md, plan-backend-register.md, plan-qt-register.md, and two rev 1 PNGs) was **deleted 2026-09-25** at the user's request. Nothing from any version has been run on the box. The user wants one .md per design, not a doc set.

**Model (user chose 2026-09-25, "go with one register"):**
- One register screen covers Journal, Contra, Debit Note, Credit Note, Purchase (acct), Sales (acct), and Receipt/Payment in voucher mode.
- **One server posting routine** is also called by Receipt (99) and Payment (100). Those keep their own screens, because tenders, cheques and the bill-wise grid are workflows.
- A form gets its own screen only if it has objects the legs model lacks (tenders, instruments, items, stock).

**Decisions approved on the PNG:**
- A: avh_party_id becomes nullable (Contra has none, a Journal may have several). It is NOT NULL on the live table.
- B: the GST band generates the tax legs, and the client sends only its typed lines.
- C: cancel writes a reversal `<refno>/R` under a hidden `Rev` type, dated the cancel day.
- D: menus 101 DN, 102 CN, 103 Jrl, 104 Con, 163 (renamed Purchase (Accounting)) and three new menus open the screen with the type fixed. A new Voucher Register menu shows every permitted type.
- E: rights per voucher TYPE via `vchr_menu_id` + `user_menus`; the band is filtered and the server checks every call. The user asked "how do I stop a user doing purchase/sales".
- TDS is in scope (S2 closed).
- Still open: S20 (cash ledger per branch or per counter), S1 cost centres. There is no amend in v1 (R1). R2 proposes the same Rev fix for the sale bill's D3.

**Live facts it rests on (verified 2026-09-25):**
- `reverseLegs` now numbers every reversal in the `Rev` type (Prathap fixed D3 on 2026-09-25); it used to draw from the original type.
- `allocateVoucherNumber` creates the seq rows on first use.
- `loadRights` reads only 5 flags.
- `menu_master.menu_verbs` exists.
- There are NO "Expenses"/"Income" groups, only the Tally set.
- `accounts.tds_rates` is not on the box (it is in payment plan §2.9).
- `tax_rate_ledger` gives the tax leg ledgers.
- The migration file number is 39.
- The routes are an own `/vouchers/*` family.

Related: [[accounting-scope-register]], [[payment-design]], [[receipt-design]], [[new-module-own-urls]], [[accounts-screens-own-ui-rule]].

**Hand-over 2026-09-25:**
- `share/39_voucher_register.sql` is WRITTEN and dry-run clean twice on a freshly rebuilt erp_dry (the schema plus the live menu/group/type/role rows). Its guard was tested.
- It does NOT insert `Rev`, which already exists (Prathap's D3 fix). It only sets Rev's rule columns.
- `share/HANDOVER_2026-09-25.md` lists what's for Prathap.
- Open C2: the PNG says `<refno>/R` dated the cancel day, but the live reversal is `rev000NN` dated the original. The doc recommends keeping the live rule; the user hasn't answered.

**APPLIED 2026-09-25 (verified live, evening):** Prathap ran 39 as prisma migration `20260925160000_voucher_register`. The box now has:
- types 24 Jrl, 25 Con, 26 DrN, 27 CrN, 28 PurA, 29 SalA, 30 RcpV, 31 PmtV, with Rev 23 given its rule columns;
- menus 259 Sales (Accounting), 260 Receipt Voucher, 261 Payment Voucher, 262 Voucher Register, pinned by id; 163 renamed Purchase (Accounting); 101–104 visible;
- avh_party_id nullable, the 5 roles, and vtx_itc_eligibility.

His version is stricter than 39 (menu ids pinned; groups matched against the shared chart only; safe on a fresh DB).

Also done: `loadRights` now reads view/create/edit/delete/print.

NOT yet built (checked in the source snapshot):
- the `/vouchers/*` module and the posting routine;
- `accounts.tds_rates`.

Handover §2 (the four sale-bill suspects) is all fixed:
- CHG-TAX: chgTaxRate added and ledGstRate re-echoed;
- T-VOID: findDocumentTenders filters voided rows and tdIsVoided is on the payload;
- E-WAY-DB: the band in the body wins;
- the no-customer code is now SALES_CUSTOMER_REQUIRED.

**C2 DECIDED 2026-09-25:** keep Prathap's live reversal (type Rev, series rev000NN, dated the ORIGINAL). The PNG and the doc are updated to match.

**Qt plan written** as §9 + §11.2 of voucher_register.md. Main points:
- The list uses TxnMainView like the Receipt, with `TxnDocTypes::voucher(typeCode)` + `voucherRegister()`, and nine fm.reg lines.
- `/vouchers/validate` is the screen's calculator: a 400 ms debounce, a generation counter, and derived rows painted grey and locked. No tax math in C++.
- Rights come from `/types`, not from ctx.
- Ledger picker = a popup over `/vouchers/ledger-pick`.
- One addition to the approved PNG: a narrow TDS tick column (§9.8). **Added to the PNG on 2026-09-25.**
- It waits for Prathap's `/vouchers/*` module, which is not built yet.

**Handover updated (evening):** `share/HANDOVER_2026-09-25.md` now lists only what remains for Prathap:
- `accounts.tds_rates`;
- the `/vouchers/*` module + posting routine;
- ledger-pick must return `isTdsApplicable`, and `tdsBase` defaults to `led_is_tds_applicable` (both added to doc §6.2 and §7).

Suggested order for him: the first five routes, so the client can start, with `/validate` last of those five.

**`share/40_tds_rates.sql` written 2026-09-25**, dry-run clean on erp_dry with the live ledgers. It creates `tds_rates` (with EXCLUDE, an index and the sync triggers), narrows the ledger TDS columns to 16, adds `ck_led_tds_deductee`, adds `ck_led_tds_pair` NOT VALID, and seeds 7 rows for 2026-27. The CA must confirm the seed, because the Income-tax Act 2025 renumbers the sections. It is in the handover. **Not yet run on the box** (as of this note).
