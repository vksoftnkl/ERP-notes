# GSP layer — Qt client plan

**2026-09-20, all defaults.** Two masters, two registers, badges on three
documents. Mockups (PNG first) owed before the `.ui` files.

## 0. Ids to reserve (confirmed by the user 2026-09-20 — continues `../sales/plan-qt-sales.md` §0)

| kind | id | name |
|---|---|---|
| menu | 255 | GST Providers (admin) |
| menu | 256 | GST Credentials (company) |
| grid | 124 | MAIN LIST - GST PROVIDERS · 125 CHILD - PROVIDER SERVICES · 126 CHILD - SERVICE ENDPOINTS · 127 CHILD - ENDPOINT FIELD MAP · 128 CHILD - PROVIDER ERROR MAP |
| grid | 129 | MAIN LIST - GST CREDENTIALS · 130 POPUP - GST JOB LOG (per document) |
| dropdown | 58 | providers · 59 provider services · 60 NIC cancel reasons (IRN 1–4 / EWB 1–4, static rows) |
| ui_table | 43 | provider services · 44 endpoints · 45 field map · 46 error map · 47 credentials |

Grids 121 / 122 (registers 152 / 153) are reserved in the sales plan.

## 1. Masters — `src/modules/admin/master/gst_provider/`, `gst_credential/`

**GST Providers (255)** — `NexCrudMasterPage` + `NexCrudForm` for
`gst_provider`, with four child grids (services → endpoints → field map;
error map) as tabs on the form, each a child `NexTable` (memory:
nextable-grid-traps — register every column, set the editor). Read-
mostly: the seeded rows are what ships; "Add provider" is for the day a
second GSP is contracted. Secrets on the provider ACCOUNT tab are
write-only fields (a password box that shows "set, version 2").

**GST Credentials (256)** — company × service × environment rows:
GSTIN (read from the company, shown), NIC username, password
(write-only), client id / secret (write-only), environment, priority,
valid from / to, whitelisted IPs; **Verify** button → `/gst/company-
credentials/verify` shows the result inline; the status strip from
`/status` (session valid until, last error, balance). Branch-scoped
credentials are a second row with the branch set.

## 2. Registers — `src/modules/sales/gst/` (menus 152 / 153)

Both on `TxnMainView` (descriptors `einvoiceRegister()` / `ewaybill
Register()` in `txn_doc_types_sales.cpp`): rows = register rows joined to
`gde_` / `gdw_`, status badges text-derived (NA / PENDING / GENERATED /
FAILED / CANCELLED / EXPIRED / REJECTED), filter row = period + status +
document type, search = party / doc no / IRN / EWB no. Row actions (a
panel dialog like the cheques screen, memory: cheque-panel-dialog):

* 152: **Generate now** (`?sync=true`), **Fetch**, **Cancel** (reason dd
  60 + remark; greyed past the window with the hours left shown),
  **E-way from IRN** (opens the transport block editor when missing),
  **Print QR / signed invoice**, **Log** (grid 130).
* 153: **Generate**, **Update vehicle** (Part-B), **Extend**, **Cancel**,
  **Manual entry** (`/gst/ewaybill/manual`), **Print**, **Log**.
* Both: a top strip with counts Generated / Pending / Failed / Cancelled
  for the period (3.0's three counts + FAILED) and the environment label
  (SANDBOX in amber, always visible — G8); **Show JSON** (the preview
  route — 3.0's `einvoicedetail`); a progress bar for a bulk Generate over
  the filtered rows. F-keys as 3.0 had them, so the operator's hands know
  them: Change period F5 · Print bill F6 · Generate IRN F7 · Get IRN F8 ·
  Cancel IRN F3 · Generate EWB F9 · Cancel EWB F10 · Get EWB F11 · Print
  EWB F12.

`user_menus` rights: generate = `um_can_post` on the register menu,
cancel = `um_can_cancel`, manual entry = `um_can_post` + `um_can_override`.

## 3. On the documents — generation happens HERE first

Bill, return, DC, DC return: IRN badge + EWB badge from `/get`'s `posting`,
and the document's own **e-Inv** / **e-Way** quick buttons generate for
THAT document (`?sync=true`): generate / retry / fetch for the IRN,
generate / Part-B vehicle / print for the EWB, with the transport band
opening itself when fields are missing (sales plan §1.6, §1.8). Cancel /
Amend on the document grey with "IRN live since HH:MM — cancel it in the
e-Invoice register first" past the window; inside it the document's cancel
does the IRN cancel itself. Shift+F2 / Shift+F3 open the registers on the
row. The registers (152 / 153) are for cancel, extend, bulk generate,
retry sweeps and the log — not the first place a bill's IRN is made.

## 4. GSTIN fetch buttons

Customer, supplier, ledger, ship-to forms: the existing "fetch" gesture
calls `/gst/gstin/lookup` and fills name / trade name / state / reg type;
a failed lookup shows the mapped message and leaves the fields.

## 5. Tests (`tests/gst/`, offscreen)

Descriptor badge derivation for the seven statuses; the panel dialog
actions call the right route with the right keys (mock HTTP); window
greying from `posting.irn.generatedOn` vs `EINV_CANCEL_HOURS`; manual
EWB dialog validation (12-digit number, valid-upto ≥ generated-on);
credentials form never echoes a secret.

## 6. Order (P5 of the sales plan, interleaved)

1. GSTIN fetch buttons (P1 of the backend — first visible win)
2. badges on the bill (with sales P5 step 2)
3. register 152 + panel
4. register 153 + panel + transport dialog
5. masters 255 / 256
