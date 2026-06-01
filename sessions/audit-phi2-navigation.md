# Φ2 Navigation Foundation — Structural Audit

**Repo:** `oglasino-expo`
**Date:** 2026-05-25
**Mode:** read-only — no code changes
**Audit source:** code-grounded, every finding cites file path + line numbers

---

## Section 1 — Layout files

### `app/_layout.tsx`

**Renders:** `<Slot />` (line 66)

**Provider/effect wrap (outside-in):**
1. `GestureHandlerRootView` (line 52)
2. `AppContextProvider` (line 53)
3. `AppVersionConfigInit` (line 54)
4. `ToastProvider` (line 55)
5. `ThemeProvider` from `@react-navigation/native` (line 56)
6. `SafeAreaView` from `react-native-safe-area-context` (line 57)

**Auth guard:** none.

**Conditional rendering:** `status` state controls what renders. `<Slot />` only appears inside the `(status === 'ready' || status === 'loading')` branch (line 62). When status is `undefined`, shows `LoadingOverlay`; when `'maintenance'` or `'select-base-site'`, shows `BaseSiteSelector`. This means the `<Slot />` (and any future navigator replacing it) is conditionally mounted.

**Siblings to `<Slot />`:** `AppInit` (line 64), `PortalHost` (line 68), `DialogManager` (line 69). All rendered as siblings inside a `<View className="flex-1">` wrapper (line 65).

**Other:** Module-scope `NavigationBar.setVisibilityAsync('hidden')` at line 21 (Android navigation bar). Visibility listener effect at lines 29-49 auto-hides after 3 seconds.

**Child route groups:** `(portal)/`, `owner/`, `__smoke__/`, `+not-found.tsx`.

---

### `app/(portal)/_layout.tsx`

**Renders:** `<Slot />` (line 21)

**Wrap:** none — returns a fragment. A `<View>` wraps the header elements (line 13) but `<Slot />` is a direct sibling to `BottomBar`, not inside any wrapper.

**Auth guard:** none.

**Components mounted:**
- `ConsumerProtectionBanner` (line 14)
- `TopBar` portal variant (lines 15-18) — receives `usePortalFilterStore` and `getPortalAutocompleteSuggestions`
- `CategoryNavigation` (line 19)
- `BottomBar` (line 22)

**Layout structure:** header elements (banner + TopBar + categories) are in a `<View>` above `<Slot />`, and `BottomBar` is below. This is the main "portal shell."

**Child route groups:** `(public)/`, `(secured)/`.

---

### `app/(portal)/(public)/_layout.tsx`

**Renders:** `<Slot />` (line 14, inside a fragment)

**Auth guard:** none.

**Side effect:** `setPortalScope('portal')` on mount (lines 8-9).

**Child routes:** `index.tsx` (home), `about.tsx`, `pricing.tsx`, `privacy.tsx`, `terms.tsx`, `catalog/[...categories].tsx`, `product/[...productData].tsx`, `user/[...userData].tsx`, `blog/free-zone.tsx`.

---

### `app/(portal)/(secured)/_layout.tsx`

**Renders:** `<Slot />` (line 11)

**Auth guard:**
- Reads `user` via `useAuthStore((s) => s.user)` (line 5)
- Reads `_hasHydrated` via `useAuthStore((s) => s._hasHydrated)` (line 6)
- Returns `null` while not hydrated (line 8)
- Returns `<Redirect href="/" />` if no user (line 9)
- Returns `<Slot />` if authenticated (line 11)

**Child routes:** `favorites.tsx`, `messages.tsx`, `notifications.tsx`.

---

### `app/owner/_layout.tsx`

**Renders:** `<Slot />` (line 42)

**Auth guard:** same pattern as secured layout (lines 16-17, 28-29).

**Additional state reads:**
- `usePortalScope()` for `setPortalScope('dashboard')` (lines 19, 24-26)
- `usePathname()` to conditionally hide search bar (line 20, 40)
- `useAppContext()` for `selectedBaseSite` (line 21)
- `useTranslations(TranslationNamespace.INTRO)` for marketplace label (line 22)

**Components mounted (when authenticated):**
- Marketplace label header (line 35)
- `TopBar` dashboard variant (lines 37-41) — receives `useDashboardFilterStore`, `getDashboardAutocompleteSuggestions`, and `hideSearchBar` based on pathname
- `DashboardSidebar` (line 44) — renders the dashboard bottom bar + animated slide-out sidebar

**Child routes:** `dashboard/`.

---

### `app/owner/dashboard/_layout.tsx`

**Renders:** `<Slot />` (line 11)

**Auth guard:** same pattern as secured layout (lines 4-5, 8-9).

**Child routes:** `user.tsx`, `products/index.tsx`, `products/[productId].tsx`, `balance.tsx`, `follows.tsx`, `reviews.tsx`, `analytics.tsx`, `account-verification.tsx`, `not-ready.tsx`.

