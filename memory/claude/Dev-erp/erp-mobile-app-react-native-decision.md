---
name: erp-mobile-app-react-native-decision
description: "2026-10-31 decision — the ERP mobile app will be React Native (Expo), chosen over Qt-QML for growth; the user is a Qt/C++ developer new to React"
metadata:
  node_type: memory
  type: project
  originSessionId: abfff249-c0bc-419e-b972-f5ef8cfc9486
  modified: 2026-10-31T06:38:18.954Z
---

On 2026-10-31 the user decided to build the ERP mobile app in **React Native (Expo)**, not Qt-QML. Their stated reason: React Native is "the better way to grow", meaning skills and ecosystem. They know they are new to React.

Facts behind the choice (checked 2026-10-31):
- NexERP is pure Qt Widgets (279 `.ui`, ~2,300 `.cpp`, 0 QML), so Qt mobile would have meant rewriting every screen in QML anyway.
- About 900 of the 1,029 `ERP client/features/**/*.ts` files don't use browser APIs (api, types, payload, validate, state). React Native can reuse them.
- The web client authenticates with a Bearer token (`lib/api/client.ts`), so mobile login needs no server change. Only token storage moves to SecureStore.
- Qt-QML was the runner-up. It could reuse `NexERP/src/core` (33 of 41 files are widget-free: api_client, auth, session, config).

Still open: which ERP jobs go on mobile, whether it must work offline, and whether iOS is needed or Android only.

**Why:** the user prefers learning the wider ecosystem over staying inside their current Qt skill set.
**How to apply:** don't re-open Qt-QML vs React Native. Explain React code in Qt terms ([[user-qt-developer-new-to-react]]). Start with a standalone Expo pilot. Extract the shared-code package from the ERP client only after the pilot works, not as a beginner's first refactor.
