---
name: erp-till-counter-claim
description: "plan-till-counter-claim built in ERP server 2026-10-08 (open-check, claim without binding, money follows the session's device); open question on back-office/unlinked cash; relink deferred to phase 3"
metadata:
  node_type: memory
  type: project
  originSessionId: b4432845-bb98-494c-be58-deb4669eb306
  modified: 2026-10-08T07:11:45.576Z
---

`Dev/till/plan-till-counter-claim.md` was built in `src/modules/till` on 2026-10-08. It is uncommitted; the whole till module is untracked.

**What changed:**
- **New route.** `GET /till/sessions/open-check`. Each free counter carries its own `carriedFrom`. It also returns `busyCounters` and `floatMode`-free advice.
- **`claimCounter` no longer binds `tcn_device_id`.** It used to bind the counter to the first device that opened on it.
- **New codes:** `TILL_DEVICE_UNKNOWN` (409), `TILL_DEVICE_BLOCKED` (403, my addition), `TILL_NO_FREE_COUNTER`, `TILL_COUNTER_INACTIVE` and `TILL_COUNTER_NOT_YOURS`.
- **One holding session per device.** A second one is refused with `TILL_COUNTER_BUSY`.
- **`resolveForMoney`** now also governs a device that holds a live session, so a picked counter's bills land in its session.
- **Counter master** refuses `web:` and `WEB-` uids.
- **Tests:** `test/till-counter-claim-http.e2e-spec.ts` has 16 tests. One app plays many identities: the stub token parses `<user>@<device>`.

**Not built:**
- §3.3 and §3.4, re-link and device move. Both need the phase-3 `COUNTER_RELINK` approval gate.
- **Rule 2 money half.** The plan says a back-office PC takes no cash and calls that "existing"; it is not. Likewise, an unlinked device with no session still posts cash as before (the rollout rule). Enforcing either needs a branch-level gate ("the branch has a session counter") and a money-tender flag from the bill, receipt, payment and return callers. The user was asked to decide.

**Why:** the till module was built by another session that same morning; this plan changed its open path.

**How to apply:** before touching till open or the money paths, re-read `till/README.md` §"The counter claim". See [[erp-notes-95-staff-advance-till-pin]].
