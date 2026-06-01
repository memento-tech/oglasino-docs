# Expo Performance Foundation (Φ3)

**Status:** `planned`
**Owner:** Igor
**Repo:** `oglasino-expo` only
**Branch:** `dev`
**Depends on:** Φ2 shipped (`features/expo-navigation-foundation.md` — native `<Tabs>` and `<Stack>` navigators, always-mounted Stack pattern, tab state preservation)
**Audit source:** [`sessions/audit-expo-phi3-performance.md`](../sessions/audit-expo-phi3-performance.md) (2026-05-27)
**Program context:** third of four foundation chats per [`features/expo-structural-foundation.md`](expo-structural-foundation.md)

## 1. What this feature does

Φ3 makes the mobile app behave like a native app under load. The visible design does not change. The structural code underneath changes so that lists scroll without re-rendering every item on every state change, images cache to disk, store updates only re-render the components reading the changed data, and the chat store's monolithic shape becomes three focused stores.

What users gain: smooth product-feed scrolling on mid-range Android; images loaded from disk after first fetch (no re-download on scroll-back); buttons and badges that don't flicker on unrelated state changes; the city selector in product creation opens without freezing the JS thread.

What does not change: the visual design of every screen; the API contracts with backend; the auth lifecycle shipped in Φ1; the navigation shape shipped in Φ2; the messaging behavior on mobile (the known divergences from web's 2026-05-20 messaging feature stay in place — chat B owns them).

## 2. Scope

In scope:

- **F7** — Zustand selector refactoring across all ~99 subscription sites. Replace whole-store destructures with single-field selectors or `useShallow` multi-field selectors.
- **F8** — `React.memo` on six list-item components plus stable `useCallback` for every `renderItem`. `useMemo`/`useCallback` discipline on parents passing props to memoized children (per S1).
- **F9** — Staged `expo-image` migration across 16 image sites (excluding the single `ImageBackground` in `BaseSiteSelector`).
- **F10** — Split `AppContext` into `AppStateContext` + `AppActionsContext` per S3.
- **F20** — Convert `CategorySelector` and `CitySelector` from `ScrollView` to `SectionList`.
- **F23** — Split `useChatStore` into `useChatListStore`, `useActiveChatStore`, `useChatNavStore` plus a non-Zustand `userCache` utility per S2. `loadingMore` becomes per-chat during the split per S8. The D9.5 "stuck-true on early return" bug is fixed during the split.
- **F25** — `scrollEventThrottle={200}` on `ProductList`. No Reanimated adoption.
- Adjacency F8.3 — `ProductCard` redundant `useEffect` for card size, replaced with direct selector during the F7 ProductCard work.

Out of scope:

- Reanimated adoption — confirmed Phase 1.
- Pre-Φ3 baseline measurement — confirmed Phase 1; Ψ measures post-Φ3 only.
- Backend, web, router, rules — Φ3 is mobile-only, no cross-repo work.
- Messaging behavioral divergences from web's 2026-05-20 feature — D9.1, D9.3, D9.4, D9.7, D9.9, D9.10, D9.11 all stay queued for chat B.
- The `getActiveChat` wrong-collection bug (D9.4), even though it sits inside `useActiveChatStore` after the split.
- `getUniqueID` from markdown library, dual store directory naming, `console.error`/`console.warn` cleanup — all Ω scope.

## 3. The seven scopes in detail

### 3.1 F7 — Zustand selectors

**Current state.** 99 subscription sites across 10 stores. Seven sites use whole-store destructuring (highest re-render risk). 18 sites use proper single-field selectors. ~42 sites use multi-selector patterns (separate `useStore(s => s.x)` calls). Zero `useShallow` usage.

**Target state.** Every subscription site uses one of three correct shapes:

- **Single-field selector** — `useStore(s => s.fieldName)` when one field is read.
- **`useShallow` multi-field selector** — `useStore(useShallow(s => ({ a: s.a, b: s.b })))` when multiple fields are read together.
- **`getState()` outside subscription** — `useStore.getState().action()` when no re-render dependency is needed (callbacks, effects).

Actions accessed via destructure are converted to `getState()` calls inside the callbacks that use them, unless the component already subscribes for state purposes (then the action ride alongs via `useShallow`).

**Critical refactor sites** (in order of impact, from audit Section 1):

| Site | File                                                 | Lines | Action                                                                                          |
| ---- | ---------------------------------------------------- | ----- | ----------------------------------------------------------------------------------------------- |
| F7.1 | `src/components/messages/Messages.tsx`               | 27–39 | Multi-selector via `useShallow` for the 6 read fields; actions via `getState()`                 |
| F7.2 | `src/components/navigation/BottomBar.tsx`            | 27–31 | Four separate single-field selectors (one per store, badge fields only)                         |
| F7.3 | `src/components/navigation/Filters.tsx` and siblings | 40–48 | `useShallow` multi-selector for the 4 read fields; actions via `getState()`                     |
| F7.4 | `src/components/product/ProductCard.tsx`             | 26–27 | Single-field selectors; folds into F8.3 (redundant effect removal)                              |
| F7.5 | `src/components/product/FavoriteButton.tsx`          | 25–27 | Two single-field selectors (`favoriteIds.includes(productId)` boolean, `toggleFavorite` action) |

All other ~94 sites get audited during the F7 brief. Most are minor (single-line conversion). The engineer's brief includes the full inventory from audit Section 1.

**Behavior preservation.** Every subscription site must observe the same data after refactoring as before. The component re-renders less often, but renders the same content when it does render.

### 3.2 F8 — React.memo and renderItem stability

**Components to memoize** (six list-item components):

- `ProductCard` — used in `ProductList` infinite-scroll feed
- `ProductReview` — used in `ReviewsList` and `OwnerReviewList`
- `ExtraProductCard` — used in `HorizontalExtraProductsListView`
- `UserCard` — used in `follows.tsx`
- `Message` — used in `Messages.tsx` (the message-group rendering)
- Chat-list row component in `Chats.tsx` (currently inline JSX — extract to named component first, then memoize)

Each component receives `React.memo` with a default shallow-props comparison unless props include nested objects that change identity but not value (then a custom comparator).

**`renderItem` stabilization.** Every FlatList's `renderItem` becomes a `useCallback`-stabilized reference. The 11 FlatLists from audit Section 2 all need this. The dependencies of the `useCallback` are the values the renderItem closes over (e.g., navigation callbacks, store-derived values).

**Per-brief responsibility (S1).** Every brief that adds `React.memo` to a child component also audits the parent's prop construction. For each prop passed to the memoized child:

- If the prop is a primitive — no action.
- If the prop is a stable reference (e.g., a function imported from a module) — no action; document in session summary.
- If the prop is a derived value or inline object/array/function — wrap parent's construction in `useMemo` or `useCallback`.

The brief explicitly states this expectation. Mastermind verdicts the session as REVISE if memoization is added without stabilizing the props.

**Adjacency F8.3 absorbed.** During F8 work on `ProductCard`, the redundant `useEffect` at `ProductCard.tsx:30-35` for card size is replaced with a direct selector based on `currentPortalScope`. The effect is deleted.

### 3.3 F9 — Staged expo-image migration

**Stage 1 — high-impact list sites** (one brief):

- `src/components/product/ProductTopImage.tsx:43` — product feed
- `src/components/ImagesCarousel.tsx:132` — product detail carousel
- `src/components/user/OglasinoAvatar.tsx:59` — chat lists, reviews
- `src/components/user/UserAvatar.tsx:15` — user details
- `src/components/messages/MessageImages.tsx:59` — chat images (token-gated)

These five sites carry the dominant performance payoff. They all live inside or are repeatedly rendered alongside lists. Disk caching here eliminates the scroll-back re-fetch pattern.

**MessageImages** is the trickiest of stage 1. It uses view tokens for private images with 401 retry-and-refresh logic. The migration preserves the retry pattern: `expo-image` supports the `headers` prop for custom auth headers; the `onError` event passes `{ error: string }` (shape change vs RN `Image`'s `void`). The retry flow adapts to the new error shape.

**Stage 2 — remaining sites** (one brief):

- `src/components/ZoomableImage.tsx:162`
- `src/components/messages/MessageInput.tsx:169` (local file picker)
- `src/components/ImagesImport.tsx:124,168` (mixed local/remote)
- `src/components/product/ProductReview.tsx:74`
- `src/components/icons/OglasinoIcon.tsx:48` (local asset)
- `src/components/init/BaseSiteSelector.tsx:120` (remote flag)
- `src/components/dialog/dialogs/PortalConfigDialog.tsx:129` (remote flag)
- `src/components/internals/AppVersionConfigInit.tsx:133` (local asset)
- `src/components/dialog/components/ProductReviewImageImport.tsx:125`
- `app/__smoke__/upload.tsx:215`
- `app/(portal)/(public)/about.tsx:61,77,85,104` (local assets)

**Excluded from migration:** `src/components/init/BaseSiteSelector.tsx:76` — `ImageBackground` from `react-native`. Expo-image has no `ImageBackground` equivalent. Stays as RN.

**Adaptation patterns per site:**

- `onLoadEnd` → `onLoad` (1 site: ProductTopImage). The new `onLoad` event passes `{ source: { width, height, url } }`. The skeleton-toggle logic adapts.
- `onError` shape change. RN: `onError={() => ...}`. Expo-image: `onError={({ error }) => ...}`. Three sites use `onError` today.
- `resizeMode` → `contentFit`. Same values (`cover`, `contain`). Nine sites.
- `cachePolicy="memory-disk"` added to every site by default. List sites also get `recyclingKey` set to the product/user/chat ID.

**Behavior preservation.** Every image renders identically to before. The only observable change is faster load on re-mount and persistence across cold starts.

### 3.4 F10 — AppContext two-context split (S3)

**Current state.** Single `AppContext` carrying state and actions. Provider value is a new object every render (3 of 4 actions lack `useCallback`). All 26 consumers re-render on every provider re-render.

**Target state.** Two contexts:

- **`AppStateContext`** — carries `status`, `baseSites`, `selectedBaseSite`, `selectedLanguage`, `configuration`, `getCurrentLocale`.
- **`AppActionsContext`** — carries `setBaseSiteForCode`, `setLanguageForCode`, `getConfiguration`, `reBootstrap`. All wrapped in `useCallback`. The actions context value is memoized once and never changes (after the actions are stable).

`useAppContext()` is replaced with two hooks:

- `useAppState()` returns the state context value.
- `useAppActions()` returns the actions context value.

Components reading only actions never re-render on state changes. Components reading only state never re-render when actions are reconstructed (they aren't, post-fix). The 22 consumers reading only `selectedBaseSite` still re-render on rare state changes (bootstrap, base-site switch) but that's already rare and not a performance issue.

**Migration of 26 consumers.** Each consumer is updated to call the right hook based on what it reads. Consumers reading both state and actions call both hooks. The brief includes the per-consumer mapping from audit Section 4.

**Behavior preservation.** Every component receives the same state and the same actions as before. The render frequency drops for action-only consumers.

### 3.5 F20 — SectionList for CategorySelector and CitySelector

**CitySelector** — highest risk. ~750 items rendered at mount in a `ScrollView`. Convert to `SectionList`:

- Sections = regions
- Items = cities within the region
- Section header renders the region name with current expand/collapse state if the existing behavior is preserved
- Search-mode rendering (`filteredRegions`) maps cleanly to filtered sections

**CategorySelector** — lower risk but in scope per "no dropping." ~375 items when fully expanded. Convert to `SectionList`:

- Sections = top-level categories
- Items = subcategories + final categories (flattened under the top)
- Existing 3-level expand/collapse remains; SectionList manages the rendering performance
- Search-mode rendering (`searchResults`) maps to a single flat section or a different render mode

**Behavior preservation.** The visual layout, animation (`Modal` with `animationType="slide"`), search behavior, expand/collapse behavior, and selection callbacks all stay identical. The container changes from `ScrollView` to `SectionList`; the children change from mapped JSX to SectionList's `renderSectionHeader` + `renderItem` props.

### 3.6 F23 — Chat store split

**Current state.** `src/lib/store/useChatStore.ts` is 746 lines holding chat list, active chat, messages, subscriptions, user cache, navigation state, and write paths.

**Target state.** Three Zustand stores plus one utility.

#### `useChatListStore`

State:

- `chats: ChatSummary[]`
- `hasMoreChats: boolean`
- `lastVisibleChat: DocumentSnapshot | null`
- `loadingMoreChats: boolean`
- `newMessagesCount: number`
- `unsubChats: (() => void) | null` (internal)

Actions:

- `subscribeToChats()`
- `loadMoreChats()`
- `clearChatList()` (called from logout)

#### `useActiveChatStore`

State:

- `activeChatId: string | null`
- `messages: Record<string, MessageGroup[]>`
- `hasMoreMessages: Record<string, boolean>`
- `lastVisibleMessage: Record<string, DocumentSnapshot>`
- `loadingMore: Record<string, boolean>` — **per-chat per S8**, no longer global
- `unsubMessages: Record<string, () => void>` (internal)

Actions:

- `subscribeToMessages(chatId)`
- `loadMoreMessages(chatId)` — **fixed to reset `loadingMore[chatId]: false` on early return** per S8 / D9.5
- `sendMessage(chatId, content)` — **atomic batch preserved**; reads `tempReceiver` and `tempProductReason` from `useChatNavStore` via `getState()`
- `setActiveChatId(chatId)`
- `getActiveChat(chatId)` — **D9.4 bug NOT fixed in Φ3**; the wrong-collection-shape behavior is preserved as-is; chat B fixes
- `deleteChat(chatId)`
- `unsubscribeFromMessages(chatId)`
- `clearActiveChat()` (called from logout)

#### `useChatNavStore`

State:

- `tempReceiver: UserInfoDTO | null`
- `tempProductReason: ProductDetailsDTO | null` — **name not changed in Φ3**; D9.1 stays for chat B

Actions:

- `setTempReceiver(receiver)` — preserves current behavior including the existing-chat-check side effect that sets `activeChatId` on the active-chat store via `getState()`
- `setTempProductReason(product)`
- `clearTempReceiver()`
- `clearChatNav()` (called from logout)

#### `userCache` utility (not a Zustand store, per S2)

Plain module at `src/lib/store/userCache.ts` exporting:

- `getCachedUser(firebaseUid: string): UserInfoDTO | null`
- `setCachedUser(firebaseUid: string, user: UserInfoDTO): void`
- `clearUserCache(): void` (called from logout)

Backed by a `Map<string, UserInfoDTO>` with LRU eviction at a cap of 500 entries. The cache module exports a function-based API; no React subscription. Consumers (the three new stores) call it directly when hydrating their own state.

**Cross-store dependencies handled via `getState()`:**

- `sendMessage` reads `tempReceiver` and `tempProductReason` via `useChatNavStore.getState()`.
- `setTempReceiver` sets `activeChatId` via `useActiveChatStore.getState().setActiveChatId()`.
- `deleteChat` clears `activeChatId` if the deleted chat is active.

**Logout coordination (S9).** `authStore.logout()` currently calls `useChatStore.getState().clearChatStore()`. Post-split, it calls all four:

```typescript
useChatListStore.getState().clearChatList();
useActiveChatStore.getState().clearActiveChat();
useChatNavStore.getState().clearChatNav();
clearUserCache();
```

This is added to the existing `authStore.logout()` cleanup sequence from Φ1.

**The old `useChatStore` is deleted.** All 9 consumers from audit Section 6 are updated to call the new stores. The brief carries the per-consumer migration mapping.

**Behavior preservation.** Every message sends, receives, paginates, marks-seen, and renders identically to before. The send-path atomic batch stays atomic. The listener lifecycle is preserved. The `loadingMore` per-chat fix is the only observable behavior change (the stuck-spinner bug from D9.5 is fixed during the split).

**What chat B inherits.** Three stores plus the utility, in the shape above. Chat B's job becomes:

- Rename `tempProductReason` → `tempProductContext` (D9.1)
- Localize hardcoded Serbian strings (D9.3)
- Fix `getActiveChat` collection shape (D9.4)
- Wire push notification chat context (D9.7)
- Fix `setActiveChatId(undefined)` (D9.9)
- Invert message list, fix key extractor (D9.10)
- Rethrow on send failure (D9.11)
- Wire chat-list pagination via `useChatListStore.loadMoreChats()` (D9.8)

These all live inside one of the three new stores or their consuming components. Chat B's structural work is minimal because the split lands in shapes that match where the fixes go.

### 3.7 F25 — ProductList scroll throttling

**Current state.** `src/components/product/ProductList.tsx:243` — `onScroll` handler fires `setState` every frame. No `scrollEventThrottle` prop.

**Target state.** `scrollEventThrottle={200}` added to the FlatList. Handler fires ~5 times per second instead of ~60. The handler body remains unchanged.

One-line change. No Reanimated.

**Behavior preservation.** The scroll-to-top button still appears when scroll position passes the threshold and disappears when below. The threshold check happens 5x/sec instead of 60x/sec; the perceived delay is well below the 200ms human threshold.

## 4. Trust boundaries

Φ3 does not change trust boundaries. The auth guards from Φ1 (server-derived `user` from `useAuthStore` after Firebase sync) remain the only mechanism gating secured surfaces. The chat store split moves write paths between stores but the write paths still use server-derived data (Firebase Auth UID, server-issued tokens) for authorization. No new client-supplied values reach moderation, authorization, or state-transition decisions.

Per `meta/conventions.md` Part 11 — explicitly verified.

## 5. Working condition contract

Every engineer brief in Φ3 carries this hard rule: **must not change observable behavior except where this spec explicitly says so.**

The explicit behavior changes are:

- F25 — scroll-to-top button check fires at ~5Hz instead of ~60Hz (imperceptible)
- F23 / S8 — `loadingMore` becomes per-chat and resets on early return (visible only as: previously-stuck spinner now disappears correctly)
- F9 — images load from disk cache after first fetch (visible only as: faster re-mounts, no scroll-back re-fetch)

Every other change is internal — fewer re-renders, smaller bundle pressure, cleaner separation. No screen renders different content. No store action behaves differently. No API contract changes.

If an engineer brief produces a change that would alter observable behavior outside the three explicit cases above, the brief is REVISE.

## 6. Definition of done

- All seven scopes shipped per section 3.
- Zero `React.memo` count → six list-item components have memoization.
- All 11 `renderItem` props use `useCallback`.
- All ~99 Zustand subscription sites use single-field selectors, `useShallow`, or `getState()`. Whole-store destructures removed.
- `expo-image` imported in 16 of 17 image sites (BaseSiteSelector ImageBackground excluded).
- `AppContext` split into `AppStateContext` + `AppActionsContext`. All 26 consumers migrated.
- `CategorySelector` and `CitySelector` use `SectionList`.
- `useChatStore` deleted. Three new stores + `userCache` utility exist. All 9 consumers migrated.
- `scrollEventThrottle={200}` on `ProductList`.
- `npx tsc --noEmit` exit 0, zero errors.
- `npm run lint` zero errors. Pre-existing warnings unchanged from Φ2 baseline (82 warnings).
- `npm test` ≥ 109 passed. Tests may be added or modified during the split but not net-removed.
- `npx expo-doctor` no new failures vs Φ2 baseline.
- Manual smoke per section 7 clears on real device.

## 7. Smoke verification (definition of "verifying")

After code lands and CI is green, Igor runs on a real device (Pixel 4a or equivalent mid-range Android plus iPhone):

**Product feed.**

- Scroll the home feed for 30 seconds. No hitches, no dropped frames perceptible.
- Scroll down, scroll back up. Images already loaded (disk cache).
- Toggle card size. Only the visible cards update; bottom bar and other UI don't flicker.
- Filter by category. New results render without stutter.

**Messaging.**

- Open chat list. Open a chat. Scroll messages. Send a message. Receive a message.
- Verify all messaging behavior matches pre-Φ3 (known bugs from web divergence still present; chat B owns those).
- Verify `loadingMore` spinner appears during pagination, disappears when paginated, disappears immediately if no more messages.
- Logout. Verify chat data cleared (badges drop to zero, lists empty).

**City and category selection.**

- Open product creation. Open city selector. Scroll the city list. No JS thread freeze.
- Open category selector. Expand top categories. No JS thread freeze.

**AppContext consumers.**

- Switch base site. Verify UI updates (top bar logo, currency).
- Switch language. Verify translations refresh.

**Foundation sanity (Φ1 + Φ2 still working).**

- Auth guards still redirect on `(secured)/`, `owner/`, `owner/dashboard/`.
- Tab switches preserve scroll position (Φ2).
- iOS swipe-back gesture works (Φ2).
- Foreground re-validation, ban dialog (Φ1).

Bugs surfaced during smoke get fixed in patch sessions before Φ3 closes. Once smoke clears, `state.md` flips Φ3 to `shipped` and Φ4 opens.

## 8. Engineer brief sequencing

Eleven briefs expected. One per session per conventions Part 5. Order:

**Brief 1 — F25 scroll throttle.** One-line change. Trivial. Runs first to clear a quick win and establish the pattern of "every brief sets a baseline post-change."

**Brief 2 — F7 across non-chat-store sites.** Selectors and `useShallow` everywhere except `useChatStore` consumers (those land with the split in Brief 9). Covers ~70 of the 99 sites. Highest-impact site: BottomBar (F7.2).

**Brief 3 — F8 part 1: ProductCard memoization + F8.3 absorption.** `React.memo` on `ProductCard`, `useCallback` on `ProductList.renderItem`, prop stabilization on parent. Redundant card-size effect deleted. Folds in F7.4.

**Brief 4 — F8 part 2: ProductReview, ExtraProductCard, UserCard.** Three more list-item components memoized with their parents' renderItem stabilized.

**Brief 5 — F10 AppContext split.** `AppStateContext` + `AppActionsContext`. All 26 consumers migrated. Larger brief — many files touched but mechanical.

**Brief 6 — F9 stage 1: list-site expo-image migration.** Five sites: ProductTopImage, ImagesCarousel, OglasinoAvatar, UserAvatar, MessageImages. MessageImages token-gated migration handled with care.

**Brief 7 — F9 stage 2: remaining expo-image sites.** Eleven sites.

**Brief 8 — F20 SectionList migration.** CitySelector first (highest risk), then CategorySelector.

**Brief 9 — F23 chat store split.** The largest brief. Three new stores + userCache utility. Old `useChatStore` deleted. Nine consumers migrated. `loadingMore` per-chat with stuck-true fix. F7 selectors land in consumers as part of this brief (since they're being rewritten anyway).

**Brief 10 — F8 part 3: Message component memoization + chat-list-row extraction and memoization.** Lands after the chat store split so the memoized components consume the new stores cleanly.

**Brief 11 — Stabilization and verification.** Runs after manual smoke surfaces issues. May be skipped if smoke is clean. Includes Risk Watch entry drafts for `state.md`.

The engineer may consolidate brief pairs if scope is small enough or split a brief further if scope grows during implementation. Mastermind drafts one brief at a time, sees the result, then drafts the next.

## 9. Status

Φ3 status flips to `shipped` only after:

- All section 6 definition-of-done items confirmed in the final engineer session summary.
- Manual smoke per section 7 clears on real device.

Until both clear, Φ3 status is `verifying` (per the Φ1 and Φ2 precedents in `state.md`).

## 10. Factual vs inferred

**Factual:**

- All 99 subscription sites and their current shape (audit Section 1).
- All 11 FlatLists, all 17 image sites, all 26 AppContext consumers, all 9 chat-store consumers (audit Sections 2, 3, 4, 6).
- The natural 4-way chat store split mapping (audit Section 6).
- The 11 messaging divergences and their structural-vs-behavioral classification (audit Section 9).
- The decisions S1 through S5 (Phase 3 seam analysis, confirmed by Igor 2026-05-27).
- F8.3 absorption into F8 (ProductCard work touches the same file).

**Inferred:**

- Brief sequencing in section 8 — engineer judgment may amend.
- The 500-entry LRU cap on `userCache` utility — engineer can revisit if a smaller or larger cap fits the observed memory profile.
- `useShallow` vs separate single-field selectors per site — engineer chooses per site based on which pattern is more readable; either is correct.
- The exact API shape of `userCache` utility (function exports vs class) — engineer chooses.

## 11. References

- Φ3 audit: [`sessions/audit-expo-phi3-performance.md`](../sessions/audit-expo-phi3-performance.md)
- Structural audit: [`sessions/audit-expo-structural.md`](../sessions/audit-expo-structural.md)
- Φ1 spec: [`features/expo-auth-lifecycle.md`](expo-auth-lifecycle.md)
- Φ2 spec: [`features/expo-navigation-foundation.md`](expo-navigation-foundation.md)
- Program spec: [`features/expo-structural-foundation.md`](expo-structural-foundation.md)
- Decisions log: messaging feature (2026-05-20), Φ2 close-out (2026-05-27), Φ1 close-out (2026-05-25)
