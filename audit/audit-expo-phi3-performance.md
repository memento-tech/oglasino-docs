# Audit — Φ3 performance surface

**Repo:** `oglasino-expo`
**Branch:** `new-expo-dev`
**Date:** 2026-05-27
**Type:** read-only audit
**Scope:** seven structural audit findings (F7, F8, F9, F10, F20, F23, F25) plus cross-cutting adjacencies and messaging contract divergence

---

## Section 1 — Zustand subscription map (F7)

### Stores inventoried (10 total)

| Store | File | Persist? |
|---|---|---|
| `useAuthStore` | `src/lib/store/authStore.ts` | Yes (AsyncStorage) |
| `useChatStore` | `src/lib/store/useChatStore.ts` | No |
| `useFavoritesStore` | `src/lib/store/useFavoritesStore.ts` | No |
| `useNotificationStore` | `src/notifications/store/useNotificationStore.ts` | No |
| `useCardSizeStore` | `src/lib/store/useCardSizeStore.ts` | Manual AsyncStorage hydration |
| `usePortalFilterStore` | `src/lib/store/useFilterStore.ts` (factory instance) | No |
| `useDashboardFilterStore` | `src/lib/store/useFilterStore.ts` (factory instance) | No |
| `useDialogStore` | `src/components/dialog/store/useDialogStore.ts` | No |
| `useChatBlockStore` | `src/lib/store/useChatBlockStore.ts` | No |
| `useViewTokenStore` | `src/lib/stores/viewTokens.ts` | No |

**Note:** `useCatalogStore` was deleted during Φ1 dead-code cleanup. The structural audit's reference to it is stale.

### Subscription site count

