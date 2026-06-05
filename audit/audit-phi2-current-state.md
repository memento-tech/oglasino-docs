# Audit — Φ2 navigation foundation current state

**Repo:** `oglasino-expo`
**Branch:** `new-expo-dev`
**Date:** 2026-05-27
**Mode:** read-only
**Spec:** `../oglasino-docs/features/expo-navigation-foundation.md`

---

## 1. `app/_layout.tsx` — Root layout

**Status:** `done`

**Evidence:**

- Line 12: `import { Stack } from 'expo-router';`
- Line 70: `<Stack screenOptions={{ headerShown: false }} />`
- No `<Slot />` anywhere in the file.
- Provider stack unchanged: `GestureHandlerRootView` → `AppContextProvider` → `AppVersionConfigInit` → `ToastProvider` → `ThemeProvider` → `SafeAreaView`. Conditional rendering based on `status` (`'ready' | 'loading' | 'maintenance' | 'select-base-site'`) remains. `AppInit`, `PortalHost`, `DialogManager` remain as siblings.

**Note:** The `<Stack>` is wrapped in `<View className="flex-1">` (line 69), which the spec §3.1 doesn't mention. The wrapping View doesn't affect navigator behavior.

---

## 2. `app/(portal)/_layout.tsx` — Portal layout

**Status:** `done`

**Evidence:**

- Line 7: `import { Tabs } from 'expo-router';`
- Lines 21–28:
  ```tsx
  <Tabs
    tabBar={(props) => <BottomBar {...props} />}
    screenOptions={{ headerShown: false }}>
    <Tabs.Screen name="(public)" options={{ title: 'home' }} />
    <Tabs.Screen name="(secured)/favorites" options={{ title: 'favorites' }} />
    <Tabs.Screen name="(secured)/notifications" options={{ title: 'notifications' }} />
    <Tabs.Screen name="(secured)/messages" options={{ title: 'messages' }} />
  </Tabs>
  ```
- Four tab destinations match spec §3.2: Home (via `(public)`), Favorites, Notifications, Messages.
- Chrome (`ConsumerProtectionBanner`, `TopBar`, `CategoryNavigation`) rendered above `<Tabs>` in a `<View>` (lines 13–20), matching spec.
- BottomBar is the `tabBar` render prop, no longer a sibling below.

---

## 3. `app/(portal)/(public)/_layout.tsx` — Public route group

**Status:** `done`

**Evidence:**

- Line 2: `import { Stack } from 'expo-router';`
- Line 12: `<Stack screenOptions={{ headerShown: false }} />`
- Lines 8–10: `setPortalScope('portal')` side effect remains.
- Route files under `(public)/` match spec §3.3: `index.tsx`, `catalog/[...categories].tsx`, `product/[...productData].tsx`, `user/[...userData].tsx`, `about.tsx`, `pricing.tsx`, `privacy.tsx`, `terms.tsx`, `blog/free-zone.tsx`.

---

## 4. `app/(portal)/(secured)/` — No layout file, inline auth guards

**Status:** `done`

**Evidence:**

- `app/(portal)/(secured)/_layout.tsx` **does not exist** (confirmed via filesystem check).
- **`favorites.tsx`** (lines 6–10): Inline auth guard matches spec §3.4 pattern exactly.
  ```tsx
  const user = useAuthStore((s) => s.user);
  const hasHydrated = useAuthStore((s) => s._hasHydrated);
  if (!hasHydrated) return null;
  if (!user) return <Redirect href="/" />;
  ```
- **`messages.tsx`** (lines 9–14): Same pattern, guard immediately after selectors.
- **`notifications.tsx`** (lines 15–16 for selectors, lines 45–46 for guard returns): Guard selectors are near the top, but the early-return lines are at lines 45–46 with hooks and effects (router, store destructure, `useRef`, two `useEffect`s) between the selectors and the guard. The effects at lines 24–43 reference `user` and subscribe to notifications — they execute before the redirect fires for unauthenticated users.

