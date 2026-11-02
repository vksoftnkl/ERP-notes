# notes (53): Receipt Voucher and Payment Voucher take MANY parties

2026-09-26, the user's decision: a salesman comes back with collections from
several customers and the office keys them as **one** Receipt Voucher; a
payment run to several suppliers is **one** Payment Voucher. Both types
become party mode **MANY**, like the Journal.

## 1. The type rows

```sql
UPDATE accounts.acc_voucher_types
   SET vchr_party_mode = 'MANY'
 WHERE vchr_type_code IN ('RcpV', 'PmtV');
```

Keep `vchr_party_side` as it is (RcpV CR, PmtV DR): it still says which side
the party lines sit on, and the payment gross-up reads it. Keep
`vchr_billwise_mode = 'DEMAND'`. As a migration, please, so it is on every box.

## 2. Bill-wise: already works per line — please confirm with a test

`voucher-derive.ts` (the MANY branch at ~line 820) already makes one party leg
per typed line whose ledger is a party or bill-by-bill. The DEMAND check
(~line 943) already demands each leg be allocated in full, and allocations
carry `lineRowNo` = the typed line. So the bill-wise part should need nothing.
Please add the case to the server tests:

- RcpV: Dr Cash 15,000; Cr customer A 10,000 (two bills 6,000 + 4,000); Cr
  customer B 5,000 (one bill). Allocations `lineRowNo` 2 and 3. It posts, each
  customer's bills settle, and `abj_party_id` is right on each.
- The same with customer B allocated 4,000 of 5,000 → `VCH_BILLWISE_SHORT`
  naming customer B only.

One question: **a customer paying on account** (no bill yet) is refused
by DEMAND today. For a collection run that is common. Either keep DEMAND strict
(the operator keys the on-account part on a separate voucher), or let an
unallocated remainder on a MANY receipt raise an ADVANCE bill for that party,
the way RAISE does. Your call; the client shows whatever `/validate` answers.

## 3. TDS on a MANY Payment: per party line (the real work)

Step 5 today reads the ONE `party` (`type.tdsMode === 'DEDUCT' && party &&
party.isTdsApplicable`). With MANY there is no header party, so no TDS is ever
worked out. It needs to run **per party line**:

- for each typed party leg whose ledger `isTdsApplicable` with a section:
  - rate: from `tds_rates` by that party's section × deductee type × date;
  - `NO_PAN` → 206AA;
  - thresholds against **that party's** annual base so far;
- base: the payment-shaped gross-up already in step 5 (the typed line is the
  NET paid), per line;
- one `CR TDS Payable` leg per party (or one summed leg with `fromRows`
  listing the lines; either is fine, but the `acc_tds_register` rows must be
  one per party, because 26Q is per deductee);
- `/validate`'s `derived.tds` becomes a list, or add `derived.tdsLines[]`, with
  one entry per party: `{lineRowNo, partyId, section, rate, base, tax,
  deducted, reason}`. The client paints whichever you return;
- cancel reverses each `atd_` row.

`/vouchers/party-facts` stays as it is; the client calls it per party line.

## 4. What the client already does (coded 2026-09-26, no change needed after)

The screen holds no rules, so it follows the type row:

- **MANY:** the party field reads "— several —". Parties are picked on the
  lines, from grid 119 on the party side.
- **The bill-wise card follows the selected party line.** It asks `/open-bills`
  on the side that line settles and sends allocations with that line's
  `lineRowNo`. A reopened draft puts each line's stored allocations back.
- **The TDS tick column shows on any MANY type that deducts.** The server
  decides per line.

When §1 is applied the screen switches over by itself. `tests/register/` gets
a MANY receipt case once §2's server test passes.

---

## Checked 2026-09-26 (migration `20260926150000_receipt_payment_many_parties`)

Tested with `/vouchers/validate` (dry, nothing written), then through the Qt
screen: `tests/register/run.sh many`, 18/18.

- **§1 done.** `/types` answers RcpV and PmtV as MANY.
- **§2 works.**
  - RcpV: Dr Cash; Cr ZT-CUST-CREDIT against two bills; Cr ZT-CUST-CASH against one bill. It validates clean, and each allocation carries its own line.
  - B allocated 50 short is refused, naming B only.
- **On-account: kept strict.** An unallocated customer line is refused with
  `VCH_BILLWISE_SHORT`. The client shows that as it is. A part paid on account goes on
  a separate voucher (a Journal or a ONE-party type). Taken as your answer.
- **§3 works, as `derived.tdsLines[]`.** There is one `CR TDS Payable` leg per supplier;
  `derived.tds` is null on MANY.
  - The typed party line is the NET paid, and the party leg is the gross. For example, typed 9,800 with a
    10,000 bill: TDS 200, and the bank is credited the net.
  - The client now reads `tdsLines`, lists them in the Balance card, and marks a keyed net line "net
    of TDS" with the gross in its tooltip.
