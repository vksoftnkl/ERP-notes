---
name: transport-band-shared-table
description: the transport band is ONE shared table (txn_transport_detail, 33), not 22 columns repeated on four documents; direction decides which end is ours
metadata:
  node_type: memory
  type: project
---

**Decided 2026-09-21.** The user asked whether the band should be common
"because many companies are point of sales so all columns will be null most of
the time". I analysed it, told them the NULL reasoning does not hold, gave the
reason that does, and they said go.

**REVISED the same day, at the user's call — the split is not "everything in
the shared table".** Crew and vehicle stay on the DOCUMENT; only the addresses
and the carriage are shared:

| on all four documents (`sb_`/`sdc_`/`sr_`/`sdr_`) | in `txn_transport_detail` |
|---|---|
| `driver_id`, `supervisor_id`, `loadman_id uuid[]`, `vehicle_id`, `vehicle_no` | `from_*` / `to_*` + `direction`, `transporter_id/name/gstin`, `lr_no`, `lr_date`, `distance_km`, `transport_mode` |

**Why:** crew and vehicle are "who of ours did it" and exist whenever goods
move **even with no consignment** — a home delivery has a driver and a van and
no transporter, no LR, no e-way bill. Keeping them in the band would force a
row there just to record who drove, which is the COMMON case for a POS company.
The band is then genuinely absent for POS.
**NOT duplicated** — the user first proposed "a copy of vehicle_no / vehicle_id
by doc wise"; I pushed back because two copies of one fact diverge and no
report knows which to trust (same failure as the cancel reason on both the
header and `txn_status_log`). One home per fact: the e-way builder reads the
vehicle off the document, which it has loaded anyway, and the rest from the
band.
**So `30 §1b2` does NO migration** — the bill's five simply stay and gain the
FKs they never had. The guarded-DO migration block was removed.
**The price is drift**, and it had already happened (`supervisor_id` on the
bill alone, `vehicle_id` missing from the DC return). A **conformance test**
asserts all 20 columns exist across the 4 tables — plan-backend §12, passes.

**`public.txn_transport_detail` (`33_txn_transport_detail.sql`)** — 43 columns,
partitioned by `ttd_acc_year`, 1:1 with its document via
`ux_ttd_doc (ttd_doc_type, ttd_doc_id, ttd_acc_year)`. Carries ship-to /
dispatch / pickup / receive / transporter / LR / distance / mode for **every**
goods-moving document. `sale_bill`,
`sale_dc`, `sale_return` and `sale_dc_return` carry NONE of it.

**Why — NOT storage.** I measured it: 22 NULL columns cost ~**8 bytes a row**
(PostgreSQL keeps NULLs as one bit in a row bitmap), so half a million POS
bills a year would have cost ~4 MB. The real reason is that the band was
22 columns × 4 documents = **88 definitions and they had ALREADY drifted** —
`vehicle_id` was missing from the DC return, `supervisor_id` existed only on
the bill. Every future e-way field would have been four ALTERs.

**FROM / TO are POSITIONAL; `ttd_direction` says what they mean.** OUTWARD
(bill, challan): FROM = our godown, TO = the party. INWARD (both returns):
FROM = the party's pickup, TO = our godown. So
`gdr_dispatch_* ← ttd_from_*` and `gdr_ship_* ← ttd_to_*` with **no
per-document branch in the payload builder** (gst/gsp_flow.md §6a).

**Same pattern as `txn_charge_detail` / `txn_status_log`**, and it pays the
same price: a polymorphic child cannot FK to four partitioned parents, so the
service is what stops a row outliving its document. The transporter / vehicle /
driver / supervisor / godown FKs all moved ONTO this table.

**`30 §1b2` keeps the bill's five** (no migration; the 24 test-box bills with a
loadman array are untouched) and drops the ship/dispatch/transport block defensively in
case an earlier draft of 30 ran somewhere.

**Verified on `erp_dry`:** 29 clean, 33 clean (4 partitions, 43 cols), all 20
crew/vehicle columns present across the 4 documents, zero band columns left on
any of them, §1b2's drop block confirmed
standalone. NOTE `30` is non-idempotent on an already-dirtied database (its
`ADD CONSTRAINT`s lack IF NOT EXISTS) — fresh DB runs once, clean.