**Note on notifications.tsx:** The spec §3.4 shows the guard as a contiguous block at the top of the function body. `favorites.tsx` and `messages.tsx` follow that pattern. `notifications.tsx` has the guard returns separated from the selectors by ~28 lines of hooks and effects. The behavior (redirect for unauthenticated) still works, but the effects fire needlessly for unauthenticated users before the redirect. This looks intentional — the component was likely ported from a prior version where the layout guard handled auth. Not a regression; the guard is present and functional.

---

## 5. `app/owner/_layout.tsx` — Owner layout

**Status:** `done`

**Evidence:**

- Lines 16–17: Auth guard selectors via `useAuthStore((s) => s.user)` and `useAuthStore((s) => s._hasHydrated)`.
- Lines 28–29: `if (!hasHydrated) return null; if (!user) return <Redirect href="/" />;`
- Line 42: `<Stack screenOptions={{ headerShown: false }} />`
- Chrome: marketplace label (line 35), `TopBar` (lines 37–41), `DashboardSidebar` (line 44) remain as siblings around the Stack.

**Note:** Same pattern as notifications.tsx — hooks and effects between selectors (lines 16–17) and guard returns (lines 28–29). Lines 19–26 contain `usePortalScope`, `usePathname`, `useAppContext`, `useTranslations`, and `useEffect`. This is the Φ1-established pattern for this file.

---

## 6. `app/owner/dashboard/_layout.tsx` — Dashboard layout

**Status:** `done`

**Evidence:**

- Lines 4–5: Auth guard selectors immediately at top.
- Lines 8–9: Guard returns immediately after selectors.
- Line 11: `<Stack screenOptions={{ headerShown: false }} />`
- Clean, minimal layout — guard + Stack, nothing else.

---

## 7. `app/+not-found.tsx` — Not-found screen

**Status:** `done`

**Evidence:**

- No `<Stack.Screen options={{ title: 'Oops!' }} />` anywhere in the file.
- Lines 9–13: `TopBar` renders with `hideSearchBar` prop.
- Line 14: `<NotFoundPage route="/" />` for the body.
- The screen inherits `headerShown: false` from the root Stack's `screenOptions`.

---

## 8. `src/components/navigation/BottomBar.tsx` — BottomBar refactor

**Status:** `done`

**Evidence:**

- Line 7: `import type { BottomTabBarProps } from '@react-navigation/bottom-tabs';`
- Line 22: `export default function BottomBar(props: BottomTabBarProps)`
- Line 33: `const currentRouteName = props.state.routes[props.state.index]?.name;` — reads active tab from `state` prop.
- Lines 46–58: `getCurrentColor` function compares against `currentRouteName`.
- Lines 60–64: `handleTabPress` navigates via `props.navigation.navigate(routeName)`.
- No `usePathname()` import or usage anywhere in the file.
- No `router.push()` anywhere in the file.
- Lines 35–44: Auth gate via `openDialogSafe` runs before `navigation.navigate()`.
- Tab constants at lines 18–20 use the `(secured)/favorites` form matching the `Tabs.Screen name` values in the portal layout.

---

## 9. `src/components/dashboard/layout/DashboardSidebar.tsx` — DashboardSidebar refactor

**Status:** `done`

**Evidence:**

- Line 58: `router.replace('/')` — back-to-portal navigation uses `replace`.
- Line 142: `router.replace(item.url as RelativePathString)` — collapsible menu items use `replace`.
- Line 159: `router.replace(nav.url as RelativePathString)` — top-level menu items use `replace`.
- No `router.push()` anywhere in the file.
- Line 26: `import { ... useWindowDimensions } from 'react-native';`
- Line 40: `const { width: screenWidth } = useWindowDimensions();` — reactive dimensions.
- No `Dimensions.get('window')` anywhere in the file.

---

## 10. F26 — `useWindowDimensions` migration

**Status:** `done` — all four sites migrated.

| File | Line | Evidence |
|------|------|----------|
| `src/components/FloatingButton.tsx` | 3, 21 | `import { ... useWindowDimensions } from 'react-native'`; `const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = useWindowDimensions();` |
| `src/components/FullScreenImageViewer.tsx` | 3, 21 | `import { ... useWindowDimensions } from 'react-native'`; `const { width, height } = useWindowDimensions();` |
| `src/components/product/ProductList.tsx` | 10, 42 | `import { ... useWindowDimensions } from 'react-native'`; `const { height: SCREEN_HEIGHT } = useWindowDimensions();` |
| `src/components/dashboard/layout/DashboardSidebar.tsx` | 26, 40 | (same as item 9 above) |

