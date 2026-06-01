# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-27
**Task:** Convert every non-chat-store Zustand subscription site to single-field selectors, `useShallow` multi-field selectors, or `getState()` calls (Φ3 Brief 2, F7 Zustand selector refactor).

## Implemented

- Converted ~56 subscription sites across 8 stores (`useAuthStore`, `useFavoritesStore`, `useNotificationStore`, `useCardSizeStore`, `usePortalFilterStore`/`useDashboardFilterStore`, `useDialogStore`, `useChatBlockStore`, `useViewTokenStore`) from whole-store destructures to correct selector shapes. ~20 sites were already correct (Φ1 introduced proper selectors). `useChatStore` consumers (~29 sites) and `ProductCard.tsx` explicitly excluded per brief.
- **F7.2 — BottomBar.tsx**: Four stores converted. `useAuthStore` → two single-field selectors (`user`, `_hasHydrated`). `useFavoritesStore` → single-field selector (`favoriteIds`). `useNotificationStore` → single-field selector (`unseenCount`). `useChatStore` left as-is (Brief 9). `useDialogStore` was already a selector.
- **F7.3 — Filter store consumers**: `Filters.tsx`, `FiltersDialog.tsx`, `FilteredProductList.tsx`, `FloatingFiltersButton.tsx`, `SelectedFiltersDisplay.tsx`, `UserProductsList.tsx` converted to `useShallow` multi-field selectors for state fields. Actions converted to individual selectors (stable refs in Zustand v5, effectively free subscriptions). Smaller filter sub-components (`CurrencySelector`, `PriceFilter`, `ProductOrder`, `RegionCityFilter`, `SearchInput`, `CurrencySelectionDialog`) converted to individual single-field selectors.
- **F7.5 — FavoriteButton.tsx**: `favoriteIds` selector refined to `s => s.favoriteIds.includes(productData.id)` — returns a derived boolean, so the component only re-renders when THIS product's favorite status changes, not when any favorite is toggled. `addRemoveFavorite` and `presetClearShouldReload` moved to `getState()` in the click handler.
- **Auth dialogs** (`RegisterDialog`, `LoginOptionsDialog`, `LoginDialog`): Replaced whole-store destructures with `loading`/`error`/`user` selectors. Actions (`register`, `login`, `loginWithGoogle`, `loginWithFacebook`) moved to `getState()` in callbacks. `isAuthenticated()` calls replaced with direct `user` checks.
- **useChatBlockStore consumers** (`Messages.tsx`, `ProductUserDetails.tsx`, `ChatUserFunctionsDialog.tsx`): Subscribed to `blocking`/`blockedBy` data maps via `useShallow` or direct selectors, and computed boolean checks locally (`!!blockingMap[uid]`). This preserves re-render behavior when block status changes (the functions `isBlockedBy`/`isBlocking` use `get()` internally and wouldn't trigger re-renders if selected by reference). Actions moved to `getState()` in callbacks.

## Files touched

- src/components/navigation/BottomBar.tsx (+4 / -3)
- src/components/product/FavoriteButton.tsx (+8 / -11)
- src/components/navigation/Filters.tsx (+12 / -7)
- src/components/dialog/dialogs/FiltersDialog.tsx (+14 / -9)
- src/components/product/FilteredProductList.tsx (+10 / -7)
- src/components/FloatingFiltersButton.tsx (+5 / -2)
- src/components/filters/SelectedFiltersDisplay.tsx (+14 / -12)
- src/components/product/UserProductsList.tsx (+7 / -4)
- src/components/product/ProductList.tsx (+2 / -1)
- src/components/product/HorizontalExtraProductsListView.tsx (+1 / -1)
- src/components/filters/CurrencySelector.tsx (+2 / -1)
- src/components/filters/PriceFilter.tsx (+2 / -1)
- src/components/filters/ProductOrder.tsx (+2 / -1)
- src/components/filters/RegionCityFilter.tsx (+2 / -1)
- src/components/SearchInput.tsx (+1 / -1)
- src/components/dialog/dialogs/CurrencySelectionDialog.tsx (+2 / -1)
- src/components/dialog/DialogManager.tsx (+5 / -1)
- src/components/dialog/dialogs/PortalConfigDialog.tsx (+3 / -3)
- src/components/dialog/dialogs/CardSelectionDialog.tsx (+2 / -1)
- src/components/init/CardSizeInit.tsx (+1 / -2)
- src/components/messages/Messages.tsx (+8 / -4)
- src/components/product/ProductUserDetails.tsx (+10 / -5)
- src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx (+8 / -5)
- src/components/navigation/TopBar.tsx (+1 / -1)
- src/components/user/UserMenu.tsx (+3 / -2)
- src/components/dialog/dialogs/RegisterDialog.tsx (+5 / -4)
- src/components/dialog/dialogs/LoginOptionsDialog.tsx (+5 / -3)
- src/components/dialog/dialogs/LoginDialog.tsx (+4 / -3)
- src/notifications/components/NotificationsInit.tsx (+4 / -5)
- app/(portal)/(secured)/notifications.tsx (+8 / -3)
- src/components/ReportButton.tsx (+1 / -1)
- src/components/ConsumerProtectionBanner.tsx (+1 / -1)
- src/components/FollowUserButton.tsx (+1 / -1)
- src/components/product/ProductReview.tsx (+1 / -1)
- src/components/product/ProductFavoriteButton.tsx (+1 / -1)
- src/components/product/CallUserButton.tsx (+1 / -1)
- src/components/product/ProductReviewButton.tsx (+1 / -1)
- src/components/product/FavoritesProductList.tsx (+1 / -1)
- src/components/dialog/dialogs/PreviewProductDialog.tsx (+1 / -1)
- src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx (+1 / -1)
- src/components/dashboard/layout/DashboardSidebar.tsx (+1 / -1)
- app/owner/dashboard/user.tsx (+1 / -1)
- app/(portal)/(public)/blog/free-zone.tsx (+1 / -1)
- app/(portal)/(public)/about.tsx (+1 / -1)

## Tests

- Ran: `npx tsc --noEmit` — exit 0, zero errors
- Ran: `npm run lint` — 0 errors, 75 warnings (down from Φ2 baseline ~80)
- Ran: `npm test` (vitest) — 109 passed, 0 failed (matches Φ2 baseline)
- Ran: `npx expo-doctor` — same pre-existing 1 check failed (8 packages out of date), no new issue

## Cleanup performed

None needed. No commented-out code, no unused imports introduced, no console.log added.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing. All subscription shapes replaced in place — no files deleted, no dead code created.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports, no console.log, no TODOs added.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): confirmed — two observations noted below.
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): `useShallow` import added to 8 files that needed multi-field selectors. This is the smallest correct shape for components reading 3+ state fields from a store. No new abstractions, wrappers, or configuration introduced.
  - Considered and rejected: (1) A shared `useAuthUser()` custom hook wrapping `useAuthStore(s => s.user)` for the ~20 components that just read `user` — rejected because a one-line selector is already the simplest form, and the hook would add indirection for no reduction in code. (2) Putting all filter actions inside `useShallow` alongside state fields — rejected because actions are stable references in Zustand v5 and don't trigger re-renders; separating them makes the subscription surface explicit.
  - Simplified or removed: ~56 whole-store destructures narrowed to precise selectors, eliminating re-render propagation from unrelated store mutations (representative: `BottomBar` no longer re-renders on auth loading toggle; `FavoriteButton` no longer re-renders when a different product is favorited).

