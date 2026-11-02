---
name: user-qt-developer-new-to-react
description: The user knows Qt/C++ (NexERP) but is not familiar with React; explain React/React Native in Qt terms
metadata:
  node_type: memory
  type: user
  originSessionId: abfff249-c0bc-419e-b972-f5ef8cfc9486
  modified: 2026-10-31T06:38:22.292Z
---

The user works in Qt/C++ (Qt Widgets, the NexERP desktop app) and said on 2026-10-31 that they are "not familiar in react". They chose to learn React Native for the ERP mobile app ([[erp-mobile-app-react-native-decision]]).

When explaining React or React Native code, map it to Qt ideas they already know: components ≈ widgets, props/callbacks ≈ signals/slots, state ≈ Q_PROPERTY + notify, FlatList ≈ model/view. Keep the explanations beginner-friendly for React, but don't over-explain general programming.