**Client:** the payload is a nested `transport` object with `direction`, not
header fields — plan-qt-sales §1.6. One widget serves all four screens.


**The register keeps NO transport columns (confirmed 2026-09-21).** Three
layers, no fact stored twice for the wrong reason: document + `ttd_` = the
editable working copy; `acc_voucher_doc_register` = frozen at post, ADDRESS
blocks only (both the e-invoice and the e-way need them); `gdw_` = what was
declared to the portal, and transport is e-way-ONLY so it goes straight there,
reaching the addresses via `gdw_gdr_id`. Transport columns on `gdr_` would be
dead weight on every e-invoice-only document.
**`gdr_dispatch_*`/`gdr_ship_*` DO duplicate `ttd_from_*`/`ttd_to_*` on
purpose** — `ttd_` moves on amend, `gdr_` must keep what was declared, and the
GSTR-1 builder reads `gdr_` without touching the sales tables. Written into
`gst/gsp_flow.md` §6a so it is not "tidied" away later.


**LOCK MODEL — FINAL (user, 2026-09-21): "after generation of IRN / eway bill
we should stop editing" + "no amendment after eway and einv generation only
cancel possible".** TWO lock points, not three:

| # | when | frozen |
|---|---|---|
| 1 | POST | document content; the band stays OPEN (nothing declared yet) |
| 2 | **IRN or EWB generated** | **everything incl. the band. NO AMEND — cancel only** |

Either generation trips lock 2 (the IRN payload carries ShipDtls/DispDtls, so
an IRN alone freezes the addresses). No override, no right unlocks amend —
`rights.amend` says the USER may, lock 2 says this DOCUMENT may not. Correct a
declared document by CANCELLING inside the portal window and raising a fresh
one; past it, a credit note.

**A later vehicle change is a PORTAL operation, not an edit** —
`/gst/ewaybill/vehicle` → `gdw_`. It never re-opens the band. So: document +
band = what we PREPARED to declare (frozen at lock 2); `gdw_` = what we
DECLARED and every update since (the only thing that moves).
**READ RULE, easiest thing to get wrong: once an EWB exists the CURRENT vehicle
is `gdw_vehicle_no`, not the document's.** The document keeps the original.

Derive both — lock 1 `*_status='POSTED'`, lock 2 `gde_status='GENERATED' OR
gdw_status='GENERATED'`. Never a stored flag. Codes: `SALES_IRN_LIVE`,
`SALES_EWB_LIVE`, `GST_DECLARED_LOCKED`, `SALES_DAY_CLOSED`.
Written in HANDOVER §1.3/§1.3a/§2.12/§9 + §4/§5/§6, plan-qt §1.6a + §4/§5/§6,
gst/gsp_flow §6b.

**Per document:** challan — never an IRN, so only the EWB trips lock 2; has a
THIRD editable flag `purpose` until then. DC return — no `/amend` at all ever,
no IRN; band is its only editable surface between the locks. Sale return — IRN
is a CREDIT NOTE (CRN); `SALES_CN_APPLIED` + `GST_CREDIT_NOTE_CUTOFF` on lock 1.

**The STRONGEST justification for the separate table, found 2026-09-21 when the
user proposed locking the document at IRN/EWB** (gst/gsp_flow.md §6b): the band
has a **different mutability lifecycle** from the document. A Part-B vehicle
update is normal for the whole life of an e-way bill, while the invoice must
not move at all once an IRN exists. As columns on `sale_bill`, "lock the bill"
and "let them change the vehicle" would be the same sentence.

**THREE lock points:** POST freezes the document's content; IRN freezes it
harder (no amend, cancel-in-window then credit note) while the band stays fully
open; EWB freezes the band's Part-A (`from_*`, `to_*`, `distance_km`) leaving
Part-B (`vehicle_no`, `mode`, `lr_no/date`, transporter) open until
`gdw_valid_upto`. So after an IRN the bill screen is read-only EXCEPT the band,
and after an EWB the band is read-only EXCEPT Part-B. **Derive all three from
`*_status` / `gde_status` / `gdw_status` — never a `ttd_is_locked` column.**

**But the band CANNOT generate the IRN.** It is the transport source, not the
whole source: the IRN is mostly item lines (`acc_voucher_doc_detail` / `vtx_`,
15 item columns incl. HSN and the four tax rates) plus values from `gdr_`; the
band supplies 2 of its 6 blocks. The EWB is roughly half band. Build both from
**`gdr_` + `vtx_` + the band**.

