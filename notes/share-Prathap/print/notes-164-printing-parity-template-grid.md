Date: 2026-10-30
From: Viji / Claude
To: Prathap

# Notes 164: printing parity in Qt, and the Print Templates grid (2026-10-30)

Everything server-side is in `/home/vk/Dev/erp/ERP server`, branch DEV-v1, on .106. API :3011 already serves it. Nothing is pushed to prod yet.

## 1. Migrations APPLIED today (in `_prisma_migrations`, applied by the user)

1. `20261030150000_print_template_list_grid` registers grid `MAIN LIST - PRINT TEMPLATES`.
   - It takes id 196 where that id is free; on dev it is 196. Elsewhere the sequence picks the id, and both clients find the grid BY NAME.
   - It is a SELECT over `public.print_template`, joined to its purpose, its published revision, its newest live revision and its fork source. "Used by" is counted in the SQL over ACTIVE `print_template_assignment` rows of the company plus the every-company ones.
   - It has no sync calls, so it is safe on the VPS.
   - Committed (41bf7124).
2. `20261030160000_print_template_list_grid_widths` brings the visible widths to 100. Committed (41bf7124).
3. `20261030170000_print_template_list_grid_badges` NOT committed yet.
   - It rewrites the grid SQL so the pills have fixed words: `owner` = SHIPPED | OWN, a new `state` = PUBLISHED | NOT PUBLISHED, `published` = 'rev N' | '—', and `ptl_is_active` as the boolean itself.
   - It adds the State column, moves positions, and drops the amber condition on Published.
   - A width moves only while it still has its shipped value.

**Tokens** (send all six; '' means no filter): `iptl_company_id` (required), `iptl_active`, `iptl_purpose_id`, `iptl_state` (PUBLISHED | UNPUBLISHED), `iptl_owned`, `iptl_engine`.

**Checked on dev:** 28 active designs, 27 published + 1 not published, 3 owned, engine filters 1 + 27. These match `/print-templates/list`.

## 2. Routes Qt now calls (no server change needed; listed so you know the traffic)

1. **POST `/print-render/log`** from the Qt preview dialog. It sends PREVIEW when the paper is shown (unless auto-print), PRINT after the printer accepted the job, and FILE after Save as.
   - Body: versionId, outputMode, docId / docIds, companyId, accYear, branchId, pageCount (from `X-Print-Pages`), byteCount.
   - Qt never sends REPRINT. The live `print_log` CHECK refuses it anyway: see the server memory "print_log CHECK refuses REPRINT".
2. **POST `/print-share/create`** after a PDF renders, for WhatsApp from the Sale Bill and the Purchase Order. The link goes in the message; the PDF is still copied for pasting.
3. **`/print-render/preview` with top-level `params`** for report prints: LEDGER_STATEMENT (docId = ledger id) and PARTY_OUTSTANDING (docId = company id, the 16 filters as prompts).
4. **Voucher purposes now printed from Qt:** RECEIPT_VOUCHER, PAYMENT_VOUCHER, EXPENSE_VOUCHER (docId = avh_voucher_id). PAYMENT_VOUCHER has NO design on dev: one has to be drawn and assigned before payments print.
5. **`/grid-details/get?search=`** to find a grid by name. This is new for Qt; React already does it.

## 3. Editing a PUBLISHED design (testing phase, one client)

The user asked that a published design be editable without New draft or cloning.
- Both clients now send the edits as the NEXT revision (no `ptvId`) with `ptvStatus` PUBLISHED and the approver, in one `/print-templates/create` call. That is your rule 1: compose and publish in one call, the same as roll back.
- The published revision itself is never updated, so `print_log` stays true.
- Switch: Qt `PrintDesignerRules::kEditPublishedInPlace`, React `EDIT_PUBLISHED_IN_PLACE` (features/printing/domain/draft.ts). Both are on.
- Preview cannot show unsaved edits in this mode, because `/preview` takes a body only for a DRAFT. Both clients say so.

**Questions for you:**
1. When a new revision is published this way, is the previously published revision left as PUBLISHED (with only the pointer moving), or should the server RETIRE it? Both clients assume the pointer move is enough.
2. Is "save" through this path acceptable while the client is testing, or would you rather have a server flag that allows it per company?

## 4. Not server work, for the record

The Qt side also got: Ledger Statement (menu 258), Party Outstanding (menu 279), the designer's element and page features (borders, highlights, flow, shapes, rich text, page border, watermark, report pages, CHART, units), and theme v47 published (purchase screens). Qt commit `131b042`; React changes are uncommitted in ERP client `dev`.
