# Notes 72 — Company + Branch masters: client + backend + live audit (2026-10-01)

Scope: Company master (list grid 52, `/company-masters/*`) and Branch master (list grid 56,
`/branch-masters/*`). Both sides were read: the Qt forms, the NestJS DTOs and services, and the live
tables `companys` and `branch_master`. The findings below were then checked live with two `ZT-` rows:
`ZT-CO-AUDIT` (01a0f5e5-c0bc-7d65-bcf0-bd4c5426688a) and `ZT-BR-AUDIT` (01a0f5e5-c0e7-7052-81c7-afc886f47b51).
Both rows are soft-deleted now. The client-side fixes are already in the Qt tree. This file lists
only the items that are yours.

## A. A new company cannot be used (blockers)

| # | Finding | Ask |
|---|---|---|
| A1 | **Nothing writes `fiscal_years`.** The year combo after login (`GET /master-lookups/fiscal-years/by-company`) lists only these rows. `/company-masters/create` never seeds one, and no route can add one. A company created from the screen therefore logs in with an EMPTY year combo, so it has no accYear and every document screen is dead. (The receipt/opening guards allow a missing row, so the posting side is not what breaks.) Live: ZT-CO-AUDIT has 0 rows, and so does LEAPSWITCH NETWORKS on the box. | **`43_fiscal_year_seed.sql` (share root)** backfills every live company that has no row: one OPEN, current April–March year. It is idempotent, dry-ran on erp_dry, and the live preview = 1 row (LEAPSWITCH, 2026-2027). The going-forward part is yours: the same derivation inside `createCompany()`'s transaction. The TypeScript is at the bottom of the .sql. Later, a `/fiscal-years/get\|save\|close` family is needed for a year screen. |
| A2 | **`comp_aato_class` cannot be set.** The column is NOT NULL with default `LE_5CR`. `statutory.service.ts` reads it for e-invoice applicability. It is not in `SaveCompanyMasterDto`, so every company stays `LE_5CR`. A company above ₹5 Cr is then wrongly left out of the e-invoice rules, or the reverse. | Add `compAatoClass` (enum `LE_1_5CR / LE_5CR / LE_10CR / GT_10CR`) to the DTO, the optional-fields list and `toPayload`. Tell us when it is in, and the client adds a combo on the Tax tab. |
| A3 | **`comp_dc_purposes` cannot be set.** The bill and DC services read it, but it is missing from the DTO and the payload. | Add `compDcPurposes: string[]` the same way. |
| A4 | `company_aato` (the AATO for each year, COMPUTED/DECLARED) has no route at all. | Low priority. It needs a route only when A2 must be derived from turnover rather than picked. |

## B. Lifecycle / guards (defects, all live-verified)

| # | Finding | Ask |
|---|---|---|
| B1 | **No restore route** for company or branch. A DELETE on a deleted row now answers 404, which is correct. The item masters got `/restore` in notes 70/71. | `POST /company-masters/restore?compId=` and `POST /branch-masters/restore?brId=`, same shape as `/item-brands/restore`. Until then, the client's "Show Only Deleted" view says that restore is not available. |
| B2 | **A company with a live branch deletes with 200.** Live: ZT-CO-AUDIT was deleted while ZT-BR-AUDIT was still live, and the branch stayed `brIsDeleted=false` under a deleted company. Ledgers, groups, customers and settings are not checked either. The FKs are RESTRICT, but only on a hard delete. | Refuse with 409 while live branches exist, or ledgers / vouchers. Also refuse deleting the **default** company, because the delete clears `comp_default` and leaves no default. |
| B3 | **Branch delete has no guard**: not for the default branch, not for the last branch of a company, and not for a branch with stock, documents, users or devices. | Refuse with 409 for the default branch and for a branch with any `stock_balance` row or any document. Your call on users/devices. |
| B4 | **An update can move a branch to another company.** Live: `brCompId` was changed from ZT-CO-AUDIT to Acme and back, 201 both times. All of the branch's documents would then sit under the wrong books. | Refuse a changed `brCompId` on update (400), at least once the branch has any document. The client now locks the Company field in edit mode. |

## C. Contract / dead columns (decisions)

