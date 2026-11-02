# Notes 112: a session closes after a RECOUNT decision without being recounted (2026-10-14)

**STATUS:** §1 DONE + verified live 2026-10-14 (both (a) and (b)). §2 repair still OPEN: two sessions. §3 not built.

**Live check 2026-10-14, 11/11 PASS** (session ZT1-261014-01, Z 20): two counts of 2500 against 2000 gave SENT_FOR_APPROVAL. Close then answered 428. The approver chose RECOUNT, which left `tss_close_count_id` and `tss_counted_on` NULL and `tss_cash_counted` 0. Close then answered **409 TILL_NOT_COUNTED**. A new count of 2000 was ACCEPTED, and close went through. The old PENDING row was soft-deleted; the new one is WITHIN_TOLERANCE POSTED. The temporary approver grant was removed.

**§2 still open: the query below finds TWO sessions** (both closed before the fix):
- `VIJI PC-261012-02`: cash 5,600 / 7,500 (+1,900)
- `ZT1-261012-01`: cash 2,177 / 2,000 (−177), a test session

## What happened (live, session `VIJI PC-261012-02`, company 019c8ea6…, Coimbatore)

From `accounts.till_event` for the session:

| Time | User | Event | Detail |
|---|---|---|---|
| 11:14:33 | VIGNESH | SESSION_END_BILLING | BLIND count |
| 11:14:39 | VIGNESH | SESSION_COUNT | attempt 1 → RECOUNT_REQUIRED |
| 11:15:06 | VIGNESH | SESSION_RECOUNT | attempt 2 → SENT_FOR_APPROVAL |
| 11:15:06 | VIGNESH | APPROVAL_REQUESTED | CASH_VARIANCE: cash +1,900.00 (5,600 expected / 7,500 counted), UPI +50.00 (700 / 750) |
| 11:19:43 | VKPOS | SLIP_CHECK | |
| 11:20:19 | VKPOS | VARIANCE_DECIDED | decision **RECOUNT**, amount 1,950.00 |
| 11:21:09 | VKPOS | MOVEMENT_POSTED | CLOSE_HANDOVER 7,500.00 to the safe |
| 11:21:09 | VKPOS | SESSION_CLOSE | Z 2, float left 0, handed over 7,500.00 |

No count happened between the RECOUNT decision and the close.

State now:

- `till_session`: `tss_status = CLOSED`, `tss_variance_status = PENDING`.
- `till_variance` CLOSE rows: cash 5,600 / 7,500 and UPI 700 / 750, both `tvr_treatment = PENDING`, `tvr_status = OPEN`. **Neither is posted.**
- The handover moved the counted 7,500 to the safe. The 1,900 cash excess never went to Cash Short & Excess, so I expect the till cash ledger to be off by 1,900 for this session. Please confirm on your side.

## Why (read from the source, `till-session.service.ts`)

1. **`decideVariance` RECOUNT** sets `tssStatus = COUNTING` and `tssRecountsGranted + 1`. It leaves `tssCloseCountId` (and `tssCountedOn`) as they were, from the count that went out of tolerance.
2. **`close`** accepts COUNTING and checks only `if (!session.tssCloseCountId) → TILL_NOT_COUNTED`. The old count id is still there, so the close goes through.
3. **The variance loop in `close`** skips `PENDING` treatments: "a slip gap left PENDING by an accepted count stays OPEN". So the cash PENDING row is skipped too, and the session closes with an undecided cash gap.
4. **After that, nothing can decide it.** `decideVariance` calls `assertPendingApproval`, so a CLOSED session is refused.

## Asked

### 1 · Close must not follow a RECOUNT without a new count

Either of these would do; your choice:

- **(a)** RECOUNT clears `tssCloseCountId` / `tssCountedOn` (and `tss_cash_counted`), so close answers `TILL_NOT_COUNTED` until the next count.
- **(b)** close refuses (a new code, or `TILL_NOT_COUNTED`) while any **cash** CLOSE variance is still `PENDING`. A slip-only PENDING gap stays allowed (notes 99 §1).

(b) also catches any other path that leaves a cash gap PENDING. Doing both is fine.

### 2 · Repair `VIJI PC-261012-02`

Book the cash +1,900.00 the way an APPROVE + EXPENSE would have done (Cash Short & Excess, `tvr_status` POSTED, `tss_variance_status` set to match). The UPI +50.00 is a slip gap, so it can stay for the slip check / NONCASH decision as usual. Or tell me what you would rather do, e.g. a manual journal.

Are there other sessions in this state? This query should find them:

```sql
select s.tss_session_no, v.tvr_tender_type_id, v.tvr_expected, v.tvr_counted
  from accounts.till_session s
  join accounts.till_variance v on v.tvr_session_id = s.tss_id and v.tvr_acc_year = s.tss_acc_year
 where s.tss_status = 'CLOSED' and v.tvr_stage = 'CLOSE'
   and v.tvr_treatment = 'PENDING' and v.tvr_tender_type_id = 1 and not v.tvr_is_deleted;
```

### 3 · (Optional) Deciding a closed session's leftover

If the repair in §2 is likely to be needed again, a route for an approver to decide (APPROVE + treatment + reason) a cash variance that is still PENDING on a CLOSED session would replace hand repairs. I'm not asking for it if §1 makes the state impossible.

## Qt side (already built, uncommitted)

- **Cockpit (273):** for a CLOSED session with a PENDING cash variance, the approval band no longer says "waits for a supervisor". It says the session closed without a decision, the gap is NOT posted, and a closed session cannot be approved.
- **End Shift:** it still takes "`tssCountedOn` is set" to mean the count was accepted, so after a RECOUNT it offers Close. I'll match whichever of §1 (a)/(b) you pick. With (a) it fixes itself.
- **Open Till Session (272):** while the cashier's own session is PENDING_APPROVAL, it now explains the wait and re-checks every 15 s. Resume on a session from an earlier business date warns (`TILL_SESSION_DAY_ENDED`) instead of letting the cashier bill into a refusal.
