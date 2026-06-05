# Audit — `expo-tracking-transparency` dependency status (read-only)

**Repo:** oglasino-expo · **Branch:** new-expo-dev · **Date:** 2026-06-04
**Mode:** READ-ONLY (no edits, no deps, no prebuild). Code is ground truth.

## TL;DR — Verdict

**Case (b): JS usage is fully gone, but the dependency is still installed at all three
non-JS layers** — `package.json`, the lockfile, `node_modules`, AND the **already-prebuilt
native iOS Pod**. A real removal action remains. The `issues.md` 2026-06-02 "unused
dependency" entry is **accurate and still open** — it is not stale.

| Layer | Status |
|---|---|
| JS usage (imports / ATT calls) | **GONE** — zero hits |
| `package.json` declaration | **PRESENT** — `package.json:63` |
| Lockfile | **PRESENT** — `package-lock.json:57`, `:10068` |
| Native config (plist usage string / config plugin) | **GONE** — no `NSUserTrackingUsageDescription`, not in `plugins` array |
| Native module (compiled Pod in `ios/`) | **PRESENT** — `Podfile.lock` + Pod support files |

The ATT-crash fix did exactly what `issues.md` claims: it removed the JS call and the plist
usage string (so the launch crash is gone), but left the dependency and its already-installed
native module in the tree to be dropped at the next prebuild.

---

## 1. Is the package still declared?

**PRESENT in `package.json`** (`dependencies`):

```
package.json:63:    "expo-tracking-transparency": "~6.0.8",
```

Not in `devDependencies`.

**PRESENT in the lockfile** (`package-lock.json`):
- `package-lock.json:57` — the root package's `dependencies` mirror
- `package-lock.json:10068-10070` — the installed `node_modules/expo-tracking-transparency`
  entry, resolved to `expo-tracking-transparency-6.0.8.tgz`

So it is also physically installed in `node_modules`.

## 2. Any remaining imports or usage?

**ZERO hits** across `src/` and `app/` for all four search terms:
- `expo-tracking-transparency` — 0
- `requestTrackingPermissionsAsync` — 0
- `getTrackingPermissionsAsync` — 0
- `TrackingTransparency` — 0

(`grep -rn -E "expo-tracking-transparency|requestTrackingPermissionsAsync|getTrackingPermissionsAsync|TrackingTransparency" src/ app/` returned nothing.)

### `AnalyticsInit.tsx` — confirmed the call is gone, gate is consent-only

`src/components/init/AnalyticsInit.tsx` no longer references ATT at all. Its launch
reconcile (`syncAndSeed`, `AnalyticsInit.tsx:54-58`) just calls `syncAnalyticsState()` and
seeds the user_id. The file's own header comment states the intent:

```
AnalyticsInit.tsx:12-13:  *     First-party analytics only (no IDFA/ads), so no App Tracking Transparency.
```

The actual gate lives in `src/lib/analytics/analyticsClient.ts`:

```
analyticsClient.ts:7:  import { isAnalyticsConsentGranted } from '../consent/analyticsGate';
analyticsClient.ts:39:  const gateOpen = isAnalyticsConsentGranted();
```

— i.e. **consent-only**, exactly as `issues.md` describes. The `attAllowed` operand is gone
(zero `att`/`tracking` references except the explanatory comment at
`analyticsClient.ts:16-17`). No `attAllowed && …` survives; the gate reads
`isAnalyticsConsentGranted()` alone, not defaulted-false.

## 3. Any native / config references?

**`NSUserTrackingUsageDescription`:** ZERO hits tree-wide (excluding `node_modules`/`Pods`).
Not in `app.config.ts`, not in any source plist, not in `ios/OglasinoDev/Info.plist`
(direct grep of the prebuilt Info.plist for `tracking`/`NSUserTracking` returned nothing).
There is **no `app.json`** — config is `app.config.ts` only.

**`expo-tracking-transparency` in the `plugins` array:** NOT present. The `plugins` array
(`app.config.ts:114-153`) lists: `expo-router`, `expo-splash-screen`,
`@react-native-google-signin/google-signin`, `expo-secure-store`, `expo-dev-client`,
`expo-notifications`, `@react-native-firebase/app`, `expo-build-properties`. No ATT plugin.
(`expo-tracking-transparency` ships an autolinked native module; it does not require a
config-plugin entry to be compiled in — see §6.)

