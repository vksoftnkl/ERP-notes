---
name: erp-quotation-entry-legacy-ui-redesign
description: Quotation Entry + its F8 "Show list" popup restyled to match legacy Qt screenshots (2026-07-31) — merged header panel, toolbar/Copy-as-New, totals restructure, list-popup toolbar/filters/pager, and what was deliberately left out
metadata:
  node_type: memory
  type: project
  originSessionId: 3aad7995-7879-40a7-9eb6-a21cfaeedbec
  modified: 2026-07-31T11:39:03.123Z
---

On 2026-07-31, restyled `features/sales/quotation/` to visually match a legacy Qt
screenshot the user supplied, on top of the React port described in
[[erp-quotation-entry-port]]. Confirmed via two `AskUserQuestion` decisions before
touching code: (1) implement "Copy as New"/"Cancel" as real features, not stubs;
(2) merge the three header GroupBoxes into one bordered panel with three plain
columns (no per-section border/legend).

**What changed:**

- `header-blocks.tsx` — `CustomerBlock`/`SalesInfoBlock`/`QuoteInfoBlock` no longer
  wrap themselves in `GroupBox`; the caller (`quotation-entry-view.tsx`) puts all
  three inside one `.headerRow` panel (now bordered/backgrounded in
  `page.module.scss`, not just a bare grid). Column order is Customer → Sales info
  → Quotation, matching the reference. Fields were reordered/relabeled to the
  legacy names (Existing Customer, Customer Name, POS State Code, Quote No, Ref
  No, Quote Date, Validity Days, etc.) and `Area` moved from the Sales column to
  the Customer column. The freight/loading-basis selects and the
  discount-alters-base-rate checkbox moved from the Quotation block into the
  Sales-info block (grouping "charge policy" controls together) — `TermsBlock`
  keeps its own `GroupBox`, only its field order/labels changed (Remarks first,
  "T & C" renamed "Other Terms").
- **"Copy as New" (F9) is real**, not cosmetic: `copyDraftAsNew(draft, quoteDate)`
  in `quotation.state.ts` strips the server-owned identity (`docId`/`quoteSlno`/
  `quoteRefno`/`revisionNo`, each line's `sqiId`, each charge's `cdId`), resets
  `status`/`mode`/`pricing`/`isDirty`/`storedPricing`, and re-derives `validUntil`
  from a fresh quote date via the existing `applyHeaderField` — everything else
  (customer, terms, policy, lines, charges) carries over untouched, including
  unsaved edits. Wired through `use-quotation-draft.ts`'s new `copyAsNew()`,
  disabled when there is no `docId` yet. "Cancel" in the toolbar just reuses the
  existing `guardedRun("back")` (previously only reachable via the "‹ Quotations"
  header link) — no new dialog needed.
