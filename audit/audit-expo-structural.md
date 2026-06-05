# Structural audit — oglasino-expo

**Date:** 2026-05-24
**Branch:** dev
**Scope:** structural wrongness across the entire mobile app. Read-only.

---

## Executive summary

The mobile app functions but is structurally wrong for the platform in foundational ways. The dominant category of wrongness is **lifecycle gaps** — the app has no auth re-validation on cold start or foreground resume, no forced logout on ban/deletion, no network state detection, and `logoutFirebase` doesn't call `auth.signOut()`. The second dominant category is **performance shape** — zero `React.memo` usage across the entire codebase, `expo-image` installed but never imported (all images use RN `Image` with no disk caching), whole-store Zustand subscriptions causing re-render storms, and the AppContext value is not memoized. The third is **RN convention violations** — every layout uses bare `<Slot />` with no native Stack or Tab navigators, the message list is not inverted, and there are `<div>` elements in RN code that will crash on native.

The most consequential findings are: **F1** (auth listener never called — stale user persists indefinitely), **F2** (logout doesn't sign out of Firebase — session leak), **F5** (no 401/banned-user interceptor), **F8** (zero React.memo — every list re-renders fully), **F9** (expo-image unused — no disk caching on product feed), and **F12** (all layouts use Slot — no native navigation).

---

## Architectural overview

**Navigation shape.** Expo Router with file-based routing. Three route groups: `(portal)/(public)` for unauthenticated screens, `(portal)/(secured)` for favorites/messages/notifications, and `owner/dashboard` for the owner dashboard. Every layout file uses bare `<Slot />` — no `Stack`, `Tabs`, or `Drawer` navigator is used anywhere. The bottom bar and top bar are custom `View` components positioned manually, not native tab/header primitives. There are no native transitions, no swipe-back gesture on iOS, and tab switching remounts screens rather than preserving them.

**State management.** Zustand stores, 15 total across two directories (`src/lib/store/` and `src/lib/stores/`). `authStore` uses Zustand `persist` with AsyncStorage; `useCardSizeStore` uses manual AsyncStorage hydration; all others are in-memory only. A React Context (`AppContext`) manages bootstrap state (base site, language, maintenance status, configuration) with a 5-second polling interval. There is no centralized logout cleanup — individual stores rely on component effects to react to `user` becoming null.

**Service layer.** Flat directory (`src/lib/services/`) with 17 service files. Axios-based with a single global instance (`BACKEND_API`). Request interceptor attaches Firebase ID token via `auth.currentUser.getIdToken()`. No `nextCalls/reactCalls` split (correct for mobile). Every service swallows backend errors and returns sentinel values (`null`, `false`, `[]`), discarding the structured error response. None implement the Part 7 error contract.

**Storage.** AsyncStorage for auth persistence and card-size preference. `expo-secure-store` is in `package.json` and `app.config.ts` plugins but actual usage is limited to the secure store config plugin registration — not used for sensitive data storage. Firebase JS SDK (not native `@react-native-firebase`) for Firestore listeners and auth.

**Provider stack.** Root layout wraps: `GestureHandlerRootView` → `AppContextProvider` → `AppVersionConfigInit` → `ToastProvider` → `ThemeProvider` → `SafeAreaView`. `AppInit` (mounted when status is `ready` or `loading`) initializes card size, chats, push notifications, notification store, and favorites.

**Lifecycle handling.** `AppState` is used in exactly two places: app version check on foreground resume, and push permission revocation check. There is no auth re-validation, no Firestore listener health check, no network state detection, and no stale-data refresh on foreground resume. The `initAuthListener` method exists in the auth store but is never called from any initialization code.

---

## Findings

---

### Finding 1 — `initAuthListener` is defined but never called; no auto-login on cold start

**Category:** lifecycle
**Severity:** high
**Surface:** auth / app initialization
**Files:**
- `src/lib/store/authStore.ts:150-162` (defined, zero call sites)
- `src/components/init/AppInit.tsx` (does not call it)
- `app/_layout.tsx` (does not call it)

**What's there.** The auth store defines an `initAuthListener` method that calls `listenAuthState` (which wraps Firebase's `onAuthStateChanged`). The listener syncs the Firebase user to the backend via `syncUserToBackend` and updates the store. However, a codebase-wide search reveals zero call sites — the method is never invoked from any initialization component, layout, or effect.

**Why it's structurally wrong.** Without an active auth listener, the app relies entirely on Zustand's AsyncStorage persistence for restoring the user session on cold start. If the user's Firebase session is invalidated server-side (password change, account deletion, admin ban), the app will never know — it will keep showing the stale persisted `AuthUserDTO` from the last session. The only path that currently calls `syncUserToBackend` is the explicit `login`/`register`/`loginWithGoogle` actions, not cold-start restoration.

**Observable consequences today.** A user who was banned or deleted between sessions will see the app as if they are still logged in. API calls will fail individually with 401/403, but there is no global handler (see Finding 5), so the user sees scattered errors rather than a clean logout.

**What the right shape would look like.** `initAuthListener` called once from `AppInit` or the root layout, with a single-flight guard (stored unsubscribe ref), gated on `_hasHydrated` to avoid racing Zustand's persistence rehydration. The listener handles `syncUserToBackend` failures (e.g., USER_BANNED response) by forcing logout.

**Maps to existing future chat?** E (mobile user-deletion adoption) — the auth listener is a prerequisite for the 401/403 interceptor and ban-notice dialog.

---

### Finding 2 — `logoutFirebase` does not call `auth.signOut()`; Firebase session persists after logout

**Category:** lifecycle
**Severity:** high
**Surface:** auth / logout
**Files:**
- `src/lib/services/authService.ts:233-235`
- `src/lib/store/authStore.ts:121-145`
- `src/lib/config/api.ts:22-28`

**What's there.** `logoutFirebase` only calls `GoogleSignin.signOut()`. It never calls Firebase's `auth.signOut()`. The auth store's `logout` method calls `logoutFirebase`, then sets `user: null` in Zustand and clears `viewTokenStore`.

**Why it's structurally wrong.** Firebase's `onAuthStateChanged` will never fire with `null` on logout, so any listener-driven cleanup (favorites, notifications) won't trigger. `auth.currentUser` remains populated — the axios request interceptor at `api.ts:27-28` reads `auth.currentUser` directly and will continue attaching a valid Bearer token to every API request after the user has "logged out" in the UI. On the next cold start, Firebase persistence (via `getReactNativePersistence(AsyncStorage)`) will restore the session.