- **Part 4b adjacent observations:**
  1. `Filters.tsx:86-90` and `FiltersDialog.tsx:87-91` call `addRemoveOptionFilter()` during render (outside useEffect or callback). This is a React anti-pattern (setState during render). File: `src/components/navigation/Filters.tsx:86`. Severity: medium. I did not fix this because it is out of scope (pre-existing, behavior preservation required).
  2. `src/components/messages/Messages.tsx:142` — `<div>` element in RN code at `Filters.tsx:143` (`<div key={filter.id}>Filter not found {filter.filterKey}</div>`). Should be `<View>` + `<Text>`. File: `src/components/navigation/Filters.tsx:143`. Severity: low. I did not fix this because it is out of scope (Ω scope per structural audit).

- **Site count:** ~56 sites touched, ~20 already correct (Φ1 selectors, getState() calls), ~29 excluded (useChatStore — Brief 9), `ProductCard.tsx` excluded (Brief 3). Total inventory aligned with audit's ~99 count within expected variance (some Φ1/Φ2 additions, prop-passed filter stores counted as individual sites).

- **FavoriteButton derived selector verification:** Confirmed `favoriteIds` is `number[]` (line 6 of `useFavoritesStore.ts`). `Array.includes()` returns a boolean. Zustand's default `Object.is` comparison on the selector output means: when `favoriteIds` changes but `includes(productId)` returns the same boolean (e.g., a different product was toggled), the component does NOT re-render. When the same product's status changes, the boolean flips and the component re-renders correctly. Behavior preserved.

- **useChatBlockStore shape deviation from audit:** The audit listed `isBlockedBy` and `isBlocking` as the consumer-facing fields. The brief's recommended shape (single-field selectors on functions) would prevent re-renders when block data changes, because the function references are stable — they use `get()` internally. Instead, I subscribed to the underlying `blocking`/`blockedBy` data maps and computed the booleans locally. This preserves behavior while narrowing the subscription surface (no longer re-renders on `unsub` field changes).

- **Closure gate:** No config-file dependency introduced. This session does not adopt a feature from the Expo backlog, so no `state.md` edit is needed.
