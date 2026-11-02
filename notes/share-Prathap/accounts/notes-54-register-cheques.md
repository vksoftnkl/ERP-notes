# notes (54): the Receipt Voucher takes cheques (register instruments)

2026-09-26. The user confirmed the design on `register_cheques_ui_mockup.png`
(Q1–Q7). The full contract is **`register_cheques.md` §3**. This is the build list.

**Why:** a cheque is received before anyone knows which bank it will go into. The
register refused Cheques In Hand (S16), so a salesman's collection run could not take
cheques. Now each customer line carries its instrument. The server generates the
Dr legs from the tenders' ledgers, and a cheque becomes a HELD row in menu 51. The
operator still never types Cheques In Hand.

## Build list

1. **`acc_voucher_types.vchr_instruments boolean`** (true for RcpV), exposed as
   `instruments` on `/vouchers/types`.
2. **`GET /vouchers/instruments?companyId`**: the usable tenders
   (no TEMP_CR / CREDIT / LOYALTY / RRN), each with its type, ledger, and
   `isCash` / `isCheque` / `needsRef`.
3. **The payload `lines[].instrument`**
   `{tenderId, refNo, instrumentDate, bankName, cheque{drawerName, bankBranch, ifsc, micr}}`.
   The names are the Receipt's DTOs, so reuse its writer. Also `header.employeeIds` →
   `avh_employee_id`.
   New codes: `VCH_INSTRUMENT_NOT_ALLOWED`, `VCH_INSTRUMENT_TENDER`, `VCH_CHEQUE_DETAILS`,
   `VCH_CHEQUE_MOVED`.
4. **`/validate`**:
   - one generated leg per instrument line (`source INSTRUMENT`, role = tender type, the tender's ledger);
   - a post-dated cheque marks its leg and its party line `postDated/postsOn`;
   - `derived.postDated[]`; `totals` = today only.
5. **`/post`**:
   - an `acc_tender_detail` row per instrument;
   - an `acc_pdc_register` HELD row per cheque (`apd_party_id` = the line's party);
   - `abj_cheque_id` on that line's allocations;
   - a post-dated cheque gets its own voucher on its date (`avh_against_voucher_id` → today's), in the same transaction, the Receipt's way.
6. **`/cancel`**: refused with `VCH_CHEQUE_MOVED` unless every cheque is HELD. Otherwise the
   HELD rows → CANCELLED and the post-dated voucher(s) reversed too.
7. **`/get` and the draft** return and keep the instruments, plus `pdcId/pdcStatus` and the post-dated voucher refno.
8. **Server tests** (register_cheques.md §5):
   - three customers: a cheque today, cash, a post-dated cheque;
   - bounce one: only its customer's bills reopen;
   - deposit one, then cancel the voucher: refused.

Menu 51 needs no server change.

## After it lands

The Qt side is coded against the **live** Swagger and one real response
(register_cheques.md §4), then `tests/register/run.sh cheques`. Please note in
this file any name you change from §3.

---

## Checked 2026-09-26 (migration `20260926120000_voucher_register_instruments`)

**Done and verified live,** first through `/validate` / `/post` / `/get` / `/cancel`, then through the Qt
screen: `tests/register/run.sh cheques`, 23/23. rcv00022 was posted and cancelled; test posts
rcv00020–22 are all CANCELLED, with their cheques CANCELLED and their PDC vouchers reversed.

**The names I coded to:**
- `/vouchers/instruments` answers `data.tenders[]` (not `instruments`) with `typeName`.
- A generated money leg links back to its line by `fromRows`; `lineRowNo` is null on it.
- `/get` returns `instruments[]` and `pdcVouchers[]`.
- A cancel answers `chequesCancelled` and `pdcVouchersReversed`.

**Two small asks left (neither blocks anything):**
1. **`/cheques/get` → `receiptVoucher` has no type code.** Menu 51's Enter asks
   `/vouchers/get` first to see whether the voucher is a register one: one extra call.
   A `typeCode` on `receiptVoucher` would make it one call.
2. **`/vouchers/get` header has no `employeeIds`.** A draft keeps its collected-by
   (from the stored payload), but a POSTED voucher cannot show who collected it.
   Please add `employeeIds` (+ names) to the header.
