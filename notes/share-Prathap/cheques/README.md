# cheques — Received Cheques (menu 51), revision 1

What happens to a cheque after the receipt (`../receipt/`, rev 3) took it in:
deposit, clearance, bounce, re-presentation, replacement, return. One
register, one page, one table (`accounts.acc_pdc_register`, R1 of the receipt
design), and a lifecycle the table's own CHECKs already enforce.

**Read in this order:**

| file | what it is |
|---|---|
| `received_cheques_ui_mockup.png` | **start here** — the screen: filters, the summary strip with its due buckets and the Cheques-In-Hand tie, the register (nine worked cheques in every state), the action bar with the state → action map, and the Deposit / Bounce / Clear · Re-present · Replace · Return panels (`.py` regenerates it) |
| `received_cheques_tables.png` | which table, which column, in what order — the register column by column against its `ck_apd_*`, the two voucher types and one role that do not exist yet, the three lanes, decisions C1–C10 |
| `received_cheques_flow.md` | the spec — every column, every action table by table with the worked example (221870 bounces: five legs, two bills reopen, a 300 charge becomes a JOURNAL bill), the due buckets, rules, C1–C10 taken, prerequisites |
| `plan-backend-received-cheques.md` | what Prathap builds — prerequisites (two type rows, one role, two settings, the grid, the slip dataset), eight endpoints, the bounce transaction in order, tests; one phase |
| `plan-qt-received-cheques.md` | **the Qt client plan** — one page: register (`NexApiTableView`), action bar enabled by status, the six panels as a stack; shared entity + History dialog with menu 52 |

**References read:** 3.0 `receivedcheques` / `chequedeposit` / `chequebounce`;
TallyPrime (deposit slip, cheque register, post-dated summary, bounce as
reversal + bank charge); GoFrugal (receive → deposit → acknowledge clearance →
reconcile; bounce control with a debit note to the party).

**Out, by decision:** bank reconciliation (GoFrugal's step 4 — `av_recon_date`
is the hook); Issued Cheques (menu 52) — the mirror, its own plan.
