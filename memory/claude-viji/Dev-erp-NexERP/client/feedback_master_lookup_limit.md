---
name: master_lookup_no_limit_param
description: "Don't include limit parameter in master-lookup API calls from frontend"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e590d205-6262-4d28-b346-32cb6dbcc7db
---

**Rule:** Do not send the `limit` parameter with `/master-lookup` API calls from the frontend.

**Why:** The backend `/master-lookup` endpoint doesn't expect or handle a limit parameter; sending it causes issues.

**How to apply:** When making requests to `/master-lookup` from client code, omit any `limit` query parameter or payload field. Only send the parameters the endpoint actually expects (e.g., module, name search criteria).