---

### Summary table

| Layout | Navigator | Auth guard | Custom chrome |
|--------|-----------|------------|---------------|
| `app/_layout.tsx` | `<Slot />` | none | SafeAreaView, providers, AppInit, DialogManager |
| `app/(portal)/_layout.tsx` | `<Slot />` | none | TopBar, BottomBar, CategoryNavigation, ConsumerProtectionBanner |
| `app/(portal)/(public)/_layout.tsx` | `<Slot />` | none | portalScope setter only |
| `app/(portal)/(secured)/_layout.tsx` | `<Slot />` | user + _hasHydrated | none |
| `app/owner/_layout.tsx` | `<Slot />` | user + _hasHydrated | TopBar (dashboard), DashboardSidebar, marketplace label |
| `app/owner/dashboard/_layout.tsx` | `<Slot />` | user + _hasHydrated | none |

All six layouts use bare `<Slot />`. Zero native navigators.

---

## Section 2 — Bottom bar surface

### Component location

`src/components/navigation/BottomBar.tsx`

### Where rendered

Rendered once in `app/(portal)/_layout.tsx:22`. This means the bottom bar is visible on all portal routes (both public and secured) but NOT on owner/dashboard routes. The owner layout has its own bottom bar via `DashboardSidebar.tsx`.

### Navigation mechanism

All navigation uses `router.push()` from expo-router's `useRouter()` (line 24):

| Destination | Route | Line | Gate |
|-------------|-------|------|------|
| Favorites | `router.push('/favorites')` | 64 | `openDialogSafe` (auth gate) |
| Notifications | `router.push('/notifications')` | 103 | `openDialogSafe` (auth gate) |
| New Product | `openDialog(DialogId.NEW_PRODUCT_DIALOG)` | 141 | `openDialogSafe` (auth gate) |
| Messages | `router.push('/messages')` | 149 | `openDialogSafe` (auth gate) |
| User Menu | opens `UserMenu` component | 184 | `openDialogSafe` (auth gate) |

### Active tab determination

`usePathname()` at line 21 provides the current path. The `getCurrentColor` function (lines 42-54) checks `pathname.startsWith(dedicatedPathname)` to determine active state. Each tab's label and icon color are styled based on this check:
- Favorites: `pathname.startsWith('/favorites')` (lines 73, 90)
- Notifications: `pathname.startsWith('/notifications')` (lines 112, 129)
- Messages: `pathname.startsWith('/messages')` (lines 157, 175)

Additionally, `getCurrentColor` has an `isGreen` parameter that overrides with green when a count > 0 (e.g., favoriteIds.length > 0).

### Auth-related logic

The `openDialogSafe` function (lines 31-40):
1. If `!_hasHydrated` → do nothing (no-op)
2. If `!user` → open `LOGIN_OPTIONS_DIALOG` with optional description
3. If user exists → execute the `onAuthenticated` callback

This prevents unauthenticated users from navigating to secured routes. Instead they see a login dialog.

### Store subscriptions (re-render concern for Φ3)

BottomBar reads from multiple stores without selectors:
- `useAuthStore()` destructured: `{ user, _hasHydrated }` (line 26) — whole-store subscription
- `useFavoritesStore()` destructured: `{ favoriteIds }` (line 27) — whole-store subscription
- `useNotificationStore()` destructured: `{ unseenCount }` (line 28) — whole-store subscription
- `useChatStore()` destructured: `{ newMessagesCount }` (line 29) — whole-store subscription

---

## Section 3 — Top bar surface

### Component location

`src/components/navigation/TopBar.tsx`

### Where rendered

1. `app/(portal)/_layout.tsx:15-18` — portal variant, with `usePortalFilterStore` and `getPortalAutocompleteSuggestions`
2. `app/owner/_layout.tsx:37-41` — dashboard variant, with `useDashboardFilterStore`, `getDashboardAutocompleteSuggestions`, and conditional `hideSearchBar`

### Component structure

TopBar renders (lines 31-64):
- Home link (Oglasino logo + optional baseSite icon) — only when `currentPortalScope === 'portal'` (line 33). Uses `<Link href={'/'}>` from expo-router.
- Search bar (`SearchInput` component) — hidden when `hideSearchBar` prop is true.
- Config button — opens `PortalConfigDialog` via `openDialog`.

### Back-button behavior

**TopBar has no back button.** There is no back navigation in the TopBar component itself.

Per-screen back buttons exist but are rendered by individual screen components, not by the top bar:
- `app/(portal)/(public)/product/[...productData].tsx:210` — `router.back()` via a Pressable with ArrowLeft icon
- `app/owner/dashboard/products/[productId].tsx:210` — `router.back()` via Button
- `src/components/product/UserProductsList.tsx:83` — `router.back()` via Pressable
- `src/components/BackToHomeButton.tsx:16` — `router.push('/')` (not router.back, navigates to home)

