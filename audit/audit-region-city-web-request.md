# Audit — Region/City filter not reaching the search request (web)

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Date:** 2026-06-05
**Confirmed premise (not re-litigated):** backend observed `productsFilter.selectedRegionAndCityValues` arriving NULL even when region/city were selected; ES has `regionId`/`cityId` populated; backend predicate is correct when the field is non-null. So the defect is in WEB request-building. Backend reads `productsFilter.selectedRegionAndCityValues = { regionIds: number[], cityIds: number[] }` (flat entity-ID arrays). The `{region, city}` OBJECT shape (`RegionAndCityDTO`) is the user-profile shape, NOT the filter shape.

## Verdict (one line)

**Q3 = (b).** The selection IS captured in state and IS carried into the request body, but under the **wrong key AND wrong shape** — the web sends `productsFilter.selectedRegionsAndCities = { regions: RegionDTO[], cities: CityDTO[] }` (full objects), while the backend reads `productsFilter.selectedRegionAndCityValues = { regionIds: number[], cityIds: number[] }` (flat IDs). The keys never overlap, so the backend's field is NULL. The web request type has **no** `selectedRegionAndCityValues` / `regionIds` / `cityIds` field anywhere — that is the smoking gun.

---

## Q1 — Region/city filter UI: what selecting writes, and to where

**UI component:** `src/components/client/filters/RegionCityFilter.tsx` (component is `SideCityFilter`).
Both handlers build a `SelectedRegionsAndCities` object of **full DTO objects** and call `onFilterChange`:
- region select handler ≈ `RegionCityFilter.tsx:31-53`
- city select handler ≈ `RegionCityFilter.tsx:55-87`

**Wired to the store** in `src/components/client/filters/Filters.tsx`:
```tsx
// Filters.tsx:217-221
<RegionCityFilter
  selectedValues={selectedRegionsAndCities}
  onFilterChange={setRegionCityValues}   // :219
  regions={baseSite.regions || []}
/>
```

**Store field written:** `selectedRegionsAndCities` on the Zustand filter store.
`src/lib/store/useFilterStore.ts`:
```ts
selectedRegionsAndCities: SelectedRegionsAndCities;              // :17  (state type)
selectedRegionsAndCities: { regions: [], cities: [] },          // :56  (initial)
setRegionCityValues: (values) => set({ selectedRegionsAndCities: values }), // :146
```
`SelectedRegionsAndCities` is **objects, not IDs** — `src/lib/types/filter/SelectedRegionsAndCities.ts:4-7`:
```ts
export type SelectedRegionsAndCities = {
  regions: RegionDTO[];   // RegionDTO = { id, labelKey, cities }   (catalog/RegionDTO.ts:3-7)
  cities: CityDTO[];      // CityDTO   = { id, labelKey }           (catalog/CityDTO.ts:1-4)
};
```

**Conclusion Q1:** selection lands in state correctly. The store holds the full `RegionDTO`/`CityDTO` objects (it needs `labelKey` for the chips/URL and `id` for de-dup). This rules out Q3(c) — the UI handler is not broken.

---

## Q2 — Where the request body (`productsFilter`) is assembled, and whether region/city is mapped

There is **no client-side store→request assembler**. The live data path is **store → URL → SSR rebuild → request**:

1. **Store → URL** — `src/components/client/initializers/FilterManager.tsx` "SYNC TO URL" effect writes the selection out as **human-readable label** query params, then `router.replace`s:
   ```ts
   // FilterManager.tsx:270-282
   if (selectedRegionsAndCities.regions.length) {
     params.set('regions', selectedRegionsAndCities.regions.map((r) => toQueryParam(t(r.labelKey))).join(','));
   }
   if (selectedRegionsAndCities.cities.length) {
     params.set('cities', selectedRegionsAndCities.cities.map((c) => toQueryParam(t(c.labelKey))).join(','));
   }
   // ...
   router.replace(newUrl, { scroll: false });   // :321  → re-runs the server page
   ```
   Note: the URL carries **translated labels**, not IDs.

2. **URL → SSR `ProductsFilterDTO`** — the server page re-runs and hydrates filters via `filterHydrationSSR` → `parseFiltersFromQueryParams` → `getRegionAndCitiesData`:
   ```ts
   // src/lib/utils/filtersHelper.ts:53-61
   let result: ProductsFilterDTO = {
     ...
     selectedRegionsAndCities: getRegionAndCitiesData(params, baseSite, t),   // :58  ← WRONG KEY
     ...
   };
   ```
   ```ts
   // filtersHelper.ts:223-244 — getRegionAndCitiesData
   const regions = baseSite.regions?.filter((r) => regionValues.includes(toQueryParam(t(r.labelKey)))) || [];
   const cities  = baseSite.regions?.flatMap((r) => r.cities || [])
                     .filter((c) => cityValues.includes(toQueryParam(t(c.labelKey)))) || [];
   return { regions, cities };                                                // :240-243 ← WRONG SHAPE (objects, not IDs)
   ```
   `filterHydrationSSR` is the wrapper at `src/components/client/initializers/FilterHydrationSSRInit.tsx:7`; called by all four search pages (public home `(public)/page.tsx:41`, catalog `catalog/[[...slugs]]/page.tsx:114`, owner `owner/products/page.tsx:25`, admin `admin/products/page.tsx:24` + `admin/products/[userId]/page.tsx:28`).