- **Totals panel restructured, not just restyled.** The reference image's three
  columns (Items/Gross/Scheme disc/Round off, "Bags"/Item disc/Taxable,
  Weight/Special disc/Tax) map onto REAL, separate `DocumentTotals` fields that
  the old `TotalsStrip` was collapsing into one combined "Discount" cell
  (`itemDisc`, `splDisc`, `schDisc` are already distinct — no data was invented).
  "Total Bags" reuses `totQty` — `quotation.payload.ts`'s `sqTotBags: totals.totQty`
  already treats them as the same figure (comment: "Bags is the piece count the
  document was quoted in"). The fuller CGST/SGST/IGST/Cess/Freight/Loading/
  Cash-disc breakdown the legacy screen doesn't show is real, saved data and was
  NOT deleted — it lives behind a `<details>` "More totals" disclosure inside the
  same panel. `TotalsStrip` now returns only the boxed panel; a new sibling export
  `TotalsFooterStats` renders "Total Profit (margin %)" / "Total Saving (%)" (the
  old "Margin"/"Saving" cells, relabeled) as their own row above the button bar,
  matching the reference's placement outside the totals box.
- Toolbar reordered to the reference's exact left-to-right order: Save, Save &
  print, Show list, Copy as new, Edit, Delete, Clear, Cancel (previously Save was
  rightmost as the primary action; now it's leftmost per the image).
- Status shown as a big coloured word top-right (`STATUS_BADGE_CLASS` map in
  `quotation-entry-view.tsx`; colours in `page.module.scss` — red for
  DRAFT/REJECTED/CANCELLED, green for ACCEPTED/CONVERTED, amber for SENT, grey for
  EXPIRED), in addition to — not instead of — the existing functional Status
  `<select>` in the Quotation column and the "Read only" badge.
- Page background is a local pale pink (`#fdf1ee`, hard-coded in
  `page.module.scss`, not a shared `--page-bg` token — no other screen looks like
  this) plus a 3px `--primary`-coloured rule under the title bar.

**Deliberately NOT done, and why:**

- **No "Beat" field.** The reference image has one; there is no `beat`/route
  concept anywhere in `quotation.types.ts`, `quotation.constants.ts`, or any
  dropdown id — adding it would mean inventing a fake control with nowhere to
  read/write, which is worse than omitting it.
- Dropped the old combined "GST type" read-only display (never fed any
  calculation, purely informational, not in the reference image) — the
  `customer.gstType` data itself is untouched, just not painted anymore.

New tests: `quotation.state.test.ts` → `describe("copyDraftAsNew", ...)` (3 cases:
identity stripped, line/charge ids cleared while `key`/`chgId` survive, date
re-derivation). Full `npx tsc --noEmit` and `npx vitest run` both clean after the
change (124 tests).

Verified live via [[erp-client-ui-browser-verification]]: new blank entry form
screenshot matches the reference closely (3-column panel, totals grouping,
toolbar order, DRAFT badge + red rule). Opening an EXISTING saved row (any row,
not just soft-deleted ones) through the synthetic JWT session hit "Unable to load
selected quotation details: Validation failed" — not chased further (not
introduced by this change; the load/parse path in `quotation.payload.ts` was not
touched), and may just be an artifact of the synthetic session/user having no real
business context rather than a genuine server bug. Worth a real login-based check
before trusting either way.

**Follow-up same day: the F8 "Show list" popup redesigned to match a second
reference screenshot** ("Select quotation to alter"). `quotation-list-modal.tsx`
rewritten:

- Toolbar row (Select/Refresh/Print/Format, `react-icons/fi`) — Select re-fires
  the current active row's pick, Refresh calls RTK Query's `refetch()`, Print and
  Format both `toast.info` (no server print pipeline; Format has no spec at all —
  same "not available yet" treatment, not a fake no-op).
- Search/From/To/Period filter row — Period presets (`today`/`7d`/`30d`/`month`/
  `custom`/`all`, default `all`) compute From/To via the existing `addDays`/
  `todayIso`; editing a date field by hand flips Period to `custom`. All client-
  side, same reasoning as the existing text search (grid 84 has no filterable
  column).
- **Deleted rows are now SHOWN, not hidden** (dimmed/italic row + red "DELETED"
  tag next to Status) — the old version filtered `sq_is_deleted` out entirely.
  Picking one still opens it read-only for free: `onPick`→`loadDocument` already
  defaults every loaded document to browse mode regardless of this change.
  Editing a loaded-deleted document is NOT separately blocked (no `isDeleted` flag
  exists on `QuotationDraft` at all) — out of scope for a popup restyle; flagged
  here as a real gap if the next request is "stop that".
- **New `buildPageList` pure helper** in `quotation.utils.ts` (+4 tests) — copied
  faithfully from `components/ui/table.tsx`'s pagination-bar algorithm (first/
  last/±1-neighbour/ellipsis) so this dialog's pager matches every other paged
  table in the app. Pagination itself is now two-tier: one server fetch (`page:
  1`), then a real local pager (`LOCAL_PAGE_SIZE = 10`) over the filtered result
  — replacing the old "Previous/Next re-fetches the next SERVER page" control.
- **Trap hit and fixed:** `configured-grid-sql/run`'s `limit` has an undocumented
  server-side cap — `limit=200` 400s with `"limit must not be greater than 100"`
  (no `@Max` visible by grepping the DTO file directly; it's inherited from a
  base pagination DTO). `FETCH_LIMIT` is 100, not 200. Check this cap before
  bumping ANY configured-grid-sql fetch size on another screen.
- `ModalShell` gained a `wide` prop (`min(78rem,100%)`, vs. the default 64rem) —
  needed for the toolbar + filter row + 9-column table; existing modals
  (item/charge pickers) are unaffected, `wide` is opt-in.

Related: [[erp-quotation-entry-port]], [[erp-ui-table-width-is-not-a-percent]].
