---
name: erp-rtk-lazy-trigger-in-mount-effect
description: RTK Query lazy triggers resolve to undefined when fired from a mount effect — dispatch endpoints.X.initiate instead
metadata:
  type: project
---

A `useLazyXQuery()` trigger fired from a **mount effect** resolves with
`undefined` and never touches the network: the subscription does not exist yet.
It surfaces as a TypeError on the payload you expected —
"Cannot read properties of undefined (reading 'header')" — over a screen that
merely looks like it failed to load.

**Why:** the screen looks broken in a way that points at the response shape, so
the hunt starts in the parser rather than in the call.

**How to apply:** any read that a route can invoke on mount goes through the
store instead —

```ts
const payload = await dispatch(
  someApi.endpoints.getThing.initiate(arg, { subscribe: false, forceRefetch: true }),
).unwrap();
```

Hit twice now: the quotation port (`use-quotation-draft.ts` `loadDocument`) and
the receipt port, whose `/accounts/receipt/:company/:branch/:year/:id` route
opens the document from a mount effect. In `use-receipt-draft.ts` all five
reads are wrapped this way. Mutations are unaffected.

See [[erp-receipt-entry-port]].
