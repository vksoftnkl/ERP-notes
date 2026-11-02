---
name: receipt-tender-surcharge
description: Receipt UPI/card surcharge is paid ON TOP (user 2026-10-12); client mirrors the server formula; notes-107 asks for a round 202
metadata:
  type: project
---
UPI and CARD have tnd_surcharge_perc 1% in the tender master. The server seeds SURCHARGE_RECOVERED (CR, an "addition") = round2(tdAmount × % / 100) on the whole tender, and refuses an unbalanced receipt (400 "Out by -2.00"). The Receipt screen had no surcharge model, so a UPI line failed to post.

**User decided 2026-10-12: the customer pays on top.** In receipt_entry.cpp, typing an amount on a surcharge tender keeps it as TenderRow.forBills and raises amount to grossFor(): 200 → 202.02 (surcharge 2.02). refreshSeededLines seeds SURCHARGE_RECOVERED from surchargeOn() per tender. TenderRow.sendSurcharge sends tdSurchargeAmt only when tnd_edit_surcharge is on.

Dry harness: tests/receipt/run-surcharge.sh (Deepan, SEED-DEEPAN-1, CASH 100 + UPI 200 + CHEQUE 700) passes, but nothing has been posted live.

**Server ask:** notes-107 (Shared/Prathap/admin) to use % / (100 + %) so 200 → 202.00. When it is live, change grossFor() to forBills × (1 + % / 100).

**Payment screen:** not changed. It shares TenderRow but has no surcharge hook.

**2026-10-16 FIXED:** server (notes 127 §4) and Qt receipt_entry.cpp surchargeOn/grossFor now use the INSIDE formula round2(amt×p/(100+p)): 200 @1% → 202.00 / 2.00. tests/receipt/run-surcharge.sh 7/7 incl. a server draft save+delete (needs ninja on PATH).
