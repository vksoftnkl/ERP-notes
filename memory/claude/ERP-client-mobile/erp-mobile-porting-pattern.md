---
name: erp-mobile-porting-pattern
description: How to port a master module from the ERP web client to the RN app
metadata: 
  node_type: memory
  type: reference
  originSessionId: 996f24d2-8eb4-4f73-8a0f-1334ea7a1c0d
---

To add a ported master to the RN ERP app (see [[erp-mobile-port]]):

1. **Simple/declarative master** (no dynamic lookups): create
   `src/features/masters/<name>/module.ts` exporting `defineMasterModule({...})` with
   `apiEndpoints` (list = `/configured-grid-sql/run?grid_id=N`, getById/create/delete),
   `lookupKeys` (field-name-agnostic arrays), `requestPayloadKeys`, `customFields`,
   `createInitialValues`, `mapFormValues`, `buildRequestPayload`. Then a thin
   `<Name>MasterScreen.tsx` = `<CrudMasterScreen definition={module} />`.

2. **Master needing dynamic option lists** (e.g. city→states, unit→base units): make the
   screen a component that loads options via `useApi` in `useEffect`, then builds the
   definition inside `useMemo([options])` and renders `<CrudMasterScreen definition={def} />`.
   Mirror the web `features/masters/<name>/page.tsx` exactly (same endpoints, keys, payloads).

3. Register it in [src/navigation/masters-registry.ts](src/navigation/masters-registry.ts)
   (auto-appears in Masters tab) and optionally add a `route` to the matching leaf in
   [src/navigation/menu.ts](src/navigation/menu.ts) (drawer).

**Engine reuse:** value-mappers, normalizers, buildMasterRows/mapRowToFormState/resolveRecordId
are ported 1:1 from the web — copy module field schemas verbatim, only swap web field `type`s to
the RN DynamicForm set (text/number/checkbox/select/textarea/heading/custom). `colSpan` is ignored
(mobile stacks). Web `react-icons` Fi* → Feather icon names. Toasts via `@/lib/toast`.

**Toolchain pins (exact; this is the RN 0.81 New-Arch modern stack):**
- `react-native-reanimated@4.3.1` + `react-native-worklets@0.8.3` (Reanimated 4 needs worklets +
  New Architecture, which RN 0.81 enables by default via `newArchEnabled=true`). Per reanimated's
  `compatibility.json`: 4.3.x supports RN 0.81–0.85 (worklets 0.8.x); 4.4.x DROPPED 0.81 (needs 0.83+).
  The gradle `assertMinimalReactNativeVersionTask` fails the build on a mismatch.
- `nativewind@4.2.5` (→ `react-native-css-interop@0.2.5`) — its `nativewind/babel` preset injects
  `react-native-worklets/plugin` as its last plugin, so babel.config does NOT add reanimated/worklets
  plugin itself (only `module-resolver`). Adding it again would double-transform worklets.
- DO NOT downgrade to reanimated 3.16.7: it fails to compile against RN 0.81 native APIs
  (`Systrace.TRACE_TAG_REACT_JAVA_BRIDGE`, `LengthPercentage.resolve(int,int)`).
- Android JDK: `org.gradle.java.home=/opt/android-studio/jbr` in android/gradle.properties — the
  system `/usr/lib/jvm/*-21` are JREs (no javac).
The bundle error "Cannot find module 'react-native-worklets/plugin'" = worklets not installed or a
version got bumped. After version changes, clear caches AND android build dirs:
`rm -rf android/app/build android/build android/.gradle node_modules/react-native-reanimated/android/build`
then:
`rm -rf node_modules/.cache /tmp/metro-* /tmp/haste-map-*` then bundle with `--reset-cache`.
Validate with: `npx react-native bundle --platform android --dev true --entry-file index.js
--bundle-output /tmp/b.js --assets-dest /tmp/a` and `npx tsc --noEmit`.