### BUT — the native module IS already compiled into the prebuilt `ios/`

Because `ios/` has already been prebuilt with the dependency installed, the native Pod is
baked into the current iOS project:

```
ios/Podfile.lock:  - ExpoTrackingTransparency (6.0.8):
ios/Podfile.lock:  - ExpoTrackingTransparency (from `../node_modules/expo-tracking-transparency/ios`)
ios/Podfile.lock:  ExpoTrackingTransparency: 4f823a38ff7c0efea4cac722ecbc0589278ff548
ios/Pods/Target Support Files/ExpoTrackingTransparency/ExpoTrackingTransparency-Info.plist
```

So the framework links the native module today, even though no JS calls it and there is no
usage string. Harmless at runtime while unused (the crash was caused by *calling* the API
without the plist string — there is no longer any call), but it is dead native weight.

## 4. Verdict on each of the three layers

- **JS usage:** GONE. Zero imports/calls in `src/` or `app/`. `AnalyticsInit.tsx` is
  consent-only via `isAnalyticsConsentGranted()`; `attAllowed` removed.
- **`package.json` declaration:** PRESENT. `package.json:63` (`~6.0.8`), mirrored in the
  lockfile and installed in `node_modules`.
- **Native config (plist / plugins):** GONE at the *config* level (no usage string, not in
  the `plugins` array) — but the **already-prebuilt native module** is PRESENT in `ios/`
  (`Podfile.lock` + Pod support files).

## 5. Removal status

**(b)** — JS usage gone, dependency still in `package.json` + lockfile + `node_modules`, and
its native module still compiled into the prebuilt `ios/`. A real removal action remains, and
it needs a prebuild/pod reinstall to fully drop the native module. **The `issues.md`
2026-06-02 entry is correct and should stay open** until the removal lands. (Nothing
unexpected — not case (c).)

## 6. If (b): what removal actually requires

`expo-tracking-transparency` is **not** a JS-only dep — it ships an autolinked native iOS
module (the `ExpoTrackingTransparency` Pod, present in `Podfile.lock` and `ios/Pods/Target
Support Files/`). So removing the `package.json` line alone is not sufficient to drop the
native code; the already-generated `ios/` artifacts must be regenerated.

Note: it autolinks **without** a `plugins` entry (it's a native module, not a config plugin),
which is why there's nothing to delete from the `plugins` array — the only declaration to
remove is the `package.json` dependency line.

**Actual removal steps (all currently BLOCKED by hard rules / require the next prebuild):**

1. `npm uninstall expo-tracking-transparency` — drops `package.json:63`, the lockfile entries,
   and `node_modules/expo-tracking-transparency`. *(Forbidden now: a dependency change; CLAUDE.md
   bars `npm uninstall`. Can be done by Igor / at the next dep-maintenance pass.)*
2. Regenerate native: `npx expo prebuild` (or a clean `pod install` after the uninstall) so the
   `ExpoTrackingTransparency` Pod is dropped from `ios/Podfile.lock` and `ios/Pods/…`.
   *(Forbidden now: CLAUDE.md bars prebuild. Requires the next prebuild/rebuild.)*

**Can any of it be done now?** No — both steps are dependency/native changes the hard rules
forbid this session, and the brief is explicitly read-only. Step 1 is "purely package.json +
lockfile + node_modules," but step 2 (native Pod drop) is mandatory to actually remove the
module, and it can only happen at the next prebuild. So this is a "remove at the next expo
prebuild" item, exactly as `issues.md` already records. No standalone build is warranted just
for this — fold it into the next prebuild for another reason.

## Is the `issues.md` description wrong against the real tree?

**No — it is accurate.** The 2026-06-02 "unused dependency `expo-tracking-transparency`" entry
correctly states (a) the only use (the ATT call) was removed, (b) it's now unused, and
(c) removal is a dep change (npm + pod drop) requiring a prebuild/rebuild, so it was deferred.
Every claim matches the tree. One precision worth recording: the entry says "npm + pod drop" —
confirmed, the pod (`ExpoTrackingTransparency`) is real and present in the prebuilt `ios/`, so
the "requires a prebuild" qualifier is not just cautious boilerplate, it's load-bearing.
The companion 2026-06-02 ATT-crash "RESOLVED" entry is also accurate: JS call gone, gate
consent-only, no plist string — the launch crash cause is fully removed.