### iOS swipe-back gesture

**Absent.** No native `<Stack>` navigator exists, so there is no swipe-back gesture. No `gestureEnabled` configuration found anywhere. No manual gesture handling for back navigation.

### Android hardware back button

**Absent.** No `BackHandler` usage found anywhere in the codebase. The Android back button behavior is entirely default (controlled by expo-router/React Navigation, but without a `<Stack>` navigator, there's no proper stack to pop).

### Title/heading

TopBar does not render a page title. Titles/headings are handled per-screen inside the screen components themselves (e.g., breadcrumbs in product page, section headers in dashboard pages).

---

## Section 4 — Routes

### `app/(portal)/(public)/` — public portal routes

| Route file | Description | Has own header/back? | Navigation logic |
|------------|-------------|---------------------|------------------|
| `index.tsx` | Home — FilteredProductList with portal card | No | None |
| `about.tsx` | About page — static content with BackToHomeButton | BackToHomeButton (`router.push('/')`) | None |
| `pricing.tsx` | Pricing page — static content | No | None |
| `privacy.tsx` | Privacy policy — static text | No | None |
| `terms.tsx` | Terms of use — static text | No | None |
| `catalog/[...categories].tsx` | Catalog — FilteredProductList filtered by category | No | Uses `usePathname()` for category path resolution |
| `product/[...productData].tsx` | Product detail — full product page | Yes — `router.back()` at line 210 | Cross-baseSite check at line 103; `router.push('/')` on not-found |
| `user/[...userData].tsx` | User profile — user's product list | Yes — via `UserProductsList` which has `router.back()` at line 83 | `router.push('/')` on not-found |
| `blog/free-zone.tsx` | Free zone info page | Not checked (likely static) | None |

### `app/(portal)/(secured)/` — secured portal routes

| Route file | Description | Has own header/back? | Navigation logic |
|------------|-------------|---------------------|------------------|
| `favorites.tsx` | Favorites list — `FavoritesProductList` component | No | None |
| `messages.tsx` | Chat — KAV wrapped, shows Chats or Messages based on activeChatId | No | None |
| `notifications.tsx` | Notification list — FlatList | No | `router.push()` on notification items (lines 93, 102) |

### `app/owner/dashboard/` — dashboard routes

| Route file | Description | Has own header/back? | Navigation logic |
|------------|-------------|---------------------|------------------|
| `products/index.tsx` | Dashboard product list | No | None |
| `products/[productId].tsx` | Product edit form | Yes — `router.back()` at lines 210 and 332 | None |
| `user.tsx` | User settings/profile form | No | None |
| `balance.tsx` | Balance page | No | `router.push('/pricing')` at line 25 |
| `follows.tsx` | Follows list | No | None |
| `reviews.tsx` | Reviews page | Not checked | None |
| `analytics.tsx` | Analytics page | Not checked | None |
| `account-verification.tsx` | Account verification | Not checked | None |
| `not-ready.tsx` | Placeholder page | Not checked | None |

### Other routes

| Route file | Description | Notes |
|------------|-------------|-------|
| `app/+not-found.tsx` | 404 page | Uses `<Stack.Screen options={{ title: 'Oops!' }} />` (line 7) — the ONLY `Stack` usage in the app. This `Stack.Screen` is orphaned because the parent `app/_layout.tsx` renders `<Slot>` not `<Stack>`, so the options have no effect. |
| `app/__smoke__/upload.tsx` | Image upload smoke test harness | Dev-only. No navigation. |

---

## Section 5 — F13 integration check

### Guard code shape (identical across all three layouts)

```tsx
const user = useAuthStore((s) => s.user);
const hasHydrated = useAuthStore((s) => s._hasHydrated);

if (!hasHydrated) return null;
if (!user) return <Redirect href="/" />;

return <Slot />;
```

**State read:** `user` and `_hasHydrated` from `useAuthStore` via individual selectors (not whole-store).

**Return type before redirect:** `null` while `_hasHydrated` is false (no visual output, no spinner).

**Redirect target:** `/` (the portal home).

**Specific locations:**
- `app/(portal)/(secured)/_layout.tsx` — lines 5-11
- `app/owner/_layout.tsx` — lines 16-17, 28-29 (guard), line 42 (`<Slot />` in the authenticated branch)
- `app/owner/dashboard/_layout.tsx` — lines 4-5, 8-11

### Does the guard still work with `<Stack>` / `<Tabs>` instead of `<Slot />`?

**Yes.** Verified from expo-router source code.

**Evidence:** `node_modules/expo-router/build/link/Redirect.js` (lines 31-45):

```js
function Redirect({ href, relativeToDirectory, withAnchor }) {
    const router = (0, hooks_1.useRouter)();
    const isPreview = (0, PreviewRouteContext_1.useIsPreview)();
    (0, useFocusEffect_1.useFocusEffect)(() => {
        if (!isPreview) {
            try {
                router.replace(href, { relativeToDirectory, withAnchor });
            } catch (error) {
                console.error(error);
            }
        }
    });
    return null;
}
```

The `Redirect` component:
1. **Returns `null`** — renders nothing to the tree.
2. Uses `useFocusEffect` (not `useEffect`) to call `router.replace()`.

**Why this means the guard works regardless of navigator type:**

The guard pattern is:
```tsx
if (!user) return <Redirect href="/" />;   // ← this branch
return <Stack />;                           // ← never reached
```

When the layout returns `<Redirect>`, it returns a component that renders `null`. The `<Stack>` or `<Tabs>` is in the alternative branch and **never enters the React tree**. The `<Redirect>` is rendered **instead of** the navigator, not **inside** it. Therefore:
- No child screens mount (the navigator that would mount them doesn't exist in the tree)
- `useFocusEffect` fires and calls `router.replace()`, navigating away

This is structurally identical to the current `<Slot />` behavior. The guard logic works unchanged.

**Edge case to note:** The `return null` while `!hasHydrated` is also safe — it prevents any navigator from rendering during the hydration window, same as today.

---

## Section 6 — F26 inventory

### Module-scope `Dimensions.get('window')` calls

| File | Line | Values used | Usage |
|------|------|-------------|-------|
| `src/components/FloatingButton.tsx` | 6 | `width`, `height` | Initial position, clamping bounds, and snap-to-side calculations for a draggable floating button. Used in `useSharedValue` initial values and Pan gesture `onUpdate`/`onEnd` callbacks. |
| `src/components/FullScreenImageViewer.tsx` | 8 | `width`, `height` | Passed to `ZoomableImage` component as dimensions for the full-screen image viewer modal. |
| `src/components/product/ProductList.tsx` | 23 | `height` | Used as `SCREEN_HEIGHT` — likely for scroll-to-top threshold or initial list sizing. |

### Component-body `Dimensions.get('window')` calls (not module-scope but non-reactive)

| File | Line | Values used | Usage |
|------|------|-------------|-------|
| `src/components/dashboard/layout/DashboardSidebar.tsx` | 40 | `width` | Used for sidebar slide animation — `Animated.Value(screenWidth)` initial value and slide target. Called on every render (inside component body) but not reactive to dimension changes. |

### Orientation lock status

`app.config.ts:23` declares `orientation: 'portrait'`. The app is portrait-locked. This means:
- Rotation will not change window dimensions on phones.
- However, dimensions can still change on: iPad split-screen/Slide Over, foldable Android devices (fold/unfold), and potentially during app startup before the lock takes effect.

### Assessment

All four sites use `Dimensions.get('window')` values for positioning/sizing that would produce incorrect results if dimensions changed. Because the app is portrait-locked, the primary risk is latent (foldables, split-screen). The brief's F26 scope (`useWindowDimensions()` replacement) is the correct fix regardless — it's a React hook that re-renders on dimension changes and is the RN-idiomatic approach.

The structural audit named `FullScreenImageViewer.tsx` and `FloatingButton.tsx`. This audit confirms both and adds two more: `ProductList.tsx` and `DashboardSidebar.tsx`.

---

## Section 7 — Deep-link configuration

### Scheme declaration

`app.config.ts:24` declares `scheme: 'oglasino'`. This means URLs like `oglasino://path/to/screen` will open the app.

### Deep-link handling code

**Absent.** Confirmed by grep:

- `Linking.addEventListener` — **zero calls found** (no custom deep-link event listeners)
- `Linking.getInitialURL` — **zero calls found** (no manual initial URL handling)
- Custom deep-link route config — **none** (no `linking` config object or `getInitialURL` override)

**`Linking.openURL` is used** but only for outbound links (not inbound deep-link handling):
- `src/components/pricingPage/SupportButton.tsx:12` — commented-out PayPal URL
- `src/components/product/CallUserButton.tsx:60` — `tel:` URL
- `src/components/dialog/dialogs/AppVersionConfigurationDialog.tsx:23` — external URL
- `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:110-112` — product web URL
- `src/components/internals/AppVersionConfigInit.tsx:42` — external URL

### Expo-router auto-handling

Expo-router automatically handles deep links when a `scheme` is configured — incoming URLs are matched against the file-based route tree. No additional wiring is needed for basic deep-link support.

### Implication for Φ2

When `<Stack>` and `<Tabs>` navigators replace `<Slot />`, deep links are handled identically by expo-router's linking integration. The navigators receive the resolved route and push/select the appropriate screen. No custom deep-link code needs to integrate with the new navigators because no custom deep-link code exists.

The F13 auth guards correctly intercept deep links to secured routes — expo-router routes deep links through layout trees, so the guard's `<Redirect>` fires before the secured screen mounts (confirmed in Section 5).

---

## Section 8 — Cross-cutting concerns

### Provider order

Confirmed from `app/_layout.tsx` (lines 52-78):

```
GestureHandlerRootView
  └── AppContextProvider
      └── AppVersionConfigInit
          └── ToastProvider
              └── ThemeProvider (@react-navigation/native)
                  └── SafeAreaView
                      ├── [conditional status rendering]
                      ├── AppInit (sibling)
                      ├── View (flex-1 wrapper)
                      │   └── Slot  ← navigator goes here
                      ├── PortalHost
                      └── DialogManager
```

**GestureHandlerRootView must wrap the navigator:** Yes, it does. It's the outermost wrapper (line 52). Gestures (including swipe-back on `<Stack>`) will work correctly because `GestureHandlerRootView` is above the navigator position.

**ThemeProvider from `@react-navigation/native`:** Already present (line 56). This is required by `<Stack>` and `<Tabs>` navigators for theme-aware styling. It's correctly positioned above the navigator slot.

### AppInit mount point

`AppInit` mounts at `app/_layout.tsx:64` as a sibling to the `<View>` that wraps `<Slot />`. It is inside the `(status === 'ready' || status === 'loading')` conditional branch.

**AppInit children** (`src/components/init/AppInit.tsx` lines 22-29):
- `AccountStateDialogsInit` — Φ1 shipped
- `CardSizeInit`
- `ChatsInit`
- `ForegroundRevalidationInit` — Φ1 shipped
- `PushNotificationsInit`
- `NotificationsInit`
- `InitFavoritesStore`

**Does the navigator structure change where AppInit should mount?** No. AppInit is a sibling to the navigator, not inside it. Its init components use store subscriptions and Firebase listeners — they don't render visible UI (they return fragments). They correctly belong outside the navigator tree. The new `<Stack>` or `<Tabs>` replacing `<Slot />` is still inside the same `<View className="flex-1">`, and AppInit remains a sibling.

### SafeAreaView placement

`SafeAreaView` from `react-native-safe-area-context` is at `app/_layout.tsx:57` with `className="flex-1 bg-background"`. It wraps everything including the navigator slot.

**Safe area with `<Stack headerShown: false>`:** When TopBar is inside a `<Stack>` with `headerShown: false`, the native header is not rendered, but the `<Stack>` still operates within the SafeAreaView's bounds. Since SafeAreaView is at the root layout (above all navigators), and TopBar is rendered by child layouts (`(portal)/_layout.tsx` and `owner/_layout.tsx`), the safe-area math is unaffected. TopBar does NOT use `useSafeAreaInsets()` — it relies on the root SafeAreaView wrapping.

**Components that use `useSafeAreaInsets()`:**
- `src/components/user/UserMenu.tsx:36` — for modal positioning
- `src/components/basic/Select.tsx:40` — for dropdown positioning

These are modal/overlay components that calculate their own positioning. They are unaffected by the navigator change because they use `Modal` (which renders outside the navigation tree).

### KeyboardAvoidingView inconsistency

KAV `behavior` settings vary across the codebase:

| File | iOS behavior | Android behavior |
|------|-------------|-----------------|
| `app/(portal)/(secured)/messages.tsx:12` | `'padding'` | `'padding'` |
| `src/components/dialog/DialogManager.tsx:74` | `'padding'` | `'height'` |
| `src/components/dialog/components/DialogWrapper.tsx:59` | `'padding'` | `'height'` |
| `src/components/SearchInput.tsx:161` | not checked in detail | not checked in detail |

Note: `messages.tsx:12` has `behavior={Platform.OS === 'ios' ? 'padding' : 'padding'}` — both branches produce `'padding'`, making the ternary redundant.

**Impact of navigator on KAV:** When screens are inside a `<Stack>`, the KAV's `keyboardVerticalOffset` may need adjustment because the `<Stack>` adds its own chrome (even with `headerShown: false`, there may be subtle layout differences). The messages screen's `keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0}` (line 13) is a hardcoded value that assumes specific layout heights above it — this assumption may break when the layout tree changes.

---

## Section 9 — Seams

### Seam 1: BottomBar uses `usePathname` + `pathname.startsWith` for active tab

**Today's assumption:** The current active route is determined by `usePathname()` (expo-router hook) and string comparison via `pathname.startsWith('/favorites')`, `pathname.startsWith('/notifications')`, etc. (`src/components/navigation/BottomBar.tsx:45,73,90,112,129,157,175`).

**With `<Tabs>` underneath:** A `<Tabs>` navigator provides its own "current tab" state via the `tabBar` prop's `state.index` and `state.routes` parameters. The `tabBar` render prop receives `{ state, descriptors, navigation }` which is the canonical source for which tab is active.

**Change required:** When BottomBar becomes the `tabBar` render prop of `<Tabs>`, it should read the active tab from the `state` parameter rather than from `usePathname()`. The `usePathname()` approach works but is indirect — it parses a string from the URL bar equivalent when the navigator already knows which tab is focused. Additionally, `usePathname()` may include nested route segments (e.g., `/messages/123`) that `startsWith` handles correctly today but would be more robustly expressed via the tab state index.

---

### Seam 2: BottomBar is mounted in the portal layout, not as a `tabBar` prop

**Today's assumption:** BottomBar is a regular React component rendered as a sibling to `<Slot />` in `app/(portal)/_layout.tsx:22`. It is always visible when any portal route is active.

**With `<Tabs>` underneath:** The bottom bar should be the `tabBar` prop of `<Tabs>`. This changes:
1. BottomBar receives the `BottomTabBarProps` from React Navigation (`state`, `descriptors`, `navigation`)
2. BottomBar is no longer a standalone component but a render function or component conforming to the `tabBar` interface
3. Tab press handlers should use `navigation.navigate(routeName)` or the provided `onPress` from descriptors, rather than `router.push(path)`

**Change required:** Refactor BottomBar to accept `BottomTabBarProps` and wire it as `<Tabs tabBar={(props) => <BottomBar {...props} />}>`. Navigation calls change from `router.push('/favorites')` to the tab navigation mechanism. The auth gate (`openDialogSafe`) needs to intercept the tab press before the navigation fires.

---

### Seam 3: `router.push()` in BottomBar pushes onto the navigation stack instead of switching tabs

**Today's assumption:** `router.push('/favorites')` (BottomBar line 64) pushes a new route onto the expo-router history. With bare `<Slot />`, this is a simple URL change — there is no stack, so "push" is effectively "replace."

**With `<Tabs>` underneath:** Each tab press should switch tabs (preserving tab state), not push a new route. Using `router.push('/favorites')` would push the favorites route onto the current tab's stack instead of switching to the favorites tab. The user would see favorites but the back button would return to the previous screen, and tab state preservation would break.

**Change required:** Tab navigation should use the `<Tabs>` navigator's tab switch mechanism (either via the `tabBar` prop's `navigation.navigate(routeName)` or via `router.navigate('/favorites')` — not `router.push`). The distinction is: `push` adds to the stack; `navigate` switches to the tab.

---

### Seam 4: DashboardSidebar uses `router.push()` for all dashboard navigation

**Today's assumption:** DashboardSidebar navigates via `router.push(url)` at lines 58, 142, 159. With bare `<Slot />`, this replaces the current screen content.

**With `<Stack>` underneath in the owner layout:** Each `router.push()` from the sidebar would push a new screen onto the stack. If the user navigates Products → Balance → User via the sidebar, they'd have a 3-deep stack. Pressing back would go Balance, not return to the previous portal surface.

**Change required:** Sidebar navigation within the dashboard should likely use `router.replace()` or the `<Stack>` navigator's `navigate` (which reuses existing screens) rather than `push`. The expected behavior is: sidebar taps switch between dashboard screens without building a deep back stack.

---

### Seam 5: Per-screen back buttons use `router.back()` — under `<Stack>`, this pops the stack

**Today's assumption:** Three screens call `router.back()`:
- `app/(portal)/(public)/product/[...productData].tsx:210`
- `app/owner/dashboard/products/[productId].tsx:210,332`
- `src/components/product/UserProductsList.tsx:83`

With bare `<Slot />`, `router.back()` navigates to the previous URL in expo-router's history. The behavior is browser-like.

**With `<Stack>` underneath:** `router.back()` pops the top screen from the stack, revealing the previous screen with its preserved state. This is actually **better behavior** — it's what users expect on native platforms (animated pop transition, previous screen state preserved).

**Change required:** None — the `router.back()` calls work correctly with `<Stack>`. The observable improvement is: transitions become animated, and the previous screen is preserved in memory rather than re-mounted. This is the desired outcome.

---

### Seam 6: TopBar is rendered by child layouts, not by `<Stack>` header

**Today's assumption:** TopBar is rendered explicitly by `app/(portal)/_layout.tsx:15-18` and `app/owner/_layout.tsx:37-41` as a regular component above `<Slot />`. Each layout passes different props (portal vs dashboard filter store, hide-search-bar logic).

**With `<Stack>` and `headerShown: false`:** The custom TopBar continues to render as a regular component inside the layout. The `<Stack>` navigator's native header is hidden via `headerShown: false` on each `<Stack.Screen>`. The TopBar is part of the layout's render tree, not the stack's header.

**Change required:** TopBar rendering remains in the layout, outside the `<Stack>` content area. The `<Stack>` should have `screenOptions={{ headerShown: false }}` so the native header doesn't compete with the custom TopBar. This is the approach described in the brief ("keep custom TopBar; native Stack underneath; headerShown: false").

---

### Seam 7: Portal layout renders TopBar + CategoryNavigation + BottomBar alongside `<Slot />`

**Today's assumption:** `app/(portal)/_layout.tsx` renders a flat structure:
```
<View> [TopBar, CategoryNavigation] </View>
<Slot />
<BottomBar />
```
The TopBar and CategoryNavigation are always visible for all portal routes (public and secured).

**With `<Tabs>` replacing `<Slot />` in the portal layout:** The `<Tabs>` navigator becomes the main content area. TopBar and CategoryNavigation render above the `<Tabs>`. BottomBar becomes the `tabBar` prop.

The question is: **what are the tabs?** The portal layout has two route groups: `(public)/` and `(secured)/`. But these are logical groupings, not tab destinations. The actual tab destinations from BottomBar are: Home (`/`), Favorites (`/favorites`), Notifications (`/notifications`), Messages (`/messages`), plus the center new-product button and the user menu.

**Change required:** The portal layout's `<Tabs>` should define tab screens for the BottomBar's four destinations (favorites, notifications, messages + the home/catalog surface). The `(public)` and `(secured)` route groups may need to be restructured to align with the tab structure, OR the tabs can be configured as `href`-based with grouped routes. This is a structural decision for Mastermind.

---

### Seam 8: The `(public)` and `(secured)` route groups don't map to tab destinations

**Today's assumption:** Routes are organized as `(public)` (no auth required) and `(secured)` (auth required). This grouping exists for auth-guard purposes, not for navigation structure.

**With `<Tabs>` underneath:** Tab navigators need tab screens. The four BottomBar destinations cross the `(public)`/`(secured)` boundary:
- Home (`/`) → `(public)/index.tsx`
- Favorites → `(secured)/favorites.tsx`
- Notifications → `(secured)/notifications.tsx`
- Messages → `(secured)/messages.tsx`

Additionally, many public routes (product detail, user profile, catalog, about, pricing, etc.) should appear in the "home" tab but are individual screens that would need `<Stack>` navigation within the tab.

**Change required:** This is the core structural decision for Φ2. Options include:
1. Nested navigators: `<Tabs>` at portal level with nested `<Stack>` per tab. Home tab gets a Stack with catalog/product/user/about/pricing screens. Each secured tab gets its own Stack.
2. Flat tabs with route groups: configure the `<Tabs>` with `href`-based routing that respects the current directory structure.

---

### Seam 9: The `+not-found.tsx` uses `<Stack.Screen>` with no parent `<Stack>`

**Today's assumption:** `app/+not-found.tsx` (lines 4-10) renders `<Stack.Screen options={{ title: 'Oops!' }} />`. This is the ONLY Stack usage in the entire app. The parent layout (`app/_layout.tsx`) renders `<Slot />`, not `<Stack>`, so this `<Stack.Screen>` has no parent `<Stack>` navigator and its `options` have no effect.

**With `<Stack>` at the root layout:** If the root layout's `<Slot />` is replaced with `<Stack>`, the `+not-found.tsx`'s `<Stack.Screen>` becomes functional — it will configure the screen's header title as "Oops!". Since the root layout will have `headerShown: false` globally (to preserve the custom TopBar), this specific screen's title would also be hidden unless overridden.

**Change required:** Decide whether `+not-found.tsx` should show the native header with "Oops!" title or use the custom TopBar pattern. If it should use the custom pattern, remove the `<Stack.Screen>` or keep it with `headerShown: false`.

---

### Seam 10: Owner layout mounts TopBar and DashboardSidebar alongside `<Slot />`

**Today's assumption:** `app/owner/_layout.tsx` renders TopBar (dashboard variant) and DashboardSidebar around `<Slot />`. The Slot renders `dashboard/_layout.tsx` which is another `<Slot />`.

**With `<Stack>` at the owner level:** The `<Stack>` replaces the outer `<Slot />` and manages transitions between dashboard screens. DashboardSidebar remains as a sibling overlay (it uses absolute positioning for the slide-out panel). TopBar remains above the stack content area.

**Change required:** The `<Stack>` at the owner level should have `headerShown: false` so the custom TopBar is used. DashboardSidebar's navigation calls (currently `router.push()`) should change to `router.navigate()` or `router.replace()` to avoid stacking (see Seam 4). The nested `app/owner/dashboard/_layout.tsx` may become unnecessary if the owner Stack directly manages dashboard screens — or it could remain as a group layout with its own auth guard.

---

### Seam 11: `KeyboardAvoidingView` offset assumes specific layout heights

**Today's assumption:** `messages.tsx:13` sets `keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0}`. This `90` assumes a specific number of pixels above the messages screen (TopBar + CategoryNavigation + bottom bar insets). The value is hardcoded.

**With `<Stack>` or `<Tabs>` underneath:** The navigator may add or remove layout elements (e.g., a hidden native header still occupies zero space, but the navigator's wrapper View may have different dimensions). The `90` might need adjustment.

**Change required:** Measure the actual offset empirically after the navigator swap, or compute it dynamically using `useHeaderHeight()` from `@react-navigation/elements` (if the native header is visible) or layout measurements.

---

### Seam 12: Messages screen uses `activeChatId` state to toggle between Chats list and Messages view

**Today's assumption:** `app/(portal)/(secured)/messages.tsx:7-8` reads `activeChatId` from `useChatStore` and conditionally renders either `<Chats />` or `<Messages />`. This is a manual "navigation" within a single route — no actual route change occurs when opening a chat.

**With `<Stack>` underneath:** The chat list → chat detail transition could be modeled as a Stack push (list screen → detail screen). This would give native transitions, swipe-back to return to the list, and proper lifecycle management. However, the current implementation uses store state (`activeChatId`) to toggle, and the brief's scope may not include restructuring this.

**Change required:** Decision needed: keep the current state-based toggle (no change to messages.tsx, but no native transitions for chat open/close) or refactor to two routes (messages list + messages detail) with Stack navigation. This is a scope decision for Mastermind.

---

### Seam 13: UserMenu navigates to dashboard paths with `/dashboard/` prefix instead of `/owner/dashboard/`

**Today's assumption:** `src/components/user/UserMenu.tsx` navigates to paths like `/dashboard/balance` (line 110) and `/dashboard/account-verification` (line 174). The correct paths based on the directory structure are `/owner/dashboard/balance` and `/owner/dashboard/account-verification`.

**With `<Stack>` or `<Tabs>`:** Incorrect paths will fail to resolve and may trigger the `+not-found.tsx` screen or produce navigation errors. Today with bare `<Slot />`, the incorrect paths may silently fail or produce the same broken behavior — this is a pre-existing bug, not a seam introduced by the navigator change.

**Change required:** Fix the broken paths in UserMenu.tsx. This is a pre-existing bug (already noted in the structural audit as "UserMenu broken paths" — part of the Ω scope per `features/expo-structural-foundation.md` section 5). The navigator change makes it more visible but doesn't cause it.

---

### Seam 14: DashboardSidebar uses non-reactive `Dimensions.get('window').width` for animation

**Today's assumption:** `src/components/dashboard/layout/DashboardSidebar.tsx:40` calls `Dimensions.get('window').width` inside the component body (not module scope, but on every render). The value initializes `Animated.Value(screenWidth)` at line 41 for the slide animation.

**With `<Stack>` underneath:** No direct interaction. But the `Animated.Value` is created via `useRef` (line 41), so it's only initialized once. If dimensions change (foldable device), the animation target is stale.

**Change required:** Replace with `useWindowDimensions()` and update the animation target when width changes. This overlaps with F26 scope.

---

## Part 4b — Adjacent observations

### Observation 1: `console.error` in product screen

`app/(portal)/(public)/product/[...productData].tsx:119` has `console.error('Failed to load product', error)`. This is an ad-hoc debug log that should use the existing logging utility (if one exists) or be removed per conventions Part 4.

**Severity:** low
**Not fixed:** out of scope (read-only audit).

### Observation 2: Redundant KAV ternary in messages.tsx

`app/(portal)/(secured)/messages.tsx:12` — `behavior={Platform.OS === 'ios' ? 'padding' : 'padding'}`. Both branches are `'padding'`, making the ternary redundant.

**Severity:** low
**Not fixed:** out of scope.

### Observation 3: `allowPreferenceCookies` field on mobile user settings

`app/owner/dashboard/user.tsx:46` — the user settings screen reads and submits `allowPreferenceCookies`. Per the Consent Mode v2 decisions entry (2026-05-22), this field was removed from the backend (`allowPreferenceCookies` dropped from `AuthUserDTO`, `UpdateUserDTO`, `LoginRequest`, `User` entity, `users` table). The mobile screen is still sending it — likely a no-op (backend ignores unknown fields) but dead code.

**Severity:** low (dead code)
**Not fixed:** out of scope. Likely addressed by Chat C (mobile user-settings cleanup).

### Observation 4: Hardcoded Serbian text in user settings

`app/owner/dashboard/user.tsx:230` — `<Text>DODAJ ZA BASE SITE</Text>` — literal Serbian placeholder text, not translated.

**Severity:** low
**Not fixed:** out of scope.

### Observation 5: `setTimeout(() => setPreparing(false), 200)` in user page

`app/(portal)/(public)/user/[...userData].tsx:31` — artificial 200ms delay before showing user content. Likely a loading-state workaround.

**Severity:** low
**Not fixed:** out of scope.
