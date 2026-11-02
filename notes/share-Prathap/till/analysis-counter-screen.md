# Counter screen — inside NexERP or a separate app? (analysis, 2026-10-07)

> The user's question: the counter needs ONE fast screen that saves Bill / Return / DC / Order /
> Quote and converts between them (as EPOSGo does), while the office keeps its separate screens.
> Build it inside NexERP, or as a separate app? Every statement below was measured in the code on
> 2026-10-07; the command or file is named so it can be re-checked.

## 1 · EPOSGo, measured (`/home/vk/Projects/NextLvl/EPOSGo`)

| Fact | Evidence |
|---|---|
| 38,257 lines (`.cpp` + `.h`), 335 source files | `wc -l` over the tree |
| The one page is `mainwindow.cpp`, **4,739 lines** | 90+ member functions: billing, schemes/offers, hold, order, DC, receipts, temp credit, expense, cash in/out, lock, switch user |
| One grid saves **Bill, Order, DC**; converts **Order → Bill, Order → DC, DC → Bill** | `editOrder(int, QString ConvType)` with "Sale"/"DC"/"Edit"; `editDC(int, bool Convert)` |
| **No quotation** (button `tBtnQuote` exists, no code) and **no sale return** (only change/refund at tender) | no `quote`/`return` handling in `mainwindow.cpp` |
| Talks to the **database directly** through functions (`app_utils.fepos_sales` ×15, `fx_sale_entry` ×6, `fx_order_entry` ×6, `fx_sale_chln_entry` ×3, `fx_hold_transaction` ×2) | `grep functionName mainwindow.cpp` |
| Its own promotion/scheme engine (`applySchemes`, `applyFreeOffer`, `applyDiscountOnBill` …) — 3.0 has others in `saleentry`, `salesorderentry`, `salesdeliverychallan` | `grep applySchemes` |
| Copies already drifted from 3.0: `tillsessiondetail` **114** differing lines, `expenseentry` **41**, `tilltenderentry` **30** | `diff` against `3.0/pos/EPOSGo/` |
| Two copies of its own widgets inside itself: `mycombobox.cpp` **167**, `mytableview.cpp` **151** differing lines | `diff Widgets/ "DB Engine/Widgets/"` |
| "Office" starts the other program: `QProcess::startDetached("bash", … "/home/vk/VKSoftwares/DistributionKit/VKPOS.sh")` | `btnOfficeClicked()` |

**Reading:** EPOSGo's **layout and flow** are the asset. Its **code cannot be reused by NexERP
at all** — it is built on database functions, and NexERP's rule is no DB functions (all logic in
NestJS, the client talks only to the API).

## 2 · NexERP, measured

### 2.1 What a counter would reuse — and whether it really stands alone

| Piece | Lines | Depends only on core/shared? |
|---|---|---|
| promotion engine + rules + cache (`shared/promotions`) | 335 + … | yes |
| `sales_promotion_binder` | 405 | yes (engine, NexTable, item entity) |
| `sales_item_delegate` + item entity | 366 | yes (session, currency format, theme) |
| hold/recall (`shared/txn/txn_hold_*`) | 143 + … | yes (+ `txn_main_view` for the picker) |
| `weight_barcode` | 41 | yes |
| `customer_quick_add_dialog` | 203 | yes |
| tender dialog `sale_bill_tender` | 1,743 | yes (+ `party_adjust_panel` from shared) |
| login, API client, session, config, theme (`src/core`) | 7,461 | yes — `loginwindow.cpp` includes no module |
| widgets, grids, printing, party, GST helpers (`src/shared`) | 25,224 | yes, except `nex_api_table_view.cpp` → `grid_master_entry.h` (one module include) |

**Where the app picks its main window:** `main.cpp` constructs `MenuWindow` after login (line 79).
A counter window is chosen at that one place.

**What ties everything into one executable:** `CMakeLists.txt` has a single
`qt_add_executable(NexERP …)` listing 289 module files, and `core/navigation/form_manager.cpp` +
`form_*_menu.h` include the screens of every module (the menu registry).

### 2.2 The server already serves every counter action

| Document | Routes (live NestJS) | Conversion support |
|---|---|---|
| Bill | create · validate · post · cancel · amend · get · **open-sources** · party-context · tender-context · retender | bill imports from orders / DCs |
| Return | create · validate · post · cancel · amend · get · **bill-lines** | return lines come from a bill |
| DC | create · validate · post · cancel · amend · get · **open-for-bill** · convert-purpose | DC lines open for billing |
| Order | create · get · post · cancel · amend · cancel-lines · pending-amount | (bill / DC import from it) |
| Quotation | create · get · delete | — |
| Hold | create · list · resume · release · **convert** | |

**No new server route is needed for a one-screen counter.** Each "save as" calls that document's
existing route.

### 2.3 Can one grid feed five documents? (line DTO fields, prefix removed)

| Document | Line fields | Shared with the bill's 113 |
|---|---|---|
| Bill | 113 | — |
| Order | 109 | **94 (83%)** |
| Quote | 100 | **95 (84%)** |
| DC | 79 | 68 (60%) — adds DcQty, OpenQty, BilledQty, ReturnedQty, SrcDocAccYear, SrcLineNo, LineStatus |
| Return | 75 | 67 (59%) — adds BillItemId, BillLineNo, BillAccYear, Condition |