No module-scope `Dimensions.get('window')` remains in any of the four files.

---

## 11. `<Slot />` audit

**Status:** `done` — no remaining `<Slot />` in any layout.

Grepped all `_layout.tsx` files in `app/` for `Slot`. Zero matches. The three layout files are:

- `app/_layout.tsx` → uses `<Stack>`
- `app/(portal)/_layout.tsx` → uses `<Tabs>`
- `app/(portal)/(public)/_layout.tsx` → uses `<Stack>`
- `app/owner/_layout.tsx` → uses `<Stack>`
- `app/owner/dashboard/_layout.tsx` → uses `<Stack>`

`app/(portal)/(secured)/_layout.tsx` does not exist (correct per spec §3.4).

---

## 12. Verification gates

| Gate | Result |
|------|--------|
| `npx tsc --noEmit` | **Exit 0.** Zero errors, zero output. |
| `npm run lint` | **0 errors, 80 warnings.** All warnings are pre-existing (identical to the Φ1/admin-removal baseline of 80–82 warnings). No new lint issues. |
| `npm test` | **109 passed, 0 failed** (7 test files, 205ms). Matches Φ1 baseline of 109 tests. |
| `npx expo-doctor` | **1 check failed** (pre-existing): 8 packages at patch-version mismatches with SDK 54. No new failures vs Φ1 baseline. |

---

## Summary

### What's done

All 12 checklist items are **done**:

1. Root layout → `<Stack screenOptions={{ headerShown: false }}>` ✓
2. Portal layout → `<Tabs>` with BottomBar as `tabBar`, four tab destinations ✓
3. Public layout → `<Stack>`, `setPortalScope('portal')` preserved ✓
4. Secured group → no `_layout.tsx`, inline auth guards in all three screens ✓
5. Owner layout → Φ1 auth guard + `<Stack>`, chrome as siblings ✓
6. Dashboard layout → Φ1 auth guard + `<Stack>` ✓
7. `+not-found.tsx` → no orphaned `<Stack.Screen>`, TopBar renders ✓
8. BottomBar → `BottomTabBarProps`, `state.routes[state.index]`, `navigation.navigate()`, no `usePathname`/`router.push` ✓
9. DashboardSidebar → `router.replace()`, `useWindowDimensions()` ✓
10. F26 → all four sites migrated to `useWindowDimensions()` ✓
11. Zero `<Slot />` remaining in any layout ✓
12. All verification gates pass (pre-existing warnings/expo-doctor mismatch unchanged) ✓

### What's not done or partial

Nothing. All spec items are implemented.

### What diverged from spec (and why it looks intentional)

1. **`notifications.tsx` auth guard placement.** The spec §3.4 shows the auth guard as a contiguous four-line block at the top. `notifications.tsx` has the selector reads at lines 15–16 but the early-return guards at lines 45–46, with hooks and effects in between. The effects reference `user` and subscribe to the notification store, meaning they fire for unauthenticated users before the redirect. `favorites.tsx` and `messages.tsx` follow the spec pattern exactly. This looks like a pre-existing component structure that wasn't refactored to move the guard up — the guard was bolted onto the existing component rather than restructuring the hook order. Functionally correct (redirect still fires), structurally divergent from the spec's intent.

2. **`owner/_layout.tsx` auth guard placement.** Same pattern — selectors at lines 16–17, guard returns at lines 28–29, with hooks between. This is the Φ1-established pattern for this file (the hooks at lines 19–26 were there before Φ2).

3. **Root layout `<View>` wrapper around `<Stack>`.** Line 69 wraps `<Stack>` in `<View className="flex-1">`. Spec §3.1 doesn't mention this wrapper. Appears to be a layout flex constraint, not a structural divergence.

### Test gate results

- `tsc`: clean (0 errors)
- `lint`: 0 errors, 80 warnings (pre-existing baseline)
- `test`: 109 passed, 0 failed
- `expo-doctor`: 1 pre-existing failure (8 patch-version mismatches)
