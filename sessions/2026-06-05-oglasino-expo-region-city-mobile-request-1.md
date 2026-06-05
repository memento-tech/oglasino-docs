# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-05
**Task:** READ-ONLY audit — confirm whether mobile region/city product-search filtering has the same wire-contract root cause as web, a different one, or is fine. Write findings to `.agent/audit-region-city-mobile-request.md`.

## Implemented

- No code changes (read-only audit per brief).
- Produced `.agent/audit-region-city-mobile-request.md` answering Q1–Q7 with file:line evidence.
- **Verdict:** SAME class of bug as web (failure mode b). Mobile captures region/city into state correctly but ships it in the request under the WRONG key `selectedRegionsAndCities` as full `RegionDTO`/`CityDTO` OBJECTS, instead of the backend's `selectedRegionAndCityValues = {regionIds, cityIds}` flat entity-id arrays. The strings `selectedRegionAndCityValues`/`regionIds`/`cityIds` (wire sense) appear nowhere in `src/`.
- **Loss point pinned:** `src/components/product/FilteredProductList.tsx:93-96` (the request assembler `useMemo`). No transform downstream — `productsSearchService.ts:93` POSTs it verbatim.
- **Fix shape:** web fix transplants directly (reshape to flat IDs); location differs (mobile has no SSR layer — fix at the client-side assembler + `ProductsFilterDTO.ts:20` type, leaving store/UI/deep-link object shapes intact).

## Files touched

- `.agent/audit-region-city-mobile-request.md` (new, audit deliverable)
- `.agent/2026-06-05-oglasino-expo-region-city-mobile-request-1.md` (this summary)
- `.agent/last-session.md` (exact copy)
- No `src/` files modified.

## Tests

- Not run — read-only audit, no code touched. (Brief: "No fixes. Part 4a N/A (read-only).")

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change. This is a Phase-2 audit deliverable; no feature is being adopted or removed from the Expo backlog table by this session. (Mastermind/Docs may later add a `region-city-mobile-request` row when the fix brief is queued — not this session's call.)
- issues.md: no change authored by me. Note: the open 2026-06-04 region/city web wire-contract issue has a mobile sibling now confirmed by this audit; whether to log a mobile issues.md entry is Mastermind's call (drafted note in "For Mastermind").

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed, no debug logging, no dead code introduced.
- Part 4a (simplicity): N/A (read-only, no code added) — see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind" (no outbound-wire test for region/city).
- Part 6 (translations): N/A this session.
- Part 8 (architectural defaults) — confirmed relevant: the fix is client-side reshape only; no new mobile-specific backend route (mobile reuses `/public/product/search`), consistent with Part 8 route-reuse.
- Part 11 (trust boundaries): N/A — region/city IDs are client-supplied search filters validated server-side against the base-site catalog, not moderation/auth inputs.

## Known gaps / TODOs

- none. Q7 deliberately not deep-dived per brief (send-path was the priority and is fully answered).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Confirmed: mobile == web root cause.** Wrong key + wrong shape at the request DTO/assembler. The fix is the web fix transplanted to `FilteredProductList.tsx:93-96` + `ProductsFilterDTO.ts:20`. Store, UI (`RegionCityFilter.tsx`), and deep-link parser (`filtersUtil.ts`) correctly hold full objects and should NOT change — only the wire-facing projection.
- **Implementation wrinkle (flag for the fix brief):** the assembler `useMemo` also feeds analytics (`filterChangeSignature`, `FilteredProductList.tsx:151`). Recommend reshaping at the wire boundary (a small `toWireFilter()` in `getProducts`, or inside the useMemo with care) so the analytics signature isn't disturbed. `getActiveFilterCount`/`getActiveFilterCategories` read region/city straight from the store, not from `filtersData`, so they're unaffected either way.
- **Part 4b observation (low):** no test asserts the OUTBOUND region/city wire shape — existing tests (`useFilterStore.test.ts`, `filtersUtil.test.ts`) assert store state only. The bug survived because nothing checks what actually goes on the wire. A wire-shape assertion (POST body carries `selectedRegionAndCityValues.regionIds`/`.cityIds`) would catch this class. `src/components/product/FilteredProductList.tsx` / `src/lib/services/productsSearchService.ts`. Did not fix — out of scope (read-only audit). Suggest the fix brief add it.
- **Possible issues.md entry (Mastermind's call, drafted — I do not write config files):**
  > **2026-06-05 — Mobile: region/city search filter never reaches the backend (wrong wire key+shape, same root cause as web)** · Repo: `oglasino-expo` · Severity: medium · Status: open. Mobile assembles the product-search body with `selectedRegionsAndCities: {regions: RegionDTO[], cities: CityDTO[]}` (`FilteredProductList.tsx:93-96`) and declares only that field on `ProductsFilterDTO` (`ProductsFilterDTO.ts:20`); the backend reads `selectedRegionAndCityValues: {regionIds, cityIds}`. So region/city filtering silently does nothing on mobile (search and deep-link both affected; selection lands in state fine). Fix: reshape to flat entity-id arrays at the client wire boundary; store/UI/deep-link object shapes unchanged. Sibling of the 2026-06-04 web region/city wire-contract issue. See `.agent/audit-region-city-mobile-request.md`.
