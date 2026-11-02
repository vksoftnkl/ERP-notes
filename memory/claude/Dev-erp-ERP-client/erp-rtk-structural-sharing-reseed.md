---
name: erp-rtk-structural-sharing-reseed
description: "A refetch that answers identical data keeps the SAME object reference, so an effect that seeds editor state from query data never re-runs — \"Reload\" silently keeps the edits"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 1134ba40-4fa2-4d1d-a7d9-d767812d7421
  modified: 2026-10-07T07:59:34.426Z
---

RTK Query's structural sharing returns the previous object when a refetch's
JSON is deep-equal. An editor that seeds local state in
`useEffect(() => …, [query.currentData])` therefore does NOT re-seed on a
manual Reload whose answer is unchanged — the user's discarded edits stay on
screen. Hit 2026-10-07 on the App Themes template 409 → Reload path
([[erp-app-themes-port]]).

**Why:** the reference is the dependency; equal data means no new reference.

**How to apply:** for an explicit "reload / drop my edits" action, apply the
result directly: `const fresh = await refetch().unwrap(); setText(fresh.x)`,
and update the seed key ref yourself. Keep the effect only for passive
arrival. Also guard that effect with "not dirty", because the
[[erp-data-freshness-bus]] invalidates every tag on tab focus.