**And `gdr_` does NOT become redundant under a lock-on-IRN model** — I had
justified it as a frozen snapshot, which is the WEAKER reason. The real ones:
`gdr_source_module` spans EIGHT modules (SALES, PURCHASE, INVENTORY, ACCOUNTS,
SERVICE, JOBWORK, POS, OTHER) so GSTR-1/3B read one table instead of UNIONing
every document family; and it holds seven classifications DERIVED at post that
exist nowhere on the document (`tran_nature`, `doc_flow`, `doc_sign`,
`doc_type`, `supply_nature`, `is_einvoice_applicable`,
`is_ewaybill_applicable`). It is a normalising projection.


**The lock rules are WRITTEN (2026-09-21):** `HANDOVER_endpoints.md` §1.3
(`locks.editable { document, transportBand, transportPartB }` + `ewbValidUpto`),
§1.3a (the three points, the refusal codes, "derive never store"), §2.12
(`PUT /bills/transport`), §9 (SALES_IRN_LIVE, GST_EWB_PARTA_LOCKED,
GST_EWB_EXPIRED, SALES_DAY_CLOSED); and `plan-qt-sales.md` §1.6a + §10.

**The route that had to exist:** `PUT /bills/transport` — the band needs its own
verb because `/bills/amend` is refused at lock 2 while the band is still open.
It does NOT bump `sb_revision_no` (the document did not change), writes a
`txn_status_log` row `TRANSPORT_EDITED`, and after an EWB accepts Part-B only.
Same route on challans and both returns, `direction` INWARD on the returns.
The client greys from `locks.editable` and NEVER recomputes the rule.


**The lock table is DIFFERENT per document — do not copy the bill's (written
2026-09-21, HANDOVER §4/§5/§6 + plan-qt §4/§5/§6):**

| document | lock 1 POST | lock 2 IRN | lock 3 EWB | editable after POST |
|---|---|---|---|---|
| sale bill | content | yes (INV) | Part-A | band |
| sale return | content | **yes, but CRN** (credit note) | Part-A (inward) | band |
| delivery challan | content | **NEVER — a challan carries no IRN (Q8)**, `gde_*` stays NA | Part-A | band **+ purpose** (convert-purpose) |
| DC return | content, **and there is no /amend at all, ever** | **never** (CHL — moves goods, raises no debt) | Part-A (inward) | band ONLY |

So the challan's `locks.editable` has a FOURTH flag, `purpose`, false once any
line is billed or returned. The DC return's `editable.document` is false from
POST and never true again — no amend flag to consult. The DC return is the
clearest illustration of why the band is a separate table: it is the only
editable surface a posted one has.

Extra non-GST refusals on the sale return's lock 1: `SALES_CN_APPLIED` (the
credit note is already set off or refunded — reverse with a document) and
`GST_CREDIT_NOTE_CUTOFF` (past the statutory cut-off for the original invoice's
FY, date from `fn_statutory_limit`, never a constant).


**`gdr_ship_*` / `gdr_dispatch_*` DROPPED (2026-09-21, 32 §2b).** Reversing what
I said earlier in the same session — I had given three reasons to keep them and
TWO WERE WRONG: (a) "the band can diverge from what was declared" died when the
user set the lock rule (the band freezes at generation, so it IS the declared
copy); (b) "GSTR-1 reads gdr_" is wrong for these — **ship-to and
dispatch-from appear in NEITHER GSTR-1 NOR GSTR-3B**; GSTR-1 B2B needs
`gdr_party_*` (13 cols), which STAYS. Only the 8-module reason survived, and
it is handled by widening the band instead.

**`ck_ttd_doc_type` now admits `STOCK_TRANSFER`** and widens per module
(`PURCHASE_BILL` / `PURCHASE_RETURN` when built). That closed a REAL GAP found
on the way: `stock.stock_voucher` carries only `svh_from_godown_id` /
`svh_to_godown_id` — no vehicle, transporter, LR or distance — so an
inter-branch transfer that needs an e-way bill had nowhere to record a vehicle.

**`schema/05m_migrate_doc_register.sql` inserted all 18 by name** — updated,
and it must run BEFORE 32 or not at all. `schema/05` keeps them (it is what was
deployed) with a forward pointer to 32.
