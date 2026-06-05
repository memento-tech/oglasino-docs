# Session — iOS Firebase build fix (Expo SDK 54 / RN 0.81 / Xcode 26)

- **Repo:** oglasino-expo
- **Slug:** ios-firebase-nonmodular-fix (order 1)
- **Date:** 2026-06-02
- **Branch:** new-expo-dev (not committed by me — Igor commits & cuts the build)
- **Trigger:** EAS iOS build failure pasted by Igor (no feature brief). Toolchain/native-config debugging task, not a feature adoption.
- **Final status:** GREEN. `expo run:ios` → `Build Succeeded`, 0 errors, installed + launched on iPhone 17 Pro (iOS 26) simulator locally (Xcode 26.2, matching the EAS image).

## Final working config — the whole fix

`app.config.ts`, `expo-build-properties` ios block:

```ts
ios: {
  useFrameworks: 'static',
  buildReactNativeFromSource: true,
}
```

That is the entire change. **No** custom config plugin, **no** `forceStaticLinking`, **no** `$RNFirebaseAsStaticFramework`, **no** `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES`. All of those were intermediate dead-ends (see below) and were removed.

## Root cause

RN 0.81 / SDK 54 ship React Native core as a **precompiled `React.xcframework`** (prebuilt core; the Podfile sets `RCT_USE_PREBUILT_RNCORE=1` by default). Firebase + Google Sign-In force `useFrameworks: 'static'`, so the RNFB pods are static **framework modules** that `#import <React/...>` headers out of the precompiled React core. Xcode 26's **Explicitly Built Modules** turns the resulting non-modular-include and cross-module-ordering diagnostics into hard `-Werror`. The boundary is the prebuilt core. Building React from source (`buildReactNativeFromSource: true`) removes the precompiled-framework boundary; React's headers then resolve as normal source/modular headers and every layer disappears.

## Debugging journey — what was tried and why each was wrong/insufficient

| Attempt | Result | Why |
|---|---|---|
| Custom plugin: `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES = YES` on `RNFB*` targets (post_install) | Cleared layer-1 RNFBApp non-modular errors, then failed deeper in RNFBAnalytics | Suppresses the warning on the target, but under explicit modules the module **precompile** action (`PrecompileModule RNFBApp-…scan`) doesn't honor it against the prebuilt-core header. |
| Plugin: `$RNFirebaseAsStaticFramework = true` (top-level) | No effect; cross-module error byte-identical | `static_framework` podspec attr is a near-no-op when `use_frameworks! :linkage => :static` is already in force — all pods are already static frameworks. |
| `forceStaticLinking: ['RNFBApp','RNFBAnalytics']` (expo-build-properties) | Fixed layer-3 cross-module error; layer-2 returned | Verified in generated Pods project: RNFB products became `libRNFBApp.a` / `libRNFBAnalytics.a` (static libs) AND `CLANG_ALLOW` did land (4 build-config entries). But RNFBApp still `DEFINES_MODULE = YES`, so its module still precompiles and still imports the prebuilt-core `<React/RCTConvert.h>` → layer-2 `-Werror`. forceStaticLinking removes the inter-pod framework boundary, not RNFBApp's dependence on the prebuilt React core. |
| **`buildReactNativeFromSource: true`** (alone, plugin + forceStaticLinking removed) | **GREEN** | Removes the precompiled React core entirely — the actual root. |

Deployment-target warnings (SDWebImage / PromisesObjC / GTMSessionFetcher iOS 9/10) were confirmed **red herrings**: harmless transitive-pod minimums; the app floors at iOS 15.1. They persist as warnings in the successful build.

## How it was validated

Local environment matches EAS: **Xcode 26.2 / clang 17 / iOS 26 simulators**, CocoaPods 1.16.2. So local repro is representative of the Xcode 26 explicit-modules behavior.

- `expo prebuild --clean -p ios` + `pod install` succeeded with the final config.
- Confirmed from-source switch in `Podfile.properties.json` (`ios.buildReactNativeFromSource: "true"`, `ios.forceStaticLinking: "[]"`), and 0 plugin injections in the generated Podfile.
- `expo run:ios` (iPhone 17 Pro sim): React Native compiled from source, RNFBApp/RNFBAnalytics built with no non-modular / cross-module errors → `Build Succeeded`, 0 errors, app installed and launched.
- `npx eslint app.config.ts` exit 0; `npx tsc --noEmit` clean; `expo config` resolves with `buildReactNativeFromSource: true`.

