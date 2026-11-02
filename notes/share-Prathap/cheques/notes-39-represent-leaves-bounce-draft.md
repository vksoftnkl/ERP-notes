# Notes (39) — re-present leaves the BOUNCE voucher in DRAFT

**Viji, 2026-09-19.** Found while checking something else.

**FIXED AND VERIFIED the same day.** Retested:

| retested | result |
|---|---|
| bounce voucher after BOUNCE | **POSTED**, totals stamped Dr 500 / Cr 500 |
| bounce voucher after RE-PRESENT | **still POSTED**, totals still stamped |
| POSTED-only ledger once the cheque clears | Cheques in Hand nets to **0.00** |
| any new DRAFT bounce since the fix | **none** — newest DRAFT 14:33, newest POSTED 15:07 |

**CORRECTION to §1 below — I understated it.** Those vouchers are not merely
DRAFT: the old re-present **SOFT-DELETED the whole bounce voucher, header and
every ledger leg**, and set the header back to DRAFT. It left the bounce's
BILL adjustment row live. So the ledger has no trace of the bounce at all,
while the bill still reflects it. Same consequence, worse mechanism.

The four pre-fix vouchers were **not backfilled** (`chqbnc00017`, `00061`,
`00062`, `00075`):

| voucher | header deleted | legs | live legs | live bill rows |
|---|---|---|---|---|
| chqbnc00017 | yes | 5 | **0** | 2 (−2,000) |
| chqbnc00061 | yes | 5 | **0** | 1 (−500) |
| chqbnc00062 | yes | 2 | **0** | 1 (−500) |
| chqbnc00075 | yes | 0 | 0 | 0 |

**`fix-draft-bounce-vouchers.sql` in this folder repairs the first three** —
un-delete header and legs, stamp the totals, set POSTED — and deliberately
leaves `chqbnc00075` alone: it has no legs, no bill rows and no cheque, so
posting it would create the empty voucher `ck_avh_balanced` cannot catch. The
script verifies before and after and rolls back unless exactly three are
repaired. I could not run it myself; the `claude` login is read-only.

Smaller than notes (38); same area, independent of it.

---

## 1. What happens

After `POST /cheques/re-present`, the **bounce voucher is left in DRAFT**.
Three re-presented cheques on the box, three DRAFT bounce vouchers, no
exceptions:

| bounce | status | cheque | presents | cheque now owned by |
|---|---|---|---|---|
| chqbnc00017 | **DRAFT** | NQ123 | 2 | rct00753 (re-issue) |
| chqbnc00061 | **DRAFT** | NMK001 | 2 | rct00820 (re-issue) |
| chqbnc00062 | **DRAFT** | 107960 | 2 | rct00822 (re-issue) |

Every other bounce on the box is POSTED or CANCELLED. Only the re-presented
ones are DRAFT.

Two further symptoms of the same missed step:

- `avh_total_debit` and `avh_total_credit` are **0.00** on all three — the
  header totals were never stamped, though the legs are there and balance.
  (This is the `ck_avh_balanced` vacuity: the totals are stamped rather than
  derived, so a CHECK cannot notice.)
- The bounce's **bill adjustment row stays LIVE and counting**, which is
  correct — so the BILL is right while the VOUCHER says draft.

## 2. Why it matters

The bounce is the entry that re-debits the party and relieves Cheques in Hand.
Leave it out of the ledger and the cheque is credited to the party twice.

Cheque **NQ123**, 2,000, party SATHEESH — every voucher that touched it:

| voucher | type | status | party | Cheques in Hand |
|---|---|---|---|---|
| rct00702 | Rct | POSTED | −6,230 | **+2,000** |
| chqbnc00017 | ChqBnc | **DRAFT** | **+2,300** | **−2,000** |
| rct00753 | Rct re-issue | POSTED | −2,000 | **+2,000** |
| chqclr00013 | ChqClr | POSTED | — | −2,000 |

Counting POSTED only:

- **Cheques in Hand = +2,000**, for a cheque that has cleared and gone
- **the party is credited twice** for one 2,000 cheque

The legs themselves are right. `chqbnc00017` debits the party 2,300 (2,000 +
300 charges) and credits Cheques in Hand 2,000, and it balances. It is only
the STATUS that is wrong.

## 3. What I think it should be

Notes (37) already says it: *"The bounce columns are LEFT ALONE — that it
bounced on the 14th stays true."* The same should hold for the bounce
**voucher**. A re-presentation is a fresh act forward, not an undo — the
re-issue voucher carries it. So:

- the bounce voucher stays **POSTED**, with its totals stamped;
- re-present adds the re-issue voucher and changes nothing behind it.

If instead a re-presentation is meant to neutralise the bounce, then the
bounce should be **CANCELLED** with its bill row reversed — but not DRAFT,
which is neither, and which silently drops it out of every ledger read while
its bill row keeps counting.

## 4. Reproduction

```
post a receipt with a cheque -> deposit -> bounce -> re-present
SELECT avh_voucher_status, avh_total_debit, avh_total_credit
  FROM accounts.acc_voucher_header
 WHERE avh_voucher_id = <the bounce voucher>;
-- DRAFT, 0.00, 0.00 — while acc_vouchers holds its balanced legs
```

## 5. NOT a defect — noted so nobody chases it

- **POSTED bounce vouchers whose party leg is CR and Cheques in Hand DR** are
  the REVERSAL vouchers a receipt cancellation writes, 0.3s after the original
  goes CANCELLED. Correct.
- **chqbnc00037-40**: POSTED, 18,500, zero legs, no cheque linked, all created
  inside one second at 13:46:20. A test-harness artifact, not a product path.
