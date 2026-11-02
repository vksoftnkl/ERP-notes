# receipt — the Receipt voucher (menu 99), revision 3

The first screen that WRITES `accounts.acc_vouchers` legs, and the direct
consumer of `../openingbal/`: an opening bill exists only to be receipted
against, and nothing in that design is exercisable until this exists.

Three revisions in one day, each read against a reference: rev 1 off the
live schema; rev 2 off the 3.0 screen (`NextLvl/3.0/pos/Accounts/
customerreceipt.cpp`) and TallyPrime's Receipt (F6); rev 3 off GoFrugal's
receivables features. Rev 3 is ONE phase by decision.

**Building the endpoints? Start with `HANDOVER_endpoints.md`** — the final
six routes after the 2026-09-15 corrections (one `partyId`, no `/list`, the
four keys, no approve/reject), so notes (7), (11) and (12) do not have to be
reconciled by hand.

**Read in this order:**

| file | what it is |
|---|---|
| `receipt_ui_mockup.png` | **start here** — the screen, rev 3: header with Beat + customer, last-receipts / pending-cheques panels, the bills (due-date first, PDC HELD column, PPD-seeded discount), the credits the party holds, the instruments (a post-dated cheque is its own voucher), the OTHER LEDGERS band (TDS, bank charges, discount, TCS, claims — every line a role), the identity strip, Approve / Post & Print (`.py` beside it regenerates it) |
| `receipt_flow.md` | the spec — every table, column, CHECK and index a receipt touches, read off the live DB 2026-09-14; §1 the five live-vs-spec divergences; §4 the worked example (70,000 as cash + UPI + PDC cheque, TDS 700, credit note 4,000, two discounts, 4,650 on account) written row by row through six tables; §5 the post-dated model; §6 cancel / approval / header edit; §10 decisions R1–R18, all taken |
| `plan-backend-receipt.md` | what Prathap builds — §2 schema prerequisites with the DDL (balance trigger + daily PDC regularise, header-totals trigger, `Rct` type, cheque-table repoint, `av_role`, roles, settings, `um_can_approve`, ~~`cus_ledger_id`~~ **withdrawn — `cus_id` IS `led_id`, §2.11**, the area dropdown), nine endpoints, the posting transaction step by step, 40-odd tests, §10 what is deliberately out |
| `plan-backend-delete-draft.md` | **owed** (2026-09-17) — `POST /receipts/delete`, the one route with no workaround: `/cancel` refuses a DRAFT and nothing replaces it, so an abandoned draft is permanent. Why it is safe (a draft writes no bill, no leg, and — verified — no `acc_pdc_register` row even carrying cheque tenders), the transaction, and decision R19 |
| `plan-backend-amend-posted.md` | **owed** (2026-09-17, client request) — `POST /receipts/amend`, editing a POSTED receipt WHOLE rather than field by field: the create+post payload against the same header, same number, `avh_revision_no + 1`. Why the full payload beat a tier of small routes, the transaction as §5.3-without-the-reversal then §5.2, the three additions (one column, three tables into `audit.audit_log`, one COMPANY setting default OFF), the refusal list it shares with cancel, and decision R20 |
| `gap-vs-3.0.md` | what 3.0's `CustomerReceipt` has that menu 99 does not — 14 items graded A (blocking) to D (deliberately dropped), each naming the 3.0 widget or slot it came from. Read with the client plan; the A items are prev/next voucher, Save&New / Save&Print / Clear as separate verbs, and auto-adjust as a MODE |
| `plan-qt-receipt.md` | **the Qt client plan** (2026-09-14 evening) — own `.ui` on `NexDialog`, file set, the bands as widgets, the client-side allocation engine + identity, endpoint calls, tests. Coded after the backend is up |

**What the schema was missing** (flow §1, plan §2): `acc_bill_balance` and
`acc_pdc_register` are PARTITIONED live though both specs say not; the
bill-balance trigger from `09` is not deployed; two cheque tables with
`abj_cheque_id` on the older one (R1 → `acc_pdc_register` only);
`acc_ledger_map` empty; no Receipt voucher type; no `av_role`; no
right; no customer ↔ ledger bridge.

Nothing in these files is assumed. Every decision is taken and says who took it.