**Observable consequences today.** After logout: (1) API calls continue to work with the old user's credentials, (2) if a different user logs in on the same device, there is a window where both Firebase sessions overlap, (3) the `InitFavoritesStore` component (which listens to Firebase auth state) never receives a null, so favorites are never cleared via the Firebase-driven path.

**What the right shape would look like.** `logoutFirebase` calls `auth.signOut()` first (Firebase), then `GoogleSignin.signOut()` (Google SSO). The auth store's `logout` directly clears all user-scoped stores (chat, favorites, notifications, filters) rather than relying on component effects.

**Maps to existing future chat?** E (mobile user-deletion adoption) — B1 in Bucket 1 of expo-release-readiness.md.

---

### Finding 3 — Store cleanup on logout is fragile; relies on component mount state, not direct store clearing

**Category:** state-shape
**Severity:** high
**Surface:** all user-scoped stores
**Files:**
- `src/lib/store/authStore.ts:121-145` (logout — only clears `user` and `viewTokenStore`)
- `src/components/init/ChatsInit.tsx:17-19` (unsubscribes but doesn't clear chat data)
- `src/components/init/InitFavoritesStore.ts:11-13` (listens to Firebase auth, not Zustand)
- `src/notifications/components/NotificationsInit.tsx:11` (resets on user null)
- `src/lib/store/useFilterStore.ts` (never cleared)
- `src/lib/store/useCatalogStore.ts` (never cleared, once-only init)
- `src/lib/store/userPreferenceStorage.ts` (never cleared)

**What's there.** `authStore.logout()` directly clears only two things: `user: null` and `useViewTokenStore.getState().clear()`. Every other store's cleanup depends on React components being mounted and reacting to the user becoming null. Filter stores, catalog store, and user preference storage are never cleared on logout at all.

**Why it's structurally wrong.** If a cleanup component is unmounted (user navigated away), the store keeps stale data. Chat stores' `unsubscribeAll()` tears down Firestore listeners but does not call `clearChatStore()` — the `chats`, `messages`, and `userCache` arrays remain populated with the previous user's data until a new user logs in. Filter stores carry user A's filters into user B's session. The catalog store's once-only `initialized` flag means `setCatalog` can never update after the first call — stale catalog data persists across user sessions until app restart.

**Observable consequences today.** If user A sets filters, logs out, user B logs in, user B sees user A's filters. Chat data from user A can briefly flash when user B logs in. The catalog store can't recover from stale data without an app kill.

**What the right shape would look like.** `authStore.logout()` directly calls `clearChatStore()`, `useFavoritesStore.getState().clear()`, `useNotificationStore.getState().reset()`, `usePortalFilterStore.getState().clearAllFilters()`, `useDashboardFilterStore.getState().clearAllFilters()`. Component-driven cleanup remains as a secondary safeguard but is not the primary mechanism.

**Maps to existing future chat?** E (mobile user-deletion adoption) — the logout cleanup is a prerequisite for the deletion flow.

---

### Finding 4 — No auth re-validation on app foreground; stale tokens and stale user state after backgrounding

**Category:** lifecycle
**Severity:** high
**Surface:** auth / app lifecycle
**Files:**
- `src/components/context/AppContext.tsx:140-165` (maintenance poll exists, no auth check)
- `src/lib/store/authStore.ts` (no AppState listener)
- `src/lib/config/api.ts` (no 401 response handling)

**What's there.** `AppState` is used in two places: `AppVersionConfigInit.tsx` (version check on foreground) and `pushNotificationRegister.ts` (permission revocation check on foreground). There is no AppState listener that re-validates auth on foreground resume.

**Why it's structurally wrong.** After the app has been backgrounded for hours or days: Firebase ID tokens expire after 1 hour, the user may have been banned or deleted, and `auth.currentUser` may be stale. The axios interceptor calls `getIdToken()` which auto-refreshes expired tokens, but there is no mechanism to detect that the user's account state has changed server-side. The 5-second maintenance poll exists but checks server health, not auth state.

**Observable consequences today.** Latent — Firebase's `getIdToken()` handles token refresh transparently in most cases. But if the user was banned while backgrounded, they will continue to use the app until an individual API call happens to fail, with no clean forced-logout experience.

**What the right shape would look like.** An AppState listener that calls `syncUserToBackend` on foreground resume after extended backgrounding (e.g., >5 minutes), with the response checked for ban/deletion status.

**Maps to existing future chat?** E (mobile user-deletion adoption) and I (backend-calls optimization — the single-flight guard on syncUserToBackend).

---

### Finding 5 — No global 401/403 response interceptor; no forced logout on banned/disabled/deleted users

**Category:** lifecycle
**Severity:** high
**Surface:** auth / API layer
**Files:**
- `src/lib/config/api.ts:33-62` (response interceptor handles ERR_NETWORK, 404 only)
- `src/lib/store/authStore.ts:150-162` (initAuthListener silently catches errors)

**What's there.** The axios response interceptor handles `ERR_NETWORK`, `ECONNABORTED`, missing response, and 404. It does not handle 401 or 403. The only 401+TOKEN_EXPIRED retry logic exists in `uploadImages.ts:282-298`, scoped to the image upload pipeline.

**Why it's structurally wrong.** If the backend returns 401 (token expired/invalid) or 403 (USER_BANNED), there is no global mechanism to force-logout or show a ban notice. The user remains "authenticated" in the Zustand store with stale data. Individual screens may show error states, but there is no coherent experience.

**Observable consequences today.** If a user is banned by an admin while using the app, API calls will start returning 403 but the user will see individual error messages (or swallowed errors returning sentinel values) rather than being signed out and shown a ban notice.

**What the right shape would look like.** A response interceptor that catches 401 (force token refresh, then retry; on second failure, sign out) and 403 with `USER_BANNED` code (sign out and show ban-notice dialog). Web's `UseTokenRefresh` equivalent.

**Maps to existing future chat?** E (mobile user-deletion adoption) — the 401/403 interceptor is explicitly in scope.

---

### Finding 6 — Maintenance poll treats offline as maintenance; no network state detection

**Category:** lifecycle
**Severity:** medium
**Surface:** app initialization / network handling
**Files:**
- `src/components/context/AppContext.tsx:143-162` (5-second poll, catch sets maintenance)
- `package.json` (no NetInfo dependency)

**What's there.** The app polls for maintenance status every 5 seconds. The catch block at line 157-160 sets `status: 'maintenance'` on any error. There is no `@react-native-community/netinfo` or equivalent anywhere in the codebase or `package.json`.

**Why it's structurally wrong.** When the device loses network connectivity, every poll throws, and the app enters the maintenance screen. The user sees "maintenance" when the real problem is "offline." There is no offline indicator, no request queuing, no retry-on-reconnect. The 5-second poll is also wasteful — 720 calls/hour regardless of app state.

**Observable consequences today.** Turning on airplane mode or entering a dead zone shows the maintenance screen instead of an offline indicator. The user may think the service is down.

**What the right shape would look like.** Install NetInfo, distinguish network errors from server-side maintenance responses. Show an offline banner instead of the maintenance screen when the device has no connectivity. Reduce maintenance poll frequency (already tracked as chat I).

**Maps to existing future chat?** I (mobile backend-calls optimization) — the 5-second poll is explicitly in scope. The NetInfo gap is new — not covered by I's scope.

---

### Finding 7 — Whole-store Zustand subscriptions causing re-render storms across the app

**Category:** performance
**Severity:** high
**Surface:** all screens
**Files:**
- `src/components/messages/Messages.tsx:27-39` (destructures 10 fields from `useChatStore()`)
- `src/components/navigation/BottomBar.tsx:26-29` (destructures from auth and chat stores)
- `src/components/user/UserMenu.tsx:32`
- `src/components/dialog/dialogs/LoginDialog.tsx:30`
- `src/components/dialog/dialogs/LoginOptionsDialog.tsx:23`
- `src/components/navigation/Filters.tsx:48`
- 22+ additional components subscribing to `useAuthStore()` without selectors

**What's there.** Components call `useAuthStore()`, `useChatStore()`, `useFavoritesStore()` without selectors, destructuring the entire store. In Zustand v5, calling `useStore()` without a selector subscribes to the entire store and uses `Object.is` comparison. No `useShallow` from `zustand/shallow` is used anywhere in the codebase.

**Why it's structurally wrong.** Every `loading: true/false` toggle in the auth store re-renders all 22+ subscribed components. Every chat list update, every message arrival, every `userCache` addition re-renders the entire Messages component (which contains a FlatList with potentially hundreds of messages). On a mid-range Android device, this causes perceptible jank.

**Observable consequences today.** Logging in or out causes a cascade of unnecessary re-renders across the bottom bar, user menu, consumer protection banner, report buttons, follow buttons, and every other auth-subscribed component. Chat screens re-render on unrelated chat-list changes.

**What the right shape would look like.** Atomic selectors: `useAuthStore(s => s.user)`, `useChatStore(s => s.messages)`. For multiple fields: `useAuthStore(useShallow(s => ({ user: s.user, loading: s.loading })))`. Actions accessed via `useAuthStore.getState().login` outside render.

**Maps to existing future chat?** No — needs its own chat or folds into a cross-cutting performance pass.

---

### Finding 8 — Zero `React.memo` usage; all list items re-render on every parent render

**Category:** performance
**Severity:** high
**Surface:** product feed, chat list, message list, review list
**Files:**
- `src/components/product/ProductCard.tsx` (not memoized, rendered per product in infinite scroll)
- `src/components/product/ProductList.tsx:222` (inline `renderItem` arrow function)
- `src/components/messages/Messages.tsx:137` (inline `renderMessageGroup`)
- `src/components/messages/Chats.tsx:51` (inline `renderItem`)
- `src/components/ReviewsList.tsx:65` (inline `renderItem`)
- `src/components/product/HorizontalExtraProductsListView.tsx:71`

**What's there.** No component in the entire codebase uses `React.memo`. Every `FlatList` uses inline arrow functions for `renderItem`, creating new function references on every render. `ProductCard.tsx` subscribes to `useCardSizeStore()` and `usePortalScope()` on every card instance — if either store changes, every visible ProductCard re-renders.

**Why it's structurally wrong.** Without `React.memo`, FlatList cannot skip re-rendering items that haven't changed. Combined with inline `renderItem` functions (new reference every render), the FlatList's built-in optimization of comparing `renderItem` identity is defeated. The product feed — the app's primary surface — re-renders every visible card on every state change to any subscribed store.

**Observable consequences today.** Scrolling jank on the product feed, especially on mid-range Android devices. Each card re-renders on every filter change, card-size change, portal-scope change, or parent state change.

**What the right shape would look like.** `React.memo` on `ProductCard`, `ExtraProductCard`, `ProductReview`, `Message`, and chat-list items. Extract `renderItem` to stable `useCallback` references. Pass `cardSize` as a prop from the parent `ProductList` instead of having each card subscribe to the store independently.

**Maps to existing future chat?** No — needs its own chat or a cross-cutting performance pass.

---

### Finding 9 — `expo-image` installed but never imported; all images use RN `Image` with no disk caching

**Category:** performance
**Severity:** high
**Surface:** product feed, product detail, chat, user avatars
**Files:**
- `package.json:44` (`"expo-image": "~3.0.11"`)
- `src/components/product/ProductTopImage.tsx:43` (RN `Image` for product cards in feed)
- `src/components/ImagesCarousel.tsx:132` (product detail carousel)
- `src/components/user/OglasinoAvatar.tsx:59` (avatars in chats, reviews)
- `src/components/messages/MessageImages.tsx:59` (chat images)
- 14+ additional files using RN `Image`

**What's there.** `expo-image` is listed in `package.json` as a dependency but is never imported anywhere in the codebase. Every `<Image>` component comes from `react-native`. The CDN variant system (`src/lib/images/variants.ts`) properly applies size variants (card: 400x300, hero: 1600x1200), which mitigates full-resolution loading.

**Why it's structurally wrong.** RN `Image` has no disk caching on iOS (only in-memory cache), no progressive loading, no placeholder/transition support, and decodes images on the JS thread. `expo-image` (backed by SDWebImage on iOS, Glide on Android) provides disk caching, hardware-accelerated decoding, blurhash placeholders, and transition animations. The impact is highest on `ProductTopImage` which renders inside the infinite-scroll product feed — scrolling back to previously viewed products re-fetches images from the network after memory cache eviction.

**Observable consequences today.** Scrolling back up in the product feed causes images to reload from the network. On slow connections, images flash/pop in. On cellular data, unnecessary bandwidth consumption.

**What the right shape would look like.** Replace all `<Image source={{ uri }}` with `<Image source={{ uri }}` from `expo-image`. Add `placeholder` prop with blurhash for hero images. Add `transition` for smooth fade-in. Decision on `expo-image` adoption is already tracked as P3 in expo-release-readiness.md.

**Maps to existing future chat?** H (mobile image-pipeline polish) — P3 decision.

---

### Finding 10 — AppContext value not memoized; all consumers re-render on any state change

**Category:** performance
**Severity:** medium
**Surface:** entire app (AppContext is consumed by many components)
**Files:**
- `src/components/context/AppContext.tsx:229-238` (value prop creates new object every render)
- `src/components/context/AppContext.tsx:169,196,225` (un-memoized action functions)

**What's there.** The `AppContext.Provider` value creates a new object on every render, spreading `state` and including `setBaseSiteForCode`, `setLanguageForCode`, `getConfiguration`, and `reBootstrap` — none of which are wrapped in `useCallback` or `useMemo`. The 5-second maintenance poll can trigger re-renders that cascade to all `useAppContext()` consumers.

**Why it's structurally wrong.** Every state change in `AppContextProvider` causes all consumers of `useAppContext()` to re-render, even if they only read `selectedLanguage` or `selectedBaseSite`. The maintenance poll (when it causes a state transition) propagates to every consumer.

**Observable consequences today.** Mostly latent — the maintenance poll short-circuits on no-change, and base-site/language changes are infrequent. But any future addition of more frequent state updates to AppContext will cascade widely.

**What the right shape would look like.** Wrap action functions in `useCallback`. Wrap the provider value in `useMemo` keyed on the relevant state fields. Alternatively, split AppContext into separate contexts for rarely-changing data (baseSite, language) and frequently-changing data (status, configuration).

**Maps to existing future chat?** No — needs its own chat or a cross-cutting performance pass.

---

### Finding 11 — Message FlatList uses index-based `keyExtractor` and is not inverted

**Category:** RN-convention
**Severity:** high
**Surface:** messaging
**Files:**
- `src/components/messages/Messages.tsx:203-213`
- `src/components/messages/Messages.tsx:163-168` (`handleContentSizeChange` bug)

**What's there.** The message FlatList uses `keyExtractor={(_, i) => i.toString()}` (index-based keys). The list is not inverted — instead, it uses manual `scrollToEnd` calls. `handleContentSizeChange` has a bug: lines 164-167 call `scrollToEnd` conditionally, then line 167 calls `scrollToEnd` unconditionally (the second call executes regardless of `firstLoadRef` or `isNearBottom`).

**Why it's structurally wrong.** Index-based keys cause React to remount all items when the array shifts — e.g., when loading older messages prepends items, every existing item gets a new index/key. This causes a full re-render of the visible list. The non-inverted pattern with manual `scrollToEnd` is unreliable — `scrollToEnd` uses estimated content height and can miss the actual bottom with variable-height items (images, multi-line text). The standard RN chat pattern is `inverted={true}` FlatList with messages in reverse-chronological order.

**Observable consequences today.** Loading older messages causes the entire visible list to re-render and potentially jump/flash. The unconditional `scrollToEnd` at line 167 causes the list to auto-scroll to the bottom on every content size change, even when the user has scrolled up to read older messages.

**What the right shape would look like.** `inverted={true}` FlatList with messages in newest-first order. `keyExtractor` using message ID (e.g., `item.messages[0].id` or a group ID). Remove all manual `scrollToEnd` logic — inverted lists automatically show the newest content at the visual bottom.

**Maps to existing future chat?** B (mobile messaging adoption).

---

### Finding 12 — All layouts use bare `<Slot />`; no native Stack or Tab navigators

**Category:** RN-convention
**Severity:** high
**Surface:** navigation / entire app
**Files:**
- `app/_layout.tsx:66`
- `app/(portal)/_layout.tsx:21`
- `app/(portal)/(public)/_layout.tsx:14`
- `app/(portal)/(secured)/_layout.tsx:4`
- `app/owner/_layout.tsx:35`
- `app/owner/dashboard/_layout.tsx`

**What's there.** Every layout file uses bare `<Slot />` from expo-router. The bottom bar (`BottomBar.tsx`) is a custom `View` component with manual navigation. The top bar (`TopBar.tsx`) is a custom component, not a native header.

**Why it's structurally wrong.** `<Slot />` provides no native transitions (slide-in/out), no native header management (no back button, no title bar, no iOS swipe-back gesture), and no tab state preservation. Tab switching remounts the screen rather than preserving it. There is no system-level tab reselect-scroll-to-top behavior. The app feels like a web SPA running in a native shell rather than a native app.

**Observable consequences today.** Instant-cut transitions between screens (no animation), no iOS swipe-back gesture, no native header, tab screens remount on every switch (losing scroll position, re-fetching data).

**What the right shape would look like.** `<Stack>` for screen-to-screen navigation with native transitions. `<Tabs>` for the bottom bar with native tab persistence. `<Stack.Screen options={{ headerShown: false }}>` where custom headers are needed. The expo-router docs explicitly recommend using `Stack` and `Tabs` over bare `Slot` for native UX.

**Maps to existing future chat?** No — needs its own chat. This is a foundational navigation rearchitecture.

---

### Finding 13 — No auth guard on `(secured)` routes; unauthenticated users can reach protected screens

**Category:** RN-convention
**Severity:** high
**Surface:** favorites, messages, notifications, owner dashboard
**Files:**
- `app/(portal)/(secured)/_layout.tsx:1-5` (empty layout, just `<Slot />`)
- `app/owner/_layout.tsx` (no auth check)
- `app/owner/dashboard/_layout.tsx` (no auth check)
- `src/components/navigation/BottomBar.tsx:31-40` (UI-level gate only)
- `src/components/dashboard/layout/DashboardSidebar.tsx:188` (accesses `user.profileImageKey` without null check)

**What's there.** The `(secured)` layout is empty — no `useAuthStore`, no `Redirect`, no guard. The only protection is that `BottomBar.tsx` shows a login dialog when unauthenticated users tap the icons. The owner dashboard layout also has no auth check and directly accesses `user.profileImageKey` without null checking.

**Why it's structurally wrong.** The UI-level gate in BottomBar is not a route-level guard. If deep links are ever enabled (the app has `scheme: 'oglasino'` in `app.config.ts`), a deep link to `/favorites`, `/messages`, `/notifications`, or `/owner/dashboard/*` bypasses the bottom bar and renders the screen with `user: null`, causing crashes or data leaks. Even without deep links, expo-router's `<Link>` or programmatic `router.push` from anywhere in the app can reach these routes without auth.

**Observable consequences today.** Latent — the BottomBar gate prevents normal navigation. But `DashboardSidebar.tsx:188` would crash if an unauthenticated user reached the owner layout (accessing `.profileImageKey` on null `user`).

**What the right shape would look like.** The `(secured)/_layout.tsx` checks `useAuthStore(s => s.user)` and `_hasHydrated`. If not authenticated after hydration, redirect to home or show login. Same for `owner/_layout.tsx`. This is the standard expo-router auth pattern using `<Redirect href="/" />`.

**Maps to existing future chat?** E (mobile user-deletion adoption) — the auth plumbing work.

---

### Finding 14 — Existing-chat send is non-atomic; three sequential writes instead of a batch

**Category:** state-shape
**Severity:** medium
**Surface:** messaging
**Files:**
- `src/lib/store/useChatStore.ts:504-556` (existing-chat send path)
- `src/lib/store/useChatStore.ts:456-502` (new-chat send path — correctly uses `writeBatch`)

**What's there.** For new chats, the send uses `writeBatch()` — correct and atomic. For existing chats, the send does three sequential writes: (1) `addDoc` to create the message, (2) `getDoc` + `setDoc` to update sender's `userchats`, (3) `setDoc` with `merge: true` to update receiver's `userchats`.

**Why it's structurally wrong.** If the app crashes, loses connectivity, or the user backgrounds the app after step 1 but before step 3, the message exists in Firestore but the receiver's unread count and preview are stale. Web adopted atomic batch writes for existing-chat sends; mobile has not.

**Observable consequences today.** Latent — the failure window is small (milliseconds between writes). But on flaky mobile connections, the probability of a partial write is higher than on web.

**What the right shape would look like.** `writeBatch` for the existing-chat path, matching the new-chat path and web's implementation.

**Maps to existing future chat?** B (mobile messaging adoption) — atomic batch is explicitly in scope.

---

### Finding 15 — Firestore `onSnapshot` listeners accumulate per-chat; never cleaned up until logout

**Category:** lifecycle
**Severity:** medium
**Surface:** messaging
**Files:**
- `src/lib/store/useChatStore.ts:228-297` (`subscribeToMessages`)
- `src/lib/store/useChatStore.ts:699-731` (`getActiveChat` calls `subscribeToMessages`)
- `src/components/messages/Messages.tsx:197` (`setActiveChatId(null)` on back — does not unsubscribe)

**What's there.** When a user opens a chat, `getActiveChat` calls `subscribeToMessages(chatId)`, creating an `onSnapshot` listener stored in `unsubMessages[chatId]`. When the user navigates back (`setActiveChatId(null)`), nothing calls `unsubscribeFromMessages(chatId)`. The listener remains active. The guard at line 231 (`if (get().unsubMessages[chatId]) return`) prevents double-subscription to the same chat, but listeners from all previously visited chats accumulate across the session. The only cleanup is `unsubscribeAll()` on user change or unmount.

**Why it's structurally wrong.** A user who browses 20 chats in a session has 20 active `onSnapshot` listeners running simultaneously, each receiving real-time updates for chats the user is no longer viewing. Each update triggers Zustand state mutations and potential re-renders.

**Observable consequences today.** Increased memory usage and Firestore read costs proportional to the number of chats visited in a session. On a long session with active conversations, this could cause perceptible performance degradation.

**What the right shape would look like.** Unsubscribe from messages when `setActiveChatId(null)` is called (navigating away from a chat). Keep only the currently active chat's listener alive.

**Maps to existing future chat?** B (mobile messaging adoption).

---

### Finding 16 — `loadMoreMessages` leaves `loadingMore` stuck on `true` after early return

**Category:** state-shape
**Severity:** medium
**Surface:** messaging
**Files:**
- `src/lib/store/useChatStore.ts:312-316`
- `src/components/messages/Messages.tsx:125` (ActivityIndicator gated on `loadingMore`)

**What's there.** `loadMoreMessages` sets `set({ loadingMore: true })` at line 313, then checks guards at line 316 (`if (!user || !chatId || hasMoreMessages[chatId] === false) return`). The early return does not reset `loadingMore` to `false`. Additionally, `loadingMore` is a single global boolean, not per-chat.

**Why it's structurally wrong.** If the early-return guard fires (no user, no chatId, or no more messages), `loadingMore` is permanently `true`, showing an `ActivityIndicator` instead of the "Load older" button for all chats until the app is killed.

**Observable consequences today.** After scrolling to the top of a chat that has no more messages, the "Load older" button is replaced by a permanent spinner.

**What the right shape would look like.** Set `loadingMore: false` on early returns. Make `loadingMore` per-chat (`loadingMore: Record<string, boolean>`) to match the per-chat nature of `hasMoreMessages`.

**Maps to existing future chat?** B (mobile messaging adoption).

---

### Finding 17 — `getActiveChat` fallback reads wrong Firestore collection shape

**Category:** state-shape
**Severity:** medium
**Surface:** messaging
**Files:**
- `src/lib/store/useChatStore.ts:711-720`
- `src/lib/store/useChatStore.ts:459-473` (shows correct field locations)

**What's there.** When the chat is not found locally, `getActiveChat` reads from `chats/{chatId}` and maps the result as if it has `withUserFirebaseUid` and `withUser` fields. But the `chats` collection document contains `{ users: [...], createdAt, lastMessage, lastUpdated }` — it does NOT have `withUserFirebaseUid` or `withUser`. Those fields exist on `userchats/{uid}/chats/{chatId}` documents.

**Why it's structurally wrong.** The Firestore read produces a `ChatSummary` with `withUserFirebaseUid: undefined` and `withUser: undefined`. The Messages component at line 189 accesses `activeChat.withUser.profileImageKey`, which would crash with a `TypeError: Cannot read properties of undefined`.

**Observable consequences today.** Any chat outside the initial 15-chat cache that is opened for the first time (via push notification or new-chat flow) will crash the Messages screen. Web fixed this bug; mobile has not adopted the fix.

**Maps to existing future chat?** B (mobile messaging adoption) — B11 in Bucket 1.

---

### Finding 18 — Error contract not implemented; all backend validation errors silently discarded

**Category:** web-seam
**Severity:** high
**Surface:** entire service layer
**Files:**
- All 17 files under `src/lib/services/`
- `src/lib/config/api.ts:33-62` (response interceptor does not parse error contract)

**What's there.** The project's error contract is `{errors: [{field, code, translationKey}]}` (Part 7). Every service catches errors, logs them via `logServiceError`, and returns a sentinel value (`null`, `false`, `[]`, `0`). The structured error response from the backend is silently discarded. No service parses `field`, `code`, or `translationKey` from the response.

**Why it's structurally wrong.** The mobile app cannot display field-specific validation errors, cannot use `translationKey` for localized error messages, and cannot distinguish between different error types (validation vs authorization vs server error). Product creation/editing shows generic "something went wrong" instead of "Name is too short" or "Description contains banned words."

**Observable consequences today.** Every validation failure shows as a generic error or is silently swallowed. The user gets no specific feedback on what went wrong.

**What the right shape would look like.** Services throw (or return) the parsed error response. The UI layer maps `{field, code, translationKey}` to per-field error displays using the translation system. This is the core of what chat A (mobile validation rebuild) does.

**Maps to existing future chat?** A (mobile validation rebuild) — the entire service layer rebuild for product flows.

---

### Finding 19 — `<div>` elements in React Native code; will crash on native

**Category:** RN-convention
**Severity:** medium
**Surface:** filtering
**Files:**
- `src/components/navigation/Filters.tsx:142`
- `src/components/dialog/dialogs/FiltersDialog.tsx:143`

**What's there.** Both files have a fallback branch for unknown filter types: `return <div key={filter.id}>Filter not found {filter.filterKey}</div>`. This is a web HTML element, not a React Native component.

**Why it's structurally wrong.** React Native has no `<div>` component. If this code path is hit, the app will crash with an "Unknown component" error.

**Observable consequences today.** Latent — the fallback only fires for unknown filter types, which currently don't exist. But any new filter type that doesn't match the existing cases will crash the app instead of gracefully degrading.

**What the right shape would look like.** Replace `<div>` with `<View>` and the text content with `<Text>`.

**Maps to existing future chat?** D (mobile filtering & search polish) — B5 in Bucket 1.

---

### Finding 20 — `CategorySelector` and `CitySelector` render potentially hundreds of items in ScrollView

**Category:** performance
**Severity:** medium
**Surface:** product creation/editing, filtering
**Files:**
- `src/components/basic/CategorySelector.tsx:120` (entire category tree in ScrollView)
- `src/components/basic/CitySelector.tsx:96` (all regions and cities in ScrollView)

**What's there.** `CategorySelector` renders the entire category tree (top categories, subcategories, final categories) inside a `ScrollView`. `CitySelector` renders all regions and cities inside a `ScrollView`. Both could contain hundreds of items.

**Why it's structurally wrong.** `ScrollView` renders all children at mount time — there is no virtualization. With hundreds of items, this causes a large JS-to-native bridge payload, slow initial render, and high memory usage. `FlatList` or `SectionList` would virtualize the list and only render visible items.

**Observable consequences today.** Opening the category or city selector on a device with many categories/cities will be noticeably slow, especially on lower-end Android devices.

**What the right shape would look like.** `SectionList` with category groups as sections, or `FlatList` with a flat list of items. Alternatively, if the total count is bounded and small (<50), ScrollView is acceptable.

**Maps to existing future chat?** No — needs its own chat or folds into A (validation rebuild, which touches product creation UI).

---

### Finding 21 — Push notification for MESSAGE type ignores chat context; navigates to generic chat list

**Category:** platform-API
**Severity:** medium
**Surface:** push notifications / messaging
**Files:**
- `src/notifications/components/PushNotificationsInit.tsx:76-78`

**What's there.** When a MESSAGE push notification is tapped, the handler navigates to `/messages` without extracting `chatId` or sender UID from the notification `data`. The user lands on the chat list, not the specific conversation. Compare with the `SAVED_PRODUCT` case at line 73-75 which correctly extracts `data.productId`.

**Why it's structurally wrong.** On mobile, notification-to-content navigation is a core UX expectation. Tapping a "New message from User X" notification should open the conversation with User X, not the generic chat list requiring the user to find and tap the correct chat.

**Observable consequences today.** Users tapping message notifications must manually find the correct chat.

**What the right shape would look like.** Extract `chatId` (or sender UID) from `notification.request.content.data`, call `setActiveChatId(chatId)` before navigating, navigate to `/messages`. Handle the cold-start case via `getLastNotificationResponse`.

**Maps to existing future chat?** B (mobile messaging adoption).

---

### Finding 22 — Hydration race between Zustand persistence and init components

**Category:** state-shape
**Severity:** medium
**Surface:** app initialization
**Files:**
- `src/lib/store/authStore.ts:164-172` (persist with `onRehydrateStorage`)
- `src/components/init/ChatsInit.tsx:7` (reads `user` with no hydration guard)
- `src/components/init/InitFavoritesStore.ts:11` (listens to Firebase auth, no hydration guard)
- `src/notifications/components/NotificationsInit.tsx:6` (reads `user`, no hydration guard)
- `src/components/navigation/BottomBar.tsx:32` (`_hasHydrated` checked — correct)
- `src/components/user/UserMenu.tsx:61` (`_hasHydrated` checked — correct)

**What's there.** The auth store persists `{ user }` to AsyncStorage. The `_hasHydrated` flag is checked in exactly two UI components (BottomBar, UserMenu). All init components (`ChatsInit`, `InitFavoritesStore`, `NotificationsInit`) read `user` without checking `_hasHydrated`. On cold start, these see `user: null` before hydration completes, trigger cleanup logic (unsubscribe, reset), then re-trigger setup logic when hydration delivers the user — causing double initialization.

**Why it's structurally wrong.** The double-init is wasted work and can cause subtle bugs. `ChatsInit` calls `unsubscribeAll()` on the null user (no-op since nothing was subscribed), then `subscribeToChats` when the user arrives — this works but creates a timing window. `NotificationsInit` calls `reset()` on the null user, then `initialLoad` and `subscribe` when the user arrives — the `reset()` call is wasted.

**Observable consequences today.** Wasted network calls on cold start (double initialization of chat and notification listeners). Potential for brief UI flicker as components switch from "no user" to "has user" state.

**What the right shape would look like.** All init components check `_hasHydrated` before reading `user` state. Or use a single initialization coordinator that waits for hydration to complete before triggering any init logic.

**Maps to existing future chat?** E (mobile user-deletion adoption) — the auth initialization rework.

---

### Finding 23 — `useChatStore` is a 747-line monolithic god-store

**Category:** state-shape
**Severity:** medium
**Surface:** messaging
**Files:**
- `src/lib/store/useChatStore.ts` (747 lines)

**What's there.** A single Zustand store containing: chat list state, message state per chat, Firestore subscriptions, user cache (unbounded growth), navigation-like state (`activeChatId`, `tempReceiver`, `tempProductReason`), and full Firestore write paths (sending, deleting, marking seen). `loadingMore` is a single global boolean shared across all chats. `userCache` grows unboundedly — every message load adds users, never evicted.

**Why it's structurally wrong.** Any change to the chat list (e.g., new unread count) triggers re-renders in components that only care about messages. The monolithic shape makes it impossible to subscribe to only the data a component needs. `clearChatStore()` exists but is never called (Finding 3). The unbounded `userCache` is a memory leak over long sessions.

**Observable consequences today.** Re-render storms in the Messages component on chat-list changes (Finding 7). Memory growth proportional to the number of unique chat participants over the session lifetime.

**What the right shape would look like.** Split into at least: `useChatListStore` (chat list, pagination, unread counts), `useActiveChatStore` (active chat ID, messages, message pagination), `useChatUserCache` (user data with LRU eviction). Navigation state (`tempReceiver`, `tempProductReason`) in a separate ephemeral store.

**Maps to existing future chat?** B (mobile messaging adoption).

---

### Finding 24 — `reportService` boolean logic bug; `error` field is inverted

**Category:** web-seam
**Severity:** medium
**Surface:** user reporting
**Files:**
- `src/lib/services/reportService.ts:14-16`
- `src/lib/services/reportService.ts:22-24`

**What's there.** The return shape uses: `error: res.status !== 406 || (res.status >= 200 && res.status < 300)`. This evaluates to `true` for ALL successful (2xx) responses because `res.status !== 406` is `true` for any 2xx status. The `error` field is `true` when it should be `false`.

**Why it's structurally wrong.** The `error` field is always truthy for success responses, so the calling code (if it checks `error`) will treat every successful report submission as a failure.

**Observable consequences today.** Depends on how the caller handles the `error` field. If the caller shows an error toast on `error: true`, every report submission shows as failed even when it succeeded. If the caller ignores the `error` field, the bug is latent.

**What the right shape would look like.** Fix the boolean: `error: !(res.status >= 200 && res.status < 300)` or `error: res.status === 406`.

**Maps to existing future chat?** No — one-line bug fix, can ride into any chat that touches reporting.

---

### Finding 25 — `ProductList.onScroll` fires `setState` every frame; no `scrollEventThrottle`

**Category:** performance
**Severity:** medium
**Surface:** product feed
**Files:**
- `src/components/product/ProductList.tsx:244`

**What's there.** `onScroll={(e) => setShowScrollTop(e.nativeEvent.contentOffset.y > SCREEN_HEIGHT)}` fires `setState` on every scroll event. No `scrollEventThrottle` prop is set — on iOS, this defaults to 60fps firing. Every frame triggers a `setShowScrollTop` call, creating a new render cycle.

**Why it's structurally wrong.** Calling `setState` on every scroll frame causes unnecessary re-renders of the entire `ProductList` component. Combined with the lack of `React.memo` on child components (Finding 8), this cascades to every visible `ProductCard`.

**Observable consequences today.** Scroll jank on the product feed, especially on lower-end devices. The scroll-to-top button's visibility toggle doesn't need per-frame updates.

**What the right shape would look like.** Add `scrollEventThrottle={200}` to reduce firing frequency. Or use `useSharedValue` from Reanimated to track scroll position without triggering React re-renders, and use `useAnimatedStyle` for the button visibility.

**Maps to existing future chat?** No — needs a cross-cutting performance pass.

---

### Finding 26 — `Dimensions.get('window')` called at module level; doesn't update on rotation

**Category:** RN-convention
**Severity:** low
**Surface:** image viewer, floating button
**Files:**
- `src/components/FullScreenImageViewer.tsx:8`
- `src/components/FloatingButton.tsx:6`

**What's there.** `const { width, height } = Dimensions.get('window')` at module scope, executed once at import time.

**Why it's structurally wrong.** On rotation (if ever enabled) or window resize (multi-window Android), the dimensions won't update. Should use the `useWindowDimensions()` hook inside the component.

**Observable consequences today.** The app is locked to portrait (`orientation: 'portrait'` in `app.config.ts`), so this is latent. If portrait lock is ever removed, the image viewer and floating button will use stale dimensions.

**What the right shape would look like.** `const { width, height } = useWindowDimensions()` inside the component body.

**Maps to existing future chat?** No — minor, can ride into any chat touching these files.

---

## Cross-repo questions surfaced

**Q1 — Does the backend still serve the old `POST /secure/product/addUpdate` endpoint?** Already tracked as X3 in expo-release-readiness.md. Re-confirmed as relevant: the answer determines whether mobile product creation is currently broken against the validation-refactor backend.

**Q2 — What does the backend return in the error response body for 401 and 403 status codes?** Specifically: does a USER_BANNED 403 response include a machine-readable code (e.g., `{errors: [{code: "USER_BANNED"}]}`) that mobile can detect in a response interceptor? This shapes the design of Finding 5's fix.

**Q3 — Does the backend's `POST /secure/push/token` endpoint validate the `userId` body parameter against the authenticated principal?** Already tracked as X1 in expo-release-readiness.md. The mobile app sends `userId` in the push token registration body at `pushTokenService.ts:8-9`. If the backend trusts this field, any logged-in user can register push tokens under another user's ID.

---

## Bucket-1 bugs noticed in passing

- **B1 (session leak on logout)** — confirmed still present. `authService.ts:233-235` still only calls `GoogleSignin.signOut()`. Finding 2 above.
- **B5 (`<div>` in RN code)** — confirmed still present. `Filters.tsx:142` and `FiltersDialog.tsx:143`. Finding 19 above.
- **B10 (`NOTMAL` typo)** — confirmed still present in `src/notifications/lib/firebaseNotifications.ts`.
- **B11 (`getActiveChat` wrong collection)** — confirmed still present. `useChatStore.ts:711-720`. Finding 17 above.
- **B12 (hardcoded Serbian strings in messaging)** — confirmed still present. `Messages.tsx:103,129,220,227`, `Chats.tsx:32`, `useChatStore.ts:54`.
- **B16 (console.error/console.warn calls)** — confirmed still present across `authStore.ts`, `useChatStore.ts`, `userPreferenceStorage.ts`, `authService.ts`.
- **B17 (`DODAJ ZA BASE SITE` placeholder)** — confirmed still present at `app/owner/dashboard/user.tsx:230`.

---

## Out-of-scope observations

- **`react-dom` in production dependencies.** `package.json` lists `react-dom: 19.1.0` as a production dependency (~130KB minified). If the app is mobile-only (no Expo web), this is dead weight.
- **`dotenv` in production dependencies.** `package.json` lists `dotenv: ^17.3.1` in production dependencies. This is a Node.js module that doesn't run on React Native. Should be in `devDependencies` or removed.
- **`getUniqueID` imported from `react-native-markdown-display` in `ProductList.tsx:11`.** Using a markdown rendering library solely for generating unique IDs. A uuid or `crypto.randomUUID()` would be lighter.
- **`loginWithFacebookFirebase` is a dead stub.** `authService.ts:200-228` — entire body commented out, returns `null`. Called from `authStore.ts:108`.
- **12 exported service functions have zero consumers.** Detailed list: `getUserDetails`, `updateUserAuth`, `updateUser`, `getUserFollows`, `getUserForId` (userService), `getPortalProductDetails`, `getDashboardProductDetails`, `getPortalAutocompleteSuggestions`, `getDashboardAutocompleteSuggestions`, `getDashboardProducts` (productsSearchService), `markAsSeen` (productService), `healthCheck` (healthCheckService — entire file unused).
- **`healthCheckService.ts` is an entirely dead file.** Nothing imports from it.
- **`clearChatStore()` is defined but never called.** `useChatStore.ts:651-667`.
- **`clearCatalog()` is defined but never called.** `useCatalogStore.ts:25`.
- **`src/lib/types/cookie/` directory (3 files) is web-only dead code.** `GlobalCookie.ts`, `ConsentData.ts`, `UserPreference.ts` — never imported.
- **Three `.tsx` service files contain no JSX.** `configurationService.tsx`, `appVersionConfigService.tsx`, `maintenanceService.tsx`.
- **Dual store directory naming:** `src/lib/store/` (singular) vs `src/lib/stores/` (plural). Confusing.
- **`var` declarations in `useChatStore.ts:454` and `user.tsx:178`.** Should be `const`/`let`.
- **`KeyboardAvoidingView` behavior on Android is `'padding'` in messages screen but `'height'` in dialogs.** `messages.tsx:12` vs `DialogManager.tsx:74`. Inconsistent, and `'padding'` is often wrong on Android.
- **User settings screen (`user.tsx`) and product edit screen (`products/[productId].tsx`) are input-heavy forms with no `KeyboardAvoidingView`.** Keyboard will obscure lower fields on iOS.
- **`UserMenu.tsx` navigates to `/dashboard/balance` and `/dashboard/account-verification` — paths that don't exist.** Should be `/owner/dashboard/balance` and `/owner/dashboard/account-verification`. These would produce not-found screens.
- **`setActiveChatId(undefined)` at `ChatUserFunctionsDialog.tsx:45` — type mismatch.** Signature expects `string | null`, receives `undefined`.
- **Chat list FlatList has no pagination wired.** `Chats.tsx` renders a FlatList but never calls `loadMoreChats()`. Users with >15 chats never see older conversations.

---

## What this audit did NOT cover

- **Runtime verification.** The brief prohibits running the app. All findings are from code reading. Actual performance measurements (FPS, memory, JS thread load) require a running app on a real device.
- **Deep-link behavior.** The app has `scheme: 'oglasino'` configured but no deep-link handling code was found. Whether expo-router auto-handles deep links into these routes was not verified at runtime.
- **Expo config plugins.** `app.config.ts` plugin configuration was read but not validated against the actual native build output. Whether `expo-notifications`' `enableBackgroundRemoteNotifications` and other native config flags are correct requires a build + device test.
- **Android-specific issues.** No Android-specific testing. Keyboard behavior, navigation bar, status bar, and gesture handling differences between iOS and Android were inferred from code, not verified.
- **i18n system internals.** The translation loading/caching system (`src/i18n/`) was not audited deeply. The 21-calls-per-init finding from the readiness audit was not re-investigated.
- **Image upload pipeline.** `src/lib/images/uploadImages.ts` was not read deeply. The 401+TOKEN_EXPIRED retry logic there was noted but not audited for correctness.
- **Test coverage.** The existing test files were not reviewed for coverage or correctness.
- **`openAiService.ts` internals.** Read the signature but did not audit the AI suggestion flow deeply.

---

## For Mastermind

**Part 4a simplicity evidence:** N/A this session — read-only audit, no code added, no abstractions introduced, nothing simplified.

**Other notes for Mastermind:**

The audit surfaced a pattern that cross-cuts the existing future-chat queue: **performance-shape wrongness** (Findings 7, 8, 9, 10, 25) does not map cleanly to any single chat in the A–I queue. Chats A–I are feature-scoped; the performance issues (React.memo, Zustand selectors, AppContext memoization, expo-image adoption, scroll throttle) affect every feature surface equally. A dedicated cross-cutting "mobile performance pass" chat would be more effective than distributing these fixes across feature chats, because:

1. The re-render analysis needs to be done holistically (which components subscribe to which stores) rather than per-feature.
2. `expo-image` adoption is a search-and-replace across 14+ files, not a per-feature change.
3. `React.memo` on list items needs to be done alongside `renderItem` stabilization and store selector refactoring — doing them separately would leave the fix incomplete.

The **navigation rearchitecture** (Finding 12) is the most foundational issue. Moving from all-`<Slot />` to proper `<Stack>` and `<Tabs>` navigators affects every screen's transition behavior, header rendering, and tab state preservation. This should arguably be the first structural change, before feature adoption work, because feature chats (especially B for messaging and E for user-deletion) will need to work within the navigation structure.

Recommended sequencing amendment: insert a **"mobile structural foundation"** chat before A in the queue. Scope: native navigators (Finding 12), auth guards (Finding 13), expo-image adoption (Finding 9), React.memo on list items (Finding 8), Zustand selector refactoring (Finding 7). This establishes the structural floor that all subsequent feature chats build on.

**Findings the audit found harder to assess without runtime:**
- Finding 7's actual performance impact (re-render storms) — need a profiler trace to quantify.
- Finding 6's maintenance-as-offline behavior — need to test on a real device in airplane mode.
- Finding 22's hydration race — need cold-start timing to determine if the double-init window is user-visible.