**Total subscription sites: 99** (up from the structural audit's "22+" estimate). Breakdown:

- Proper single-field selectors (`useStore(s => s.field)`): ~42
- Multi-field destructure from whole store (`const { a, b } = useStore()`): 7 **← high re-render risk**
- Multi-selector (separate `useStore(s => s.x)` calls per field): ~18
- `getState()` (no subscription, safe): ~12
- Prop-passed store (generic filter components): ~20

**Zero `useShallow` usage** anywhere in the codebase.

### Critical re-render risk sites

#### F7.1 — `src/components/messages/Messages.tsx:27-39`

Destructures 11 fields from `useChatStore()`:

```tsx
const {
  hasMoreMessages, messages, loadingMore, loadMoreMessages,
  sendMessage, setActiveChatId, getActiveChat, activeChatId,
  tempReceiver, tempProductReason, chats
} = useChatStore();
```

**Fields actually read:** `messages[activeChatId]`, `hasMoreMessages[activeChatId]`, `loadingMore`, `activeChatId`, `tempReceiver`, `tempProductReason`, `chats` (for `getActiveChat` lookup).

**Re-render risk:** CRITICAL. The `messages` object can be enormous (per-chat arrays of `MessageGroup[]`). Any message arrival in ANY chat, any chat-list update, any `userCache` addition, any `loadingMore` toggle re-renders the entire Messages component — including the FlatList with potentially hundreds of message items.

**Actions via subscription:** `loadMoreMessages`, `sendMessage`, `setActiveChatId`, `getActiveChat` are all accessed via destructure (subscription), not `getState()`. These are stable function references in Zustand, so they don't cause re-renders themselves, but they unnecessarily widen the subscription surface.

#### F7.2 — `src/components/navigation/BottomBar.tsx:27-31`

Subscribes to 4 stores:

```tsx
const openDialog = useDialogStore((s) => s.openDialog);     // selector — OK
const { user, _hasHydrated } = useAuthStore();               // whole-store destructure
const { favoriteIds } = useFavoritesStore();                 // whole-store destructure
const { unseenCount } = useNotificationStore();              // whole-store destructure
const { newMessagesCount } = useChatStore();                 // whole-store destructure
```

**Re-render risk:** MEDIUM-HIGH. BottomBar re-renders on any `authStore` mutation (loading toggle, error set, restored flag), any `favoritesStore` mutation (the `favoriteIds` array reference changes on every toggle), any `notificationStore` mutation, and any `chatStore` mutation. BottomBar is always mounted (it's the tab bar), so these re-renders are constant.

**Fix shape:** Convert to selectors: `useAuthStore(s => s.user)`, `useAuthStore(s => s._hasHydrated)`, `useFavoritesStore(s => s.favoriteIds.length)` (if only badge count needed), `useNotificationStore(s => s.unseenCount)`, `useChatStore(s => s.newMessagesCount)`.

#### F7.3 — `src/components/navigation/Filters.tsx:40-48`

Destructures 7 fields from `usePortalFilterStore`:

```tsx
const {
  selectedFilters, selectedOrder, selectedPriceRange,
  selectedRegionsAndCities, addRemoveOptionFilter,
  addRemoveRangeFilter, clearAllFilters
} = useFilterStore();
```

**Re-render risk:** HIGH. `selectedFilters` is an array, `selectedPriceRange` and `selectedRegionsAndCities` are nested objects. Any filter toggle re-renders the entire Filters component. Same pattern in `FiltersDialog.tsx:39-50` and `FilteredProductList.tsx:49`.

#### F7.4 — `src/components/product/ProductCard.tsx:26-27`

Each ProductCard subscribes to 2 stores:

```tsx
const { getSizeForPortalScope, portalSize, dashboardSize } = useCardSizeStore();
const { currentPortalScope } = usePortalScope();
```

**Re-render risk:** MEDIUM. Every visible ProductCard re-renders when either card-size store or portal-scope changes. In the product feed with 20+ visible cards, a single card-size toggle causes 20+ re-renders.

#### F7.5 — `src/components/product/FavoriteButton.tsx:25-27`

Three separate selector calls:

```tsx
const { favoriteIds } = useFavoritesStore();
const { presetClearShouldReload } = useFavoritesStore();
const { toggleFavorite } = useFavoritesStore();
```

**Re-render risk:** MEDIUM. `favoriteIds` is an array reference that changes on every toggle. Each FavoriteButton in the product feed re-renders when any product is favorited/unfavorited.

### Safe subscription patterns (post-Φ1)

Φ1 introduced proper selector patterns in several files:

- `src/components/init/ChatsInit.tsx:7-8`: `useAuthStore((s) => s.user)` and `useAuthStore((s) => s._hasHydrated)` — correct.
- `src/components/messages/Chats.tsx:12-14`: Three separate selectors per field — correct.
- `src/components/init/ForegroundRevalidationInit.tsx:28,36,39`: Uses `getState()` — correct, no subscription.
- `src/components/messages/MessageImages.tsx:30,50`: Uses `useViewTokenStore.getState()` — correct.

### Actions accessed via `getState()` (no re-render — correct)

12+ sites use `.getState()` for action dispatch inside callbacks:
- `ForegroundRevalidationInit` (auth state reads)
- `MessageImages` (view token management)
- Internal store actions (cross-store calls in `useChatStore`, `useChatBlockStore`, `authStore`)

---

## Section 2 — React.memo and renderItem stability (F8)

### React.memo usage

**Confirmed: ZERO `React.memo` usage across the entire codebase.** No component anywhere uses `React.memo` or `memo()` from React. The structural audit's claim is correct.

### FlatList/SectionList inventory (11 total)

| # | File | Line | Item Component | memo? | renderItem shape | Item store subs? |
|---|---|---|---|---|---|---|
| 1 | `src/components/product/ProductList.tsx` | 221 | `CardContentComponent` (prop — typically `PortalProductCard` → `ProductCard`) | No | Inline arrow | Yes: `useCardSizeStore`, `usePortalScope` |
| 2 | `src/components/product/HorizontalExtraProductsListView.tsx` | 71 | `ExtraProductCard` | No | Inline arrow | No |
| 3 | `src/components/messages/Messages.tsx` | 203 | Inline JSX (`renderMessageGroup` → `Message`) | No | Component-level arrow (not `useCallback`) | No (parent subscribes) |
| 4 | `src/components/messages/Chats.tsx` | 51 | Inline JSX (Pressable) | No | Inline arrow | No (parent subscribes) |
| 5 | `src/components/ReviewsList.tsx` | 65 | `ProductReview` | No | Inline arrow | Yes: `useAuthStore` |
| 6 | `src/components/ImagesCarousel.tsx` | 82 | Inline JSX | No | Inline arrow | No |
| 7 | `src/components/SearchInput.tsx` | 208 | Inline JSX (TouchableOpacity) | No | Inline arrow | No |
| 8 | `app/(portal)/(secured)/notifications.tsx` | 70 | Inline JSX | No | Inline arrow | No (parent subscribes) |
| 9 | `app/owner/dashboard/follows.tsx` | 46 | `UserCard` | No | Inline arrow | No |
| 10 | `src/components/filters/RangeUtilPicker.tsx` | 52 | Inline JSX | No | Inline arrow | No |
| 11 | `src/components/dashboard/components/OwnerReviewList.tsx` | 62 | `GivenReviewCard` / `ReceivedReviewCard` (→ `ProductReview`) | No | Inline arrow | Yes: `useAuthStore` (via ProductReview) |

**Every single `renderItem` is an inline arrow function** — none use `useCallback`. This defeats FlatList's built-in optimization of comparing `renderItem` identity across renders.

### ScrollView with mapped children (5 additional sites)

| File | Line | Children | Item count |
|---|---|---|---|
| `src/components/basic/CategorySelector.tsx` | 120 | Nested category tree | 500+ when expanded |
| `src/components/basic/CitySelector.tsx` | 96 | Region → city groups | 100–1500 |
| `src/components/messages/MessageInput.tsx` | 142 | Image thumbnails | ~5–10 |
| `src/components/navigation/CategoryNavigation.tsx` | 28, 57 | Category chips + subcategory list | ~15–50 |
| `src/components/filters/DateFilterSelector.tsx` | 53 | Year list | ~10 |

### Item components with independent store subscriptions

These create per-item subscription overhead inside lists:

- **`ProductCard`** subscribes to `useCardSizeStore` and `usePortalScope`. With 20 visible cards, that's 40 store subscriptions, all re-rendering on any card-size change.
- **`ProductReview`** subscribes to `useAuthStore` (reads `user.id` for ownership comparison). Every review item re-renders on any auth state change.

---

## Section 3 — Image usage (F9)

### expo-image status

- **In `package.json`:** Yes — `"expo-image": "~3.0.11"`
- **Imports in codebase:** ZERO. Confirmed by searching all `.tsx` and `.ts` files.
- **All image rendering uses:** `Image` from `react-native`

### Image usage sites (16 total)

| # | File | Line | Source type | In list? | Caching props? | Error handling? |
|---|---|---|---|---|---|---|
| 1 | `src/components/product/ProductTopImage.tsx` | 43 | Remote URI (`publicImageUrl(key, 'card')`) | No (but rendered per card in ProductList) | None | `onLoadEnd` + `onError` with skeleton |
| 2 | `src/components/ImagesCarousel.tsx` | 132 | Remote URI | Yes (FlatList) | None | `onError` fallback |
| 3 | `src/components/ZoomableImage.tsx` | 162 | Remote URI | No | None | `onError` fallback |
| 4 | `src/components/user/OglasinoAvatar.tsx` | 59 | Remote URI (`publicImageUrl(key, 'card')`) | No | None | None |
| 5 | `src/components/user/UserAvatar.tsx` | 15 | Remote URI | No | None | None |
| 6 | `src/components/messages/MessageImages.tsx` | 59 | Remote URI (private, token-gated) | Yes (inside message list) | None | Token invalidation + retry |
| 7 | `src/components/messages/MessageInput.tsx` | 169 | Local file (picker) | Yes (ScrollView) | None | None |
| 8 | `src/components/ImagesImport.tsx` | 124 | Mixed (local file / remote) | No | None | None |
| 9 | `src/components/ImagesImport.tsx` | 168 | Mixed (local file / remote) | Yes (ScrollView) | None | None |
| 10 | `src/components/product/ProductReview.tsx` | 74 | Remote URI | No | None | None |
| 11 | `src/components/icons/OglasinoIcon.tsx` | 48 | Local asset (4 variants) | No | None | None |
| 12 | `src/components/init/BaseSiteSelector.tsx` | 120 | Remote URI (flag) | No | None | None |
| 13 | `src/components/dialog/dialogs/PortalConfigDialog.tsx` | 129 | Remote URI (flag) | No | None | None |
| 14 | `src/components/internals/AppVersionConfigInit.tsx` | 133 | Local asset | No | None | None |
| 15 | `src/components/dialog/components/ProductReviewImageImport.tsx` | 125 | Local file (picker) | Yes (thumbnail grid) | None | None |
| 16 | `app/__smoke__/upload.tsx` | 215 | Local file (picker) | No | None | None |
| 17 | `app/(portal)/(public)/about.tsx` | 61, 77, 85, 104 | Local asset | No | None | None |

### ImageBackground usage

One site: `src/components/init/BaseSiteSelector.tsx:76` — `ImageBackground` from `react-native` for intro screen background with darkening overlay.

### Advanced image props usage

**Zero** across the entire codebase. No usage of: `placeholder`, `transition`, `priority`, `recyclingKey`, `cachePolicy`, `contentFit` (expo-image), `blurRadius`, `fadeDuration`.

### Image static method usage

**Zero.** No `Image.getSize()`, `Image.prefetch()`, or `Image.queryCache()` calls found.

### Adaptation patterns for expo-image migration

1. **`onLoadEnd` → `onLoad`:** `ProductTopImage` uses `onLoadEnd` for skeleton toggle. expo-image's `onLoad` event has a different shape (`{ source: { width, height, url } }`).
2. **`onError` shape change:** expo-image's `onError` passes `{ error: string }`, not `void`. `ImagesCarousel`, `ZoomableImage`, `MessageImages` all use `onError` with no error argument — compatible but pattern differs.
3. **`resizeMode` → `contentFit`:** 9 sites use `resizeMode` prop. expo-image uses `contentFit` with the same values (`cover`, `contain`).
4. **`ImageBackground`:** No expo-image equivalent. The single `BaseSiteSelector` usage would need to stay as RN `ImageBackground` or be restructured.
5. **Token-gated images:** `MessageImages` uses view tokens for private images. expo-image supports custom headers via `headers` prop, but the retry-on-401 pattern would need adaptation.

### Highest-impact migration targets

1. **`ProductTopImage`** — rendered per card in the infinite-scroll product feed. Most visible. Disk caching and placeholder support would eliminate re-fetches on scroll-back.
2. **`ImagesCarousel`** — inside FlatList. `recyclingKey` and `cachePolicy` would reduce memory churn.
3. **`OglasinoAvatar`** — appears in chat lists, reviews, user details. Disk caching prevents re-fetch on re-mount.
4. **`MessageImages`** — inside message list. Disk caching for already-viewed chat images.

---

## Section 4 — AppContext shape (F10)

### State shape

**File:** `src/components/context/AppContext.tsx`

```typescript
interface OglasinoAppState {
  status: 'loading' | 'maintenance' | 'select-base-site' | 'ready';
  baseSites: BaseSiteDTO[];
  selectedBaseSite?: BaseSiteDTO;
  selectedLanguage?: LanguageDTO;
  configuration?: ConfigMap;
  getCurrentLocale?: () => string;
}
```

| Field | Change frequency |
|---|---|
| `status` | Rare — on bootstrap, maintenance transitions |
| `baseSites` | Once — loaded during bootstrap, immutable after |
| `selectedBaseSite` | Rare — user switches base site |
| `selectedLanguage` | Rare — user switches language |
| `configuration` | Once — loaded during bootstrap, immutable after |
| `getCurrentLocale` | When language changes (recreated as closure) |

### Action functions

| Function | Lines | `useCallback`? | Change frequency |
|---|---|---|---|
| `setBaseSiteForCode(siteCode, withLoading?)` | 183–213 | **No** | New reference every render |
| `setLanguageForCode(langCode)` | 215–242 | **No** | New reference every render |
| `getConfiguration(key)` | 244 | **No** | New reference every render |
| `reBootstrap` (aliased from `bootstrap`) | 84 | Yes (`useCallback`) | Stable |
| `setStateWithStatusHook` (internal) | 64 | Yes (`useCallback`) | Stable |
| `bootstrap` (internal) | 84 | Yes (`useCallback`) | Stable |

### Provider value construction

**Lines 249–258:** New object every render.

```tsx
<AppContext.Provider
  value={{
    ...state,
    setBaseSiteForCode,
    setLanguageForCode,
    getConfiguration,
    reBootstrap: bootstrap,
  }}
```

The spread of `state` plus three non-memoized function references means the provider value is a **new object on every render**. All 26 consumers of `useAppContext()` re-render on every `AppContextProvider` re-render, regardless of which fields they actually read.

### Maintenance poll

**Line 176:** `setInterval(checkIfMaintenance, 5000)` — polls every 5 seconds.

The poll calls `checkIfMaintenance()` which fetches from the backend. If the maintenance flag changes, it calls `setStateWithStatusHook()` → state update → provider value recreated → all 26 consumers re-render. If transitioning from maintenance to ready, it calls `bootstrap()` → full state refresh.

**In practice:** The poll short-circuits when status hasn't changed (the `setState` only runs on actual change). But because the action functions (`setBaseSiteForCode`, etc.) are not `useCallback`-wrapped, any re-render of the provider — even one that doesn't change state — produces new function references that break the provider value's identity.

### Consumer inventory (26 files)

| File | Fields/actions read |
|---|---|
| `app/(portal)/_layout.tsx:12` | `selectedBaseSite` |
| `app/(portal)/(public)/catalog/[...categories].tsx` | `selectedBaseSite` |
| `app/(portal)/(public)/product/[...productData].tsx` | `selectedBaseSite`, `setBaseSiteForCode`, `status` |
| `app/owner/_layout.tsx:21` | `selectedBaseSite` |
| `app/owner/dashboard/products/[productId].tsx` | `selectedBaseSite`, `getConfiguration` |
| `src/components/ConsumerProtectionBanner.tsx:14` | `selectedBaseSite` |
| `src/components/SearchInput.tsx:49` | `selectedBaseSite` |
| `src/components/basic/CategorySelector.tsx:38` | `selectedBaseSite` |
| `src/components/basic/CitySelector.tsx:29` | `selectedBaseSite` |
| `src/components/dialog/dialogs/CurrencySelectionDialog.tsx` | `selectedBaseSite` |
| `src/components/dialog/dialogs/FiltersDialog.tsx` | `selectedBaseSite` |
| `src/components/dialog/dialogs/PortalConfigDialog.tsx` | `selectedBaseSite`, `selectedLanguage`, `baseSites`, `setBaseSiteForCode`, `setLanguageForCode` |
| `src/components/dialog/dialogs/PreviewProductDialog.tsx` | `selectedBaseSite` |
| `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` | `selectedBaseSite` |
| `src/components/dialog/dialogs/product-creation/BasicInfoProductDialog.tsx:43` | `getConfiguration` |
| `src/components/filters/CurrencySelector.tsx:19` | `selectedBaseSite` |
| `src/components/filters/ProductOrder.tsx` | `selectedBaseSite` |
| `src/components/filters/RegionCityFilter.tsx:22` | `selectedBaseSite` |
| `src/components/init/BaseSiteSelector.tsx` | `setBaseSiteForCode`, `baseSites` |
| `src/components/navigation/CategoryNavigation.tsx` | `selectedBaseSite` |
| `src/components/navigation/Filters.tsx` | `selectedBaseSite` |
| `src/components/navigation/Footer.tsx` | `selectedBaseSite` |
| `src/components/navigation/TopBar.tsx` | `selectedBaseSite` |
| `src/components/product/ProductBreadcrumb.tsx` | `selectedBaseSite` |
| `src/components/product/ProductList.tsx` | `selectedLanguage` |
| `src/components/user/UserMenu.tsx` | `selectedBaseSite`, `setBaseSiteForCode` |

**Dominant read:** 22 of 26 consumers read only `selectedBaseSite`. Only 3 read actions (`setBaseSiteForCode`, `setLanguageForCode`, `getConfiguration`). Only 1 reads `baseSites`. Only 1 reads `selectedLanguage`. Only 1 reads `status`.

---

## Section 5 — CategorySelector and CitySelector (F20)

### CategorySelector

**File:** `src/components/basic/CategorySelector.tsx`

| Property | Value |
|---|---|
| Container | `ScrollView` (line 120) |
| Render shape | 3-level nested expansion tree: top → sub → final categories |
| Total item count | Depends on `selectedBaseSite.catalog.categories`. Realistic: ~15 top categories × ~5 subcategories × ~5 final = **~375 items when fully expanded**. Could exceed 500+. |
| Search mode | Flattens to simple list of matching final categories (`searchResults` via `useMemo`) |
| Open frequency | Only from `BasicInfoProductDialog` (product create/edit) — not on every screen |
| Animation | `Modal` with `animationType="slide"` (line 105) |
| Virtualization | **None.** All expanded items render at once. |

**Performance concern:** When search is empty and user expands all top categories, the ScrollView renders the entire tree at mount time. The JS-to-native bridge payload is large. On lower-end Android, this causes a noticeable pause.

**Mitigating factor:** Categories are loaded once per session (from `selectedBaseSite.catalog`), and the selector is opened infrequently (product creation only). The real risk is the initial render when many categories are expanded simultaneously.

### CitySelector

**File:** `src/components/basic/CitySelector.tsx`

| Property | Value |
|---|---|
| Container | `ScrollView` (line 96) |
| Render shape | 2-level: regions (headers) → cities (always visible under each region) |
| Total item count | Depends on `selectedBaseSite.regions`. Realistic for Serbian market: ~25 regions × ~30 cities = **~750 items, all rendered at once**. Could reach 1000–1500 in larger markets. |
| Search mode | `filteredRegions` via `useMemo` filters to matching cities |
| Open frequency | Only from product create/edit — not on every screen |
| Animation | `Modal` with `animationType="slide"` (line 83) |
| Virtualization | **None.** All regions and cities render at mount time. |

**Performance concern:** HIGH. Unlike CategorySelector (which hides items behind expand/collapse), CitySelector renders ALL cities flat at mount. 750+ items in a ScrollView causes a significant JS → native bridge payload, slow initial render, and high memory usage. This is the most likely performance regression in the product creation flow.

**Fix shape:** Replace ScrollView with `SectionList` (regions as sections, cities as items). The item count and nested structure map naturally to SectionList's section/item model.

### Similar selector components

| Component | Container | Max items | Risk |
|---|---|---|---|
| `src/components/filters/RegionCityFilter.tsx` | `View` (expandable) | ~750 when all expanded | MEDIUM — items hidden behind expand/collapse |
| `src/components/filters/MultiOptionFilterSelector.tsx` | `View` | ~30 per filter | LOW |
| `src/components/filters/SingleOptionFilterSelector.tsx` | `View` | ~10 per filter | LOW |
| `src/components/basic/Select.tsx` | `ScrollView` | Varies | LOW — typically small datasets |

---

## Section 6 — useChatStore consumer map (F23)

### Store anatomy

**File:** `src/lib/store/useChatStore.ts` — 746 lines.

#### State fields (lines 93–110)

**Chat list:**
- `chats: ChatSummary[]` — active chats for current user
- `hasMoreChats: boolean` — pagination flag
- `lastVisibleChat: DocumentSnapshot | null` — pagination cursor
- `loadingMoreChats: boolean` — loading state for chat list pagination
- `newMessagesCount: number` — total unread badge count

**Active chat / messages:**
- `messages: Record<string, MessageGroup[]>` — per-chat message groups
- `activeChatId: string | null` — currently displayed chat
- `hasMoreMessages: Record<string, boolean>` — per-chat pagination flags
- `lastVisibleMessage: Record<string, DocumentSnapshot>` — per-chat pagination cursors
- `loadingMore: boolean` — **SINGLE GLOBAL BOOLEAN, NOT PER-CHAT** ← confirmed

**Ephemeral navigation state:**
- `tempReceiver: UserInfoDTO | null` — temp receiver for new-chat flow
- `tempProductReason: ProductDetailsDTO | null` — product context for first message

**User cache:**
- `userCache: Record<string, UserInfoDTO>` — keyed by `firebaseUid`
- **Growth pattern:** Unbounded. Added in `subscribeToChats` (lines 152, 157), `loadMoreChats` (lines 206, 211), `subscribeToMessages` (lines 261, 269), `loadMoreMessages` (lines 345, 352), `sendMessage` (line 400). **Never pruned** — only cleared on full `clearChatStore()`.

**Subscriptions (internal):**
- `unsubChats: (() => void) | null` — chat list listener unsubscribe
- `unsubMessages: Record<string, () => void>` — per-chat message listener unsubscribes

#### Actions (lines 113–745)

**Chat list actions:**
- `subscribeToChats()` — `onSnapshot` on `userchats/{uid}/chats`, limit 15
- `loadMoreChats()` — paginated load with `startAfter`

**Message actions:**
- `subscribeToMessages(chatId)` — `onSnapshot` on `chats/{chatId}/messages`, limit 15; marks seen via `writeBatch`
- `loadMoreMessages(chatId)` — paginated load; writes to global `loadingMore`
- `sendMessage(chatId, content)` — new-chat (atomic `writeBatch`) or existing-chat (3 sequential writes)

**Chat management:**
- `deleteChat(chatId)` — unsubscribes, deletes local state, Firestore `userchats` entry
- `unsubscribeFromMessages(chatId)` — cleanup single chat listener
- `unsubscribeAll()` — cleanup all listeners
- `clearChatStore()` — full reset

**Navigation/temp state:**
- `setTempReceiver(receiver)` — checks for existing chat, sets `activeChatId`
- `setTempProductReason(product)` — stores product context
- `clearTempReceiver()` — clears temp state
- `getActiveChat(chatId)` — finds chat locally, falls back to Firestore read, calls `subscribeToMessages`
- `setActiveChatId(chatId)` — sets active chat

**User cache:**
- `getUserData(firebaseUid)` — cache-first, fetches on miss

#### Firestore listeners

| Listener | Collection | Query | Stored in | Subscribed | Unsubscribed |
|---|---|---|---|---|---|
| Chat list | `userchats/{uid}/chats` | `orderBy('lastUpdated', desc), limit(15)` | `unsubChats` | `subscribeToChats:137` | `subscribeToChats:135` (re-sub), `unsubscribeAll:646`, `clearChatStore:653` |
| Per-chat messages | `chats/{chatId}/messages` | `orderBy('createdAt', desc), limit(15)` | `unsubMessages[chatId]` | `subscribeToMessages:241` | `deleteChat:602`, `unsubscribeFromMessages:636`, `unsubscribeAll:647`, `clearChatStore:654` |

### Consumer inventory (9 files)

| File | Category | Fields read | Actions called |
|---|---|---|---|
| `app/(portal)/(secured)/messages.tsx:11` | Nav state | `activeChatId` | — |
| `src/components/init/ChatsInit.tsx:10-11` | Subscription lifecycle | — | `subscribeToChats`, `unsubscribeAll` |
| `src/components/messages/Chats.tsx:12-14` | Chat list UI | `chats`, `activeChatId` | `setActiveChatId` |
| `src/components/messages/Messages.tsx:27-39` | Active chat UI | `messages`, `hasMoreMessages`, `loadingMore`, `activeChatId`, `tempReceiver`, `tempProductReason`, `chats` | `loadMoreMessages`, `sendMessage`, `setActiveChatId`, `getActiveChat` |
| `src/components/product/ProductUserDetails.tsx:52` | Nav state setter | — | `setTempReceiver` |
| `src/components/product/StartMessageButton.tsx:19-20` | Nav state setter | — | `setTempReceiver`, `setTempProductReason` |
| `src/components/navigation/BottomBar.tsx:31` | Badge | `newMessagesCount` | — |
| `src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx:27` | Active chat mgmt | — | `deleteChat`, `setActiveChatId` |
| `src/lib/store/authStore.ts:153` | Lifecycle | — | `clearChatStore` |

### Natural split mapping

If split into 4 stores, consumers would map cleanly:

**`useChatListStore`** (chats, hasMoreChats, lastVisibleChat, loadingMoreChats, newMessagesCount, subscribeToChats, loadMoreChats):
- `ChatsInit` → subscribeToChats
- `Chats` → chats, setActiveChatId (shared with active-chat store)
- `BottomBar` → newMessagesCount

**`useActiveChatStore`** (activeChatId, messages, hasMoreMessages, lastVisibleMessage, loadingMore, subscribeToMessages, loadMoreMessages, sendMessage, deleteChat, getActiveChat, setActiveChatId):
- `Messages` → all message reads + most actions
- `ChatUserFunctionsDialog` → deleteChat, setActiveChatId
- `messages.tsx` page → activeChatId

**`useChatNavStore`** (tempReceiver, tempProductReason, setTempReceiver, setTempProductReason, clearTempReceiver):
- `ProductUserDetails` → setTempReceiver
- `StartMessageButton` → setTempReceiver, setTempProductReason
- `Messages` → tempReceiver, tempProductReason (read for UI hints)

**`useChatUserCache`** (userCache, getUserData):
- Used internally by all stores above. Could be a shared utility rather than a Zustand store.

**Cross-store dependencies to watch during split:**
- `sendMessage` reads `tempReceiver` and `tempProductReason` — nav store dependency
- `getActiveChat` sets `activeChatId` and calls `subscribeToMessages` — active-chat store is self-contained here
- `deleteChat` touches `messages`, `hasMoreMessages`, `lastVisibleMessage`, `unsubMessages`, `activeChatId` — all within active-chat store

---

## Section 7 — ProductList scroll handler (F25)

### Current onScroll handler

**File:** `src/components/product/ProductList.tsx:243`

```tsx
onScroll={(e) => setShowScrollTop(e.nativeEvent.contentOffset.y > SCREEN_HEIGHT)}
```

**`scrollEventThrottle` prop:** ABSENT — confirmed. On iOS, this defaults to firing at 60fps.

**What happens on every scroll frame:**
1. Native scroll event fires (60fps on iOS)
2. `setShowScrollTop()` called → React state update
3. ProductList re-renders
4. FlatList re-evaluates `renderItem` (inline arrow → new reference)
5. Combined with no `React.memo` on `ProductCard`, all visible cards re-render

### Scroll-to-top button

**Lines 279–294:** Conditionally rendered based on `showScrollTop` state.

```tsx
{showScrollTop && (
  <Animated.View style={{ position: 'absolute', bottom: 20, right: 20 }}>
    <Pressable onPress={() => scrollRef.current?.scrollToOffset({ offset: 0, animated: true })}>
      <ArrowUp color="white" />
    </Pressable>
  </Animated.View>
)}
```

The button visibility is driven entirely by React state. Every frame of scrolling triggers a state update and re-render to evaluate `showScrollTop`. This would benefit from native-side state via Reanimated's `useAnimatedScrollHandler` + `useSharedValue` — the button visibility could be animated without JS thread involvement.

### Other ProductList re-render triggers

- `products` state (line 30): Array of products — changes on load, pagination, refresh
- `hasMore` state (line 32): Pagination flag
- `page` state (line 31): Current page number
- `refreshing` state (line 33): Pull-to-refresh
- `error` state (line 34): Error flag
- `shouldReload` from `useFavoritesStore` (line 45): Triggers refresh on favorite toggle
- `selectedLanguage` from `useAppContext()` (line 47): Language changes trigger key regeneration

### Other lists with scroll handlers

**`src/components/messages/Messages.tsx:211-212`** — the only other list with a scroll handler:

```tsx
onScroll={handleScroll}
scrollEventThrottle={16}
```

This DOES set `scrollEventThrottle={16}` (fires approximately once per frame at 60fps, but properly registered with the native scroll system). The handler sets `isNearBottom` state for auto-scroll-to-bottom behavior.

**All other FlatLists** use `onEndReached` for pagination (not per-frame) or have no scroll handler at all.

---

## Section 8 — Cross-cutting performance adjacencies

### F8.1 — `console.error` / `console.warn` calls across production code

**Severity:** low
**Files:** 24 `console.error`/`console.warn` calls found across production code:
- `src/lib/store/authStore.ts:88,105,118,134` — login/register error logging
- `src/lib/store/useChatStore.ts:305,580,630,691` — chat operations
- `src/lib/store/userPreferenceStorage.ts:13,27,36,46` — storage operations
- `src/lib/services/authService.ts:47,115` — image download, retry
- `src/lib/utils/serviceLog.ts:10,17` — intentional service logging utility
- `src/components/product/ProductList.tsx:90,183` — error states
- Various other components

**Why it matters:** `console.error` / `console.warn` are synchronous and cause performance overhead on every call. In production builds, they should be no-ops or routed through a structured logger. The `serviceLog.ts` calls are intentional logging, but the scattered `console.error` in stores and components are ad-hoc.

**Out of scope for Φ3 unless absorbed.** Existing `issues.md` entry (B16) already tracks this.

### F8.2 — `useMemo` / `useCallback` usage is sparse

**Severity:** medium
**Total `useMemo` + `useCallback` sites:** 56 across the entire codebase. For comparison, there are 115 `useEffect` sites.

Many components create new object/array/function references on every render and pass them as props. Without `React.memo` on children, this is currently harmless (children re-render anyway). But once `React.memo` is added in Φ3, unmemoized props will defeat the memoization. The Φ3 implementation must add `useMemo`/`useCallback` alongside `React.memo` for props passed to memoized list items.

### F8.3 — `ProductCard` has redundant `useEffect` for card size

**Severity:** low
**File:** `src/components/product/ProductCard.tsx:30-35`

```tsx
useEffect(() => {
  if (!currentPortalScope) return;
  setCardSize(getSizeForPortalScope(currentPortalScope));
}, [currentPortalScope, getSizeForPortalScope, portalSize, dashboardSize]);
```

This effect runs every time either `portalSize` or `dashboardSize` changes, even if the relevant size for the current scope hasn't changed. The `getSizeForPortalScope` function could be replaced with a direct selector: `useCardSizeStore(s => s.portalSize)` or `useCardSizeStore(s => s.dashboardSize)` based on `currentPortalScope`, eliminating the effect entirely.

### F8.4 — `ProductList` `listData` useMemo has volatile dependency

**Severity:** low
**File:** `src/components/product/ProductList.tsx:209`

```tsx
const listData = useMemo(() => { ... }, [products, hasMore, extraSections]);
```

`products` is a state array whose reference changes on every load/pagination. This is correct (the memo recomputes when products change). But `extraSections` is a prop — if the parent creates it inline, the memo recomputes on every parent render even if the sections haven't changed.

### F8.5 — `AppContextProvider` effect dependency includes `state.status`

**Severity:** medium (fixed in Φ2 close-out but worth noting for context)
**File:** `src/components/context/AppContext.tsx`

Per decisions.md 2026-05-27, the main bootstrap effect had `state.status` in its dependency array, which caused a double-bootstrap pattern. A `statusRef` was added so the polling interval can read current status without re-triggering the effect. This fix is already in place.

### F8.6 — `getUniqueID` imported from `react-native-markdown-display`

**Severity:** low
**File:** `src/components/product/ProductList.tsx:11`

A markdown rendering library is imported solely for generating unique IDs. This pulls the entire library into the bundle for one utility function. Replace with `crypto.randomUUID()` or a simple counter.

**Out of scope for Φ3.** Already tracked in structural audit out-of-scope observations.

### F8.7 — Dual store directory naming

**Severity:** low
**Files:** `src/lib/store/` (singular) vs `src/lib/stores/` (plural)

Confusing directory structure. `viewTokens.ts` lives in `stores/` while all other stores live in `store/`.

**Out of scope for Φ3.** Already tracked in structural audit out-of-scope observations (Ω scope).

---

## Section 9 — Messaging contract divergence vs web

### Divergence inventory

#### D9.1 — `tempProductReason` naming (behavioral)

**Mobile:** `tempProductReason` throughout (`useChatStore.ts:100,497,514,575,696`, `ChatStore.ts:17`, `Messages.tsx:37,111`)
**Web:** Renamed to `tempProductContext` per 2026-05-20 decision (Brief 2 W1).

**Store-split relevance:** None — naming only. Rename lands in the nav-state store regardless of split shape.

#### D9.2 — Non-atomic existing-chat send (STRUCTURAL)

**Mobile:** `useChatStore.ts:504-556` — three sequential writes:
1. `addDoc` to create message (line 518)
2. `getDoc` + `setDoc` to update sender's `userchats` sidecar (lines 524-543)
3. `setDoc` with merge to update receiver's `userchats` sidecar (lines 545-555)

**Web:** Uses `writeBatch()` for atomic batch write.

**Store-split relevance:** STRUCTURAL. The send path touches both the active-chat store (messages) and implicitly the chat-list store (sidecar updates affect chat ordering/unread counts). The batch should be atomic regardless of split boundary. The split should not break the batch into two stores' action methods — the send action belongs in one store (active-chat) and executes a single Firestore batch that touches both collections.

#### D9.3 — Hardcoded Serbian strings (behavioral)

6 hardcoded Serbian strings found:

| File | Line | String | Translation key needed |
|---|---|---|---|
| `useChatStore.ts` | 54 | `'Nove poruke ...'` | Image-only message preview |
| `Chats.tsx` | 32 | `'Pretraži ćaskanja...'` | Search placeholder |
| `Messages.tsx` | 103 | `'Izaberite ćaskanje kako bi videli poruke...'` | No-chat-selected message |
| `Messages.tsx` | 128 | `'Učitaj starije poruke'` | Load-more button |
| `Messages.tsx` | 220 | `'Ne mozete poslati poruku ovom korisniku zato sto ste blokirali korisnika...'` | Blocking notice |
| `Messages.tsx` | 228 | `'Ne mozete poslati poruku ovom korisniku zato sto vas je blokirao...'` | Blocked-by notice |

**Store-split relevance:** None — these are UI strings in components, not store state. Chat B fixes them in whichever component renders them.

#### D9.4 — `getActiveChat` reads wrong Firestore collection shape (behavioral)

**File:** `useChatStore.ts:719-720`

```tsx
withUserFirebaseUid: data.withUserFirebaseUid,  // doesn't exist on chats/{chatId}
withUser: data.withUser,                        // doesn't exist on chats/{chatId}
```

The fallback reads from `chats/{chatId}` which contains `{ users: [...], createdAt, lastMessage, lastUpdated }`. The `withUserFirebaseUid` and `withUser` fields exist on `userchats/{uid}/chats/{chatId}`, not on the chat root document.

**Store-split relevance:** None — bug is in the `getActiveChat` action, which lives entirely within the active-chat store. Fix doesn't influence boundary.

#### D9.5 — `loadingMore` stuck true after early return (behavioral)

**File:** `useChatStore.ts:313,316`

```tsx
set({ loadingMore: true });  // line 313
if (!user || !chatId || hasMoreMessages[chatId] === false) return;  // line 316 — no reset!
```

Early return without resetting `loadingMore: false`. UI shows permanent spinner.

**Store-split relevance:** None — `loadingMore` lives in the active-chat store. Bug fix is internal.

#### D9.6 — `onSnapshot` listeners (NOT a divergence)

The structural audit flagged F15 (listeners accumulate). Code review shows:
- `subscribeToChats` correctly unsubscribes old listener before creating new (lines 134-135)
- `subscribeToMessages` guards against duplicate subscriptions (line 230)
- Listeners for previously visited chats DO accumulate across a session (no unsubscribe on `setActiveChatId(null)`), but this is a design choice, not a bug. Each listener is guarded against duplicates.

**Store-split relevance:** STRUCTURAL. The listener lifecycle (subscribe/unsubscribe) is tightly coupled to which store owns the subscription. The chat-list store owns `unsubChats`; the active-chat store owns `unsubMessages`. The split must ensure `unsubscribeAll()` coordinates across both.

#### D9.7 — Push notification MESSAGE type ignores chat context (behavioral)

**File:** `src/notifications/components/PushNotificationsInit.tsx:76-77`

```tsx
case 'MESSAGE':
  router.push('/messages');  // no chatId extraction from notification data
```

**Store-split relevance:** None — this is a navigation handler, not a store concern.

#### D9.8 — Chat list pagination not wired (STRUCTURAL)

**`loadMoreChats()`** is defined in the store (lines 171-225) but **never called**. The FlatList in `Chats.tsx` has no `onEndReached` handler. Users with >15 chats cannot see older conversations.

**Store-split relevance:** STRUCTURAL. `loadMoreChats` and its state (`hasMoreChats`, `lastVisibleChat`, `loadingMoreChats`) belong in the chat-list store. The wiring gap is in the `Chats` component, which must call the chat-list store's `loadMoreChats` action.

#### D9.9 — `setActiveChatId(undefined)` type mismatch (behavioral)

**File:** `src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx:45`

```tsx
setActiveChatId(undefined);  // type expects string | null
```

**Store-split relevance:** None — trivial fix (`undefined` → `null`).

#### D9.10 — Message FlatList not inverted (behavioral)

**File:** `src/components/messages/Messages.tsx:203-236`

Messages are fetched with `orderBy('createdAt', 'desc')` and reversed with `.reverse()`. The FlatList is not inverted — manual `scrollToEnd` is used instead. Index-based `keyExtractor` at line 207: `keyExtractor={(_, i) => i.toString()}`.

**Store-split relevance:** None — UI concern in the Messages component. The message data shape is the same regardless of split.

#### D9.11 — Send failure not rethrown (behavioral)

**File:** `useChatStore.ts:579-592`

```tsx
} catch (e) {
  console.error('Send failed', e);
  // Rollback only, no rethrow
}
```

Web's contract (2026-05-20 decision): `sendMessage` rethrows after cleanup so the caller can show error UI. Mobile swallows the error — user sees optimistic message disappear with no feedback.

**Store-split relevance:** None — the rethrow pattern is internal to `sendMessage`, which lives in the active-chat store.

### Summary: structural vs behavioral

**Structural divergences** (influence store-split boundary):
- D9.2 — Non-atomic existing-chat send (batch must stay in one store)
- D9.6 — Listener lifecycle coordination across stores
- D9.8 — Chat list pagination wiring (loadMoreChats belongs in chat-list store)

**Behavioral divergences** (chat B fixes inside whichever store owns them):
- D9.1 — tempProductReason rename
- D9.3 — Hardcoded Serbian strings
- D9.4 — getActiveChat wrong collection
- D9.5 — loadingMore stuck true
- D9.7 — Push notification chat context
- D9.9 — setActiveChatId type mismatch
- D9.10 — Message list not inverted + index keys
- D9.11 — Send failure not rethrown

---

## End of audit