3. **`ProductsFilterDTO` → POST** — `src/lib/service/nextCalls/productsSearchService.ts`:
   ```ts
   // productsSearchService.ts:79-91 — getProducts
   const res = await FETCH_BACKEND_API.post<ProductOverviewsDTO>(
     uri, { productsFilter, paging }, { headers: { 'X-Base-Site': tenant, 'X-Lang': lang } });
   ```
   `productsFilter` is shipped **verbatim** — no remap. `uri` is `/public/product/search` for portal (`:14-23`).

**Is there code that maps region/city into `selectedRegionAndCityValues.{regionIds, cityIds}`? — NO.** Grep across `src/` + `app/` for `selectedRegionAndCityValues`, `regionIds`, `cityIds` returns **zero hits**. The only thing built is `selectedRegionsAndCities` (objects).

---

## Q3 — Precise failure

**(b) — it IS copied into the request body but under the WRONG KEY and WRONG SHAPE.**

- The data round-trips correctly through state and the URL (so the UI chips and shareable URL work), and is reconstituted into `ProductsFilterDTO.selectedRegionsAndCities` as `{ regions: RegionDTO[], cities: CityDTO[] }`.
- The backend reads `productsFilter.selectedRegionAndCityValues.{regionIds, cityIds}`. That key is never set by the web, so it deserializes to NULL — exactly what Igor observed. (Every other filter works because each is sent under a key/shape the backend reads: `filters` → `{filterId, optionIds|rangeFrom/To}`, `priceRange`, `orderBy`, `productStates`, `moderationStates`.)

**Exact line where the value is lost (becomes unreadable to the backend):** the value is "lost" at the wire boundary by being emitted under the wrong field. The pin points, in order:
- **Smoking gun (type):** `src/lib/types/filter/ProductsFilterDTO.ts:20` declares `selectedRegionsAndCities?: SelectedRegionsAndCities;` — there is no `selectedRegionAndCityValues` field, so the correct key can never be populated.
- **Construction of the wrong shape:** `src/lib/utils/filtersHelper.ts:240-243` (returns `{regions, cities}` objects), assigned under the wrong key at `filtersHelper.ts:58`.
- **Shipped verbatim:** `src/lib/service/nextCalls/productsSearchService.ts:79-91`.

This is NOT Q3(a) (it IS copied) and NOT Q3(c) (state capture works — `useFilterStore.ts:146`).

---

## Q4 — Correct target shape vs. the web's declared request type

**Backend expects:**
```ts
productsFilter.selectedRegionAndCityValues = { regionIds: number[]; cityIds: number[] }
```

**Web's declared request type — cross-check (`ProductsFilterDTO.ts:8-23`):**
```ts
export interface ProductsFilterDTO {
  ...
  selectedRegionsAndCities?: SelectedRegionsAndCities;   // :20  — { regions: RegionDTO[]; cities: CityDTO[] }
  ...
}
```
The web request type has **no** `selectedRegionAndCityValues`, **no** `regionIds`, **no** `cityIds` (grep-confirmed: zero occurrences in `src/` or `app/`). The type itself lacks the field the backend reads — this is the smoking gun confirming Q3(b): the only region/city field on the wire DTO is the wrong-named, wrong-shaped one.

The required IDs are readily available — both `RegionDTO` (`catalog/RegionDTO.ts:3`) and `CityDTO` (`catalog/CityDTO.ts:2`) carry `id: number`.

---

## Q5 — Minimal fix (recommendation only; NOT implemented)

**This is a reshape of existing wrong code, not a purely additive map** — the field exists but is wrong on both key and shape. The reshape is small and contained to the **wire-facing layer**; the store, UI, and URL keep `selectedRegionsAndCities` objects (they need `labelKey` for chips/URL labels and `id` for de-dup).

Three touch points:

1. **`src/lib/types/filter/ProductsFilterDTO.ts:20`** — replace
   `selectedRegionsAndCities?: SelectedRegionsAndCities;`
   with `selectedRegionAndCityValues?: { regionIds: number[]; cityIds: number[] };`
   (a small dedicated type, e.g. `SelectedRegionAndCityValues`, is cleaner than inline). Remove the now-unused `SelectedRegionsAndCities` import from this file if nothing else needs it.

2. **`src/lib/utils/filtersHelper.ts`** — change `getRegionAndCitiesData` (`:223-244`) to return
   `{ regionIds: regions.map((r) => r.id), cityIds: cities.map((c) => c.id) }`
   (resolve region/city from `baseSite.regions` by label exactly as today, then emit `.id`), and update the assignment at `:58` to the new key `selectedRegionAndCityValues`.

3. **`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:218-219`** — this reads the SSR-built `filtersData.selectedRegionsAndCities?.regions.length / .cities.length` to decide a UI flag; update it to the new shape (`filtersData.selectedRegionAndCityValues?.regionIds.length / .cityIds.length`).

**Out of scope of the fix (leave as-is):** the store (`useFilterStore.ts`), `RegionCityFilter.tsx`, `Filters.tsx`, `SelectedFiltersDisplay.tsx`, and the URL-sync/hydrate halves of `FilterManager.tsx` all legitimately use the object-shaped `selectedRegionsAndCities` for rendering and URL round-tripping — none of those is the wire DTO. Only the three sites above face the backend.

**Scope note:** the same `ProductsFilterDTO` + `getProducts` path serves portal/owner/admin search (`productSearchUri`, `productsSearchService.ts:14-23`), so this single fix corrects region/city filtering across all three surfaces simultaneously.

---

## Trust boundary (conventions Part 11)

`regionId`/`cityId` are foreign keys the server validates against its own catalog data; sending the entity IDs the client picked from the server-provided `baseSite.regions` is consistent with Part 11 (a value the client cannot misrepresent without the server rejecting it). No client-supplied "before" values involved. No trust-boundary concern introduced by the recommended fix.