Headers: 53 fields common to all five; bill has 95 of its own, order 80, quote 46, return 40, DC 35.

**Reading:** Bill / Order / Quote are the same grid with a few columns. DC and Return are
quantity-and-source driven (a return line points at a bill line; a DC line tracks open/billed
qty). One screen is feasible with a column set per type and **five payload mappers** — bounded
work, not free.

### 2.4 NexERP already has EPOSGo's disease — in its office screens

| Screen | `.cpp` lines |
|---|---|
| Sale bill (4 files) | 10,041 |
| Quotation | 3,835 |
| Sale order | 3,846 |
| Delivery challan | 3,028 |
| **Sale return** | **no Qt screen yet** (API exists) |

**42 member functions have the same name in all four screens** — `applyGstAndCess`,
`recalcDocumentTotals`, `priceRowStandard`, `applyLineDiscounts`, `fetchItemPrice`,
`recalcAfterQtyChange`, `buildSaveItemPayload`, `applyPriceLevelId`, `switchRowUom` … — and they
have **already drifted** (line-similarity to the bill's version):

| Function | Quote | Order | DC |
|---|---|---|---|
| `applyGstAndCess` | 54% | 56% | 33% |
| `applyLineDiscounts` | 94% | 94% | 25% |
| `priceRowStandard` | 66% | 75% | 35% |
| `fetchItemPrice` | 64% | 78% | 45% |
| `recalcDocumentTotals` | 72% | 92% | 8% |
| `recalcAfterQtyChange` | 18% | 27% | 16% |

So the same item can already be priced or taxed differently depending on which screen keyed it;
only the server's validate catches it. **A counter screen written as a fifth copy would make this
worse, whichever app it lives in.**

### 2.5 Speed — what actually costs time at a counter

Per scanned item the bill makes **2 GETs** (`ItemByBarcode`, `ItemPriceDetail`); a save makes
**2–4 POSTs** (validate → create → validate → post). These are identical in every option — a
separate executable makes the same calls to the same server. On a store LAN they are tens of
milliseconds. **Counter speed is decided by the screen flow** (how many fields, popups and
keystrokes per bill), not by which program the screen lives in. One server ask would help all
options: a single *create-and-post* call for counter bills.

## 3 · The options, scored on the evidence

| | A · New separate app (EPOSGo way) | B · Second executable from the NexERP repo | C · Counter module inside NexERP (Till mode) |
|---|---|---|---|
| Reuses EPOSGo code | no — DB-function architecture | no | no |
| Re-writes login/API/session/theme/printing/grids (≈32,700 lines in core + shared) | **yes** | no (links them) | no |
| Office screens touched | no | no | no |
| Build change | new project | split `CMakeLists` into a core library + 2 executables; cut `form_manager` and one grid include out of the counter build | none (one hook in `main.cpp`) |
| Installs per counter | 2 programs, kept in version step | 1 (counter exe) + office exe where needed | 1 |
| Supervisor needs an office screen at the counter | start the other app (EPOSGo's shell script) | start the other exe | PIN, same app |
| Speed | same server calls | same | same |
| Return screen | must be built | must be built | must be built (none exists yet — no extra cost to put it on the counter screen) |

**A vs B/C is the big difference**: A rebuilds ~32,700 lines of infrastructure and can reuse
nothing from EPOSGo either. **B vs C is a packaging choice**, not a code choice — if the counter
lives in its own module on the shared core, the same code can ship inside NexERP (C) or as its own
executable (B) later; B only adds the CMake split.

## 4 · The decision that matters more: the counter's engine

Wherever it lives, the counter screen needs price / discount / GST-cess / totals / price level /
UOM / promotion binding for every line. Three ways:

1. **Copy the bill's functions into the counter** — fastest start; a fifth drifting copy (§2.4).
2. **Extract a screen-free `SalesLineEngine` from the bill's versions** (the bill has the most
   complete set: 154 member functions vs 71–87) and build the counter on it. The existing screens
   are **not touched**; they can move onto the engine one at a time later, each move deleting a copy.
3. Make the server the only calculator (validate per change) — too slow for a counter.

**Option 2 is the one that does not repeat EPOSGo.** It costs the extraction up front.

## 5 · Conclusion

- **Do not build a new separate app (A).** Measured cost: re-creating ~32,700 lines NexERP already
  has, two installs per counter, versions to keep in step — and no speed gain, because the server
  calls are the same.
- **Build the counter as its own module on NexERP's core (C)**, started from the `main.cpp` hook
  on counter PCs (Till mode). Existing screens untouched.
- **Keep the module clean enough that B stays possible** (no includes from other modules except
  the shared pieces in §2.1). If counters later move to different hardware or must be installed
  without the office app, split the build then.
- **Build the counter screen on an extracted line engine**, not on copied functions.
- **Speed comes from the flow**: one screen, scan-first, F-keys, the few type-specific fields only
  when saving, and a create-and-post server call.

## 6 · What would change this conclusion

- Counters on **different hardware / OS** (Android tablets, touch-only lanes) → a separate client
  is justified (likely the React plans), still on the same API.
- Counters that must run **without the store server** (pure offline till) → a different
  architecture question, not answered by either option here.
- A **separate team** owning the counter product → B for clean ownership.

Questions back to the user: what hardware runs the counters, and do any counters run without the
store server?
