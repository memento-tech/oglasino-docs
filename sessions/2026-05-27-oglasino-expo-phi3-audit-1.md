# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-27
**Task:** Inventory the Φ3 performance surface in `oglasino-expo`. Read-only audit. No code changes.

## Implemented

- Produced `.agent/audit-expo-phi3-performance.md` with all nine sections populated:
  - **Section 1 (F7 — Zustand subscriptions):** 10 stores inventoried, 99 subscription sites mapped (up from structural audit's "22+"). 7 high-risk whole-store destructuring patterns identified. Zero `useShallow` usage. Most critical: Messages.tsx destructures 11 fields from useChatStore, BottomBar subscribes to 4 stores.
  - **Section 2 (F8 — React.memo / renderItem):** Zero `React.memo` confirmed. 11 FlatList instances found, ALL use inline arrow `renderItem` (none use `useCallback`). 5 additional ScrollView-with-mapped-children sites. ProductCard and ProductReview subscribe to stores independently inside lists.
  - **Section 3 (F9 — Image usage):** 16 Image render sites across 9+ files. Zero expo-image imports (package installed as `~3.0.11`). Zero advanced caching props anywhere. Zero Image static method usage. 1 ImageBackground usage. Key migration targets: ProductTopImage (product feed), ImagesCarousel (FlatList), OglasinoAvatar (chat lists).
  - **Section 4 (F10 — AppContext):** 26 `useAppContext()` consumers. Provider value is a new object every render. 3 of 4 action functions lack `useCallback`. 5-second maintenance poll can cascade re-renders to all 26 consumers. 22 of 26 consumers read only `selectedBaseSite`.
  - **Section 5 (F20 — CategorySelector / CitySelector):** Both use ScrollView with no virtualization. CategorySelector: ~375 items when expanded. CitySelector: ~750 items all rendered at mount — highest risk in product creation flow. SectionList is the natural replacement.
  - **Section 6 (F23 — useChatStore):** 746-line monolithic store. 9 consumer files mapped. Natural 4-way split: chat-list, active-chat, nav-state, user-cache. `loadingMore` confirmed as single global boolean. `userCache` grows unboundedly. Three structural divergences identified that influence split boundary.
  - **Section 7 (F25 — ProductList scroll):** `scrollEventThrottle` absent — confirmed. `onScroll` fires `setState` every frame (60fps on iOS). Scroll-to-top button driven by React state. Only other scroll handler (Messages.tsx) correctly uses `scrollEventThrottle={16}`.
  - **Section 8 (cross-cutting):** 7 adjacencies flagged: console.error/warn in production, sparse useMemo/useCallback (56 total vs 115 useEffect sites), ProductCard redundant effect, getUniqueID from markdown library, dual store directory naming, AppContext effect-dep fix (already applied in Φ2), ProductList listData volatile dependency.
  - **Section 9 (messaging divergence):** 11 divergences catalogued. 3 structural (non-atomic send, listener lifecycle coordination, pagination wiring), 8 behavioral (naming, hardcoded strings, wrong collection shape, stuck loading, push context, type mismatch, inverted list, send error swallowed).

## Files touched

- `.agent/audit-expo-phi3-performance.md` (new, +~450 lines)

## Tests

- N/A — read-only audit, no code changes.

## Cleanup performed

- None needed (read-only audit).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing. This audit produces new findings; it does not obsolete prior work.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code changes.
- Part 4a (simplicity): N/A — no code added, no abstractions introduced, nothing simplified. See structured evidence in "For Mastermind."
- Part 4b (adjacent observations): confirmed — 7 adjacencies flagged in Section 8 of the audit and repeated in "For Mastermind" below.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 7 (error contract) — observed in Section 9 (messaging divergence D9.11 re: send failure not rethrown); no new violations introduced.

## Known gaps / TODOs

- The subscription site count (99) was derived from automated search and manual audit. A few sites in deeply nested filter components (passed via prop as `useFilterStore`) may have been counted as one site per prop-pass rather than per ultimate usage. The order of magnitude (90–110) is correct.
- Runtime verification of actual re-render impact is out of scope (Ψ scope). All severity assessments are from code reading.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit
  - Considered and rejected: nothing — read-only audit
  - Simplified or removed: nothing — read-only audit

- **Part 4b adjacencies (from Section 8):**
  1. `console.error`/`console.warn` calls across production code (24 sites) — low severity — already tracked as B16 in structural audit. Out of scope for Φ3.
  2. Sparse `useMemo`/`useCallback` usage (56 total vs 115 `useEffect` sites) — medium severity — will become load-bearing once `React.memo` is added in Φ3. Φ3 briefs must add `useMemo`/`useCallback` alongside `React.memo` for props passed to memoized list items.
  3. `ProductCard` has redundant `useEffect` for card size (`ProductCard.tsx:30-35`) — low severity — can be replaced with a direct store selector, eliminating the effect.
  4. `getUniqueID` imported from `react-native-markdown-display` (`ProductList.tsx:11`) — low severity — already tracked in structural audit. Ω scope.
  5. Dual store directory naming (`src/lib/store/` vs `src/lib/stores/`) — low severity — already tracked. Ω scope.
  6. `ProductList` `listData` useMemo has volatile `extraSections` dependency — low severity — if parent creates `extraSections` inline, the memo recomputes on every parent render.
  7. `AppContext` effect-dep fix (status in dependency array causing double-bootstrap) — already fixed in Φ2 close-out per decisions.md 2026-05-27. Noted for completeness.

- **Key findings for Φ3 spec drafting:**
  - The subscription map (Section 1) is larger than expected: 99 sites, not 22+. The Φ3 selector refactoring scope is correspondingly larger.
  - The natural chat store split (Section 6) maps cleanly to 4 stores. Three structural messaging divergences (D9.2, D9.6, D9.8) must be considered during the split: the atomic-batch send must stay in one store, listener lifecycle must coordinate across stores, and chat-list pagination must be wired.
  - CitySelector (Section 5) is the highest-risk ScrollView site at ~750 items. CategorySelector is lower risk (~375 items, hidden behind expand/collapse). SectionList is the natural replacement for CitySelector.
  - The `useMemo`/`useCallback` sparsity (adjacency #2) means Φ3 cannot just add `React.memo` — it must also stabilize props and callbacks. Otherwise memoization is defeated by new references on every render.
  - 22 of 26 AppContext consumers read only `selectedBaseSite`. A context split (rarely-changing data vs actions) would eliminate most unnecessary re-renders.

- **Questions for Mastermind:**
  - Should the CitySelector virtualization (ScrollView → SectionList) be absorbed into Φ3, or deferred to chat A (which touches product creation UI)? The brief lists F20 in Φ3 scope, but the fix is small and self-contained.
  - The messaging divergences in Section 9 are documented for the store-split decision. Should the split shape be locked by Mastermind before Φ3 briefs begin, or should the Φ3 engineer propose the split shape in their brief?
