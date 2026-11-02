# Notes 116: the supervisor who closed a session may approve its cash (2026-10-14)

**STATUS:** DONE + verified live 2026-10-14 (scratchpad n116.py, ZT1-261014-11): VKPOS the cashier counted, zt-approver closed it (CLOSE_HANDOVER tdr00273 done_by zt-approver) and then approved its cash → 200 APPROVED; VKPOS refused with "… is yours (you were its cashier): another supervisor approves its cash".

## What happened (live, `VIJI PC-261014-05`, Acme Foods / Coimbatore)

| Time | Who | Event |
|---|---|---|
| 18:20:16 | VIGNESH (cashier) | counted ₹400 against ₹300 expected, so SENT_FOR_APPROVAL |
| 18:34:59 | VKPOS (super admin) | VARIANCE_DECIDED: approved the +₹100 |
| 18:36:32 | VKPOS | SESSION_CLOSE from the Session Cockpit's Count / close. CLOSE_HANDOVER `tdr00236` ₹400, `tcm_done_by` = VKPOS, AWAITING |
| 18:38 | VKPOS | `approve-cash` → 403 `TILL_RECEIVER_SELF`, "Session VIJI PC-261014-05 is your own: a supervisor other than its cashier approves its cash" |

VKPOS is not the cashier. The refusal comes from the second half of the check in `till-movement.service.ts` `approveCash` (around line 1292):

```ts
const own =
  caller.userId === peek.tssOperatorId || waiting.some((m) => m.tcmDoneBy === caller.userId);
```

When a supervisor closes the session for the cashier, the hand-over is the supervisor's. That supervisor can then never approve the cash. A third person has to, and a small shop often has no third person. The message also tells the supervisor the session is "your own", which is wrong.

## The user's decision

> The supervisor who closed the session may approve its cash. The rule is simply: **never the session's cashier**.

That supervisor is holding the cash already. Asking a third person adds a step, not a control. The approval still carries their name and their PIN, and the received amount is typed, so any gap goes through the usual tolerance, grant and variance path.

## Asked

1. **`sessions/approve-cash`:** refuse only the session's cashier (`tssOperatorId`), with `till.single_operator` still letting the cashier approve their own. Drop the `tcmDoneBy` half of `own`.
2. **The message for the cashier:** "Session X is yours (you were its cashier): another supervisor approves its cash." It should not say "your own" to anyone else.
3. **A DROP done by someone other than the cashier:** same rule; only the cashier is refused.
4. **`movements/receive`** (the retired Cash Office route, around line 995) has the same `tcmDoneBy` check. Change it the same way, or leave it, as you prefer: the Qt client no longer calls it.
5. **A RECEIVE difference sent on for approval** keeps today's rule: the approving supervisor is the requester, so `decide-variance` refuses them and the cashier. Deciding it in the same step with their own grant (notes 114 `treatment`) stays as it is.
6. **The e2e test** "approve-cash refused … to the cashier": add a case where a supervisor closes the session and then approves its cash (200).

## Qt side

No change needed: the Cockpit's Approve Cash is already enabled for everyone except the cashier, and it shows the server's message as it comes. `VIJI PC-261014-05` stays AWAITING until this is live, or until another supervisor approves it.