## Changes on disk (staged, not committed)

1. `app.config.ts` — `expo-build-properties.ios` now `{ useFrameworks: 'static', buildReactNativeFromSource: true }`. `useFrameworks: 'static'` unchanged; `forceStaticLinking` removed; plugin registration removed.
2. **Deleted** `plugins/withFirebaseNonModularFix.js` and the now-empty `plugins/` directory (superseded — no longer referenced anywhere).

`ios/` is gitignored / CNG-regenerated, so nothing there is committed; the fix lives entirely in `app.config.ts`.

## Trade-off Igor should know

`buildReactNativeFromSource: true` **disables the precompiled React Native binaries**, so iOS builds (local and EAS) compile all of React Native from source — **noticeably longer build times** (this session's from-source compile was the bulk of the wall-clock). This is the accepted cost until upstream resolves the prebuilt-core + `use_frameworks!` + Firebase interaction on SDK 54 (refs: expo/expo#39233, expo/expo#39607, invertase/react-native-firebase#8657). Revisit on the next Expo SDK / RNFirebase bump — if upstream fixes the prebuilt-core path, this flag can be removed to restore faster builds.

## Cleanup performed

- Deleted superseded `plugins/withFirebaseNonModularFix.js` + empty `plugins/` dir.
- Removed `forceStaticLinking` and the plugin registration from `app.config.ts`.
- No commented-out code, no debug logging, no unused imports, no TODO/FIXME left.

## Obsoleted by this session

The entire `plugins/withFirebaseNonModularFix.js` approach (CLANG_ALLOW + `$RNFirebaseAsStaticFramework`) and the `forceStaticLinking` config — all superseded by `buildReactNativeFromSource: true` and removed from disk.

## Brief vs reality

N/A — build-failure triage, no feature brief.

## Conventions check

- **Part 4 (cleanliness):** eslint + tsc clean on touched paths; superseded plugin/dir deleted; no dead code/logging/TODOs. No dependency changes (no package.json edits) → expo-doctor not required.
- **Part 4a (simplicity):** final fix is a single supported `expo-build-properties` flag — strictly simpler than the abandoned custom-plugin approach.
- **Part 4b (adjacent observations):** the deployment-version warnings are cosmetic; could be silenced later with `ios.deploymentTarget` but not needed. Any future RNFB submodules (auth, firestore, messaging) are covered by the same from-source fix with no per-module work.

## Config-file impact

No edits to the four docs-repo config files (conventions.md, decisions.md, state.md, issues.md) — Docs/QA is sole writer. No Expo backlog row adopted/cleared (build fix, not a feature). Suggested decisions.md entry below.

## For Mastermind

Suggested `decisions.md` entry (Docs/QA to write — do not let me edit it):

> **Decision:** `oglasino-expo` sets `expo-build-properties.ios.buildReactNativeFromSource: true` (alongside `useFrameworks: 'static'`) to build the iOS app. Required on Expo SDK 54 / RN 0.81 / Xcode 26: the precompiled `React.xcframework` + `use_frameworks! static` (needed by Firebase + Google Sign-In) + explicit modules produces hard `-Werror` non-modular / cross-module failures in the react-native-firebase pods. Building RN from source removes the precompiled-framework boundary and is the only approach that built green (verified locally on Xcode 26.2 → `Build Succeeded`). Intermediate workarounds (a CLANG_ALLOW / `$RNFirebaseAsStaticFramework` config plugin, and `forceStaticLinking`) were tried and rejected as insufficient. **Cost:** longer iOS build times (no precompiled RN binaries). Revisit on the next Expo SDK / RNFirebase bump — upstream may fix the prebuilt-core path, allowing this flag to be dropped. Refs: expo/expo#39233, expo/expo#39607, invertase/react-native-firebase#8657.

No open questions. Igor commits and cuts the EAS build; the same config that built green locally on Xcode 26.2 applies on the EAS Xcode 26 image.