| # | Finding |
|---|---|
| C1 | `comp_tds_applicable` is a **text** column and is missing from the DTO. The client's "TCS/TDS Applicable" box writes only `comp_tcs_applicable`. Either make it a boolean with a `compTdsApplicable` key, or drop it. |
| C2 | `comp_books_lock_date`, `comp_fin_year_from/to` and `comp_books_begin_from` duplicate `fiscal_years` (`fy_lock_date`, begin/end/books-begin), and no service reads the companys copies. The client offers FY From/To/Books Begin only as input for the A1 seed. It does not offer the lock date, because the lock belongs to the year. Decide whether the companys columns stay. |
| C3 | `comp_gst_reg_type` / `br_gst_reg_type` are free text, and the box holds both `Regular` and `REGULAR`. The client combo offers Regular/Composition/Unregistered/Consumer/SEZ/Overseas, but Consumer and Overseas make no sense for your own registration. Give us the enum: suggested `REGULAR / COMPOSITION / UNREGISTERED / SEZ` with an upper-case normalise and a CHECK. We will then cut the combo. |
| C4 | These columns are stored, but **no service reads them**: `br_rounding_mode/value`, `br_bill_prefix`, `br_invoice_series_prefix`, `br_pos_type`, `br_terms`, `br_bill_greeting`, `comp_prefix_code`, `comp_price_fixing`, `comp_bill_greeting` (print-render aside). Either wire each one to what it claims to drive, such as bill rounding or the number-series prefix, or say that they are informational. Today an operator can set "Rounding Mode = ROUND DOWN" and nothing changes. The client does not offer `br_pos_type`, because its values are undefined (the box has `Retail POS` and nulls). |
| C5 | `compAuthorizeSignature` is a text column with no upload handling. The client cannot offer it until it works like the item image: base64 in, URL on `/get`. |
| C6 | GSTIN lookup: the GSTIN field's fetch button has no route behind it. `gst_company_credential` knows a `GSTIN_VERIFY` service, but no endpoint exposes it. |
| C7 | The server does not check that the GSTIN's first two digits match `*StateCode`, or that characters 3-12 match `*PanNo`. The client now checks both and fills a blank PAN from the GSTIN. A server check would cover imports and React. |

## D. Done on the box by us (2026-10-01, through `/grid-details/create` with column ids, no columns added)

- **Grid 52** (companies): `WHERE comp_is_deleted = icomp_is_deleted`, `ORDER BY comp_default DESC, comp_name`.
  Search flags were set on code / short / name / legal name / GSTIN. The previous SQL had `= false`
  hard-coded, so "Show Only Deleted" did nothing.
- **Grid 56** (branches): `WHERE BR.br_is_deleted = ibr_is_deleted`, `ORDER BY CMP.comp_name, …`. Search
  flags were set on company / code / name / short / city, and Code is now visible. Before this, no
  column was flagged, so search never matched.
- The old definitions are saved beside this file as `grid52_before.json` and `grid56_before.json`.
  Verified: false → 5 / 7 rows, true → 1 / 0 rows, and search "Acme" → 1 / 3 rows.

---

## E. Re-check after your changes (2026-10-01, live + source)

Tested live with ZT- rows (ZT-CO-72, ZT-CO-72B, ZT-BR-72/72B/72C, all soft-deleted afterwards), and the
new source was read. **Everything in A–C is in except one defect (E1).** The Qt client has been moved
to the new contract.

| Item | Result |
|---|---|
| A1 | Create seeds an OPEN, current year (ZT-CO-72 → 2026-2027, books 2026-08-15); LEAPSWITCH backfilled. ✔ |
| A2 / A3 / C1 | `compAatoClass`, `compDcPurposes`, `compTdsApplicable` round-trip; bad enum and `[]` → 400; an update that omits them keeps them. ✔ |
| B1 | Both `/restore` routes work; restoring a live row → 409. ✔ |
| B2 | Company with a live branch → 409 "Used by 1 live branches". ✔ |
| B3 / B4 | As designed in the source: an unused non-default branch may be deleted or moved, and the only branch may be deleted even when it is the default. Accepted. The client still locks Company on edit, which is stricter and harmless. |
| C3 | Reg type is now an upper-case enum (REGULAR/COMPOSITION/UNREGISTERED/SEZ), normalised on save. ✔ The client combos now send and load the codes. Before this change, editing any company or branch would have shown a blank required combo. |
| C7 | GSTIN state / PAN checked on both masters. ✔ |

### E1 — the first fiscal year is not forced to April–March (defect)

`resolveFirstYear()` takes `compFinYearFrom`/`compFinYearTo` as given (only to > from and at most one
year), and names the year `${begin.year}-${end.year}`. Live: ZT-CO-72B was sent 2026-03-20 → 2026-03-28
and got a fiscal_years row **`2026-2026`, 20–28 Mar**, OPEN and current. Every accYear check
(`isValidAccYear`, `ck_caa_fin_year`, voucher series) expects `YYYY-YYYY` with the second year one greater
than the first, so that company's year is unusable.
**Ask:** refuse (400) a `compFinYearFrom` that is not 1 April, or a `compFinYearTo` that is not the
following 31 March. Alternatively, derive the year from the date as 43 does. The Qt form now refuses
both, so only API callers can hit it. The bad row is left on the box under the deleted ZT-CO-72B as
evidence; delete it at will.

### E1 — fixed, verified 2026-10-01

A From date other than 1 April, or a To date other than the following 31 March, is now a 400. A To date
alone, or no dates at all, gives 2026-2027. A mid-year books-begin (15 Aug) is kept. The ZT-CO-72B row
was repaired to 2025-2026. The Qt form's own collectJson body (62 keys) created, updated and read back
with no mismatch; branch create, delete, restore and delete were run from the form body. **Notes 72 is closed.**
