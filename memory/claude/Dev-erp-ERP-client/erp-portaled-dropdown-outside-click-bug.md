---
name: erp-portaled-dropdown-outside-click-bug
description: "Custom combobox outside-click handler must check the portaled dropdown ref too, or option clicks silently no-op"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 59166b0b-1e68-4bcc-8b0c-b907db8e574a
  modified: 2026-07-23T05:25:04.581Z
---

When building a custom searchable dropdown/combobox that renders its menu via `createPortal(..., document.body)` (needed to escape a scrolling/overflow:auto ancestor like a table), the global `mousedown` "close on outside click" listener must check containment against BOTH the trigger wrapper ref AND a ref on the portaled dropdown content — not just the wrapper.

**Why:** the portaled menu lives outside the wrapper element in the DOM tree, so a plain `wrapperRef.current?.contains(target)` check treats any click inside the menu (e.g. on an option) as "outside" and calls `setIsOpen(false)` on `mousedown` — before the option's own `click` handler fires. React then unmounts the (now-closed) menu between `mousedown` and `click`, so the click event never reaches the option and the selection silently never applies (trigger button just keeps showing the placeholder, no error, no console warning). Caught this building `ItemSearchCell` in [[erp-item-qty-price-page]] — Playwright driving confirmed the item stayed unselected after a scripted click until the fix landed.

**How to apply:** any time you write a bespoke combobox instead of reusing `components/design-system/ui/searchable-select.tsx` (`SearchableSelect`, which already gets this right via a `dropdownRef` check), give the portaled menu its own ref and include it in the outside-click guard. Also prefer `onMouseDown` + `preventDefault()` on the option buttons over plain `onClick` — matches `SearchableSelect`'s `commitSelection` pattern and avoids the same race by construction.

Also note: `SearchableSelect` itself only client-filters a pre-loaded `options` array — it has no `onSearchChange`/server-search hook. It's fine for small-to-medium option sets (units, price modes, company/branch) but not for large catalogs (items) that need live server-side search-as-you-type; for those, a dedicated debounced-search cell (like `ItemSearchCell`) is still needed.
