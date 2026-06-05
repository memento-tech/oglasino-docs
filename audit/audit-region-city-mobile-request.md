# Audit — Region/City Mobile Search Request

**Repo:** oglasino-expo · **Branch:** new-expo-dev · **Mode:** READ-ONLY (no code changes)
**Date:** 2026-06-05
**Question:** Does mobile region/city product-search filtering have the SAME root cause as web
(wrong key + wrong shape), a DIFFERENT one, or is it fine?

## Verdict (one line)

**SAME class of bug as web — failure mode (b).** Mobile captures the selection correctly into
state, then copies it into the search request body under the WRONG key
(`selectedRegionsAndCities`) and in the WRONG shape (full `RegionDTO`/`CityDTO` OBJECTS), so the
backend's `selectedRegionAndCityValues = {regionIds, cityIds}` field is never populated and
region/city filtering silently does nothing. The string `selectedRegionAndCityValues` and the
keys `regionIds`/`cityIds` (in the wire sense) do **not exist anywhere** in mobile `src/`.

---

## Q1 — Where is the region/city filter UI, what does a tap write, and to which state field?

**UI:** `src/components/filters/RegionCityFilter.tsx` — a collapsible region/city picker built from
`selectedBaseSite.regions` (`:101`). Region row tap → `toggleRegion(region)` (`:34-52`); city row
tap → `toggleCity(region, city)` (`:54-85`).

**What a tap writes:** both handlers build a `next` object and call
`setRegionCityValues(next)` (`:51`, `:84`). Crucially, `next` holds **whole DTO objects**, not IDs:

```ts
// RegionCityFilter.tsx:38-48
let next = {
  regions: [...selectedRegionsAndCities.regions],   // RegionDTO[]
  cities:  [...selectedRegionsAndCities.cities],    // CityDTO[]
};
...
next.regions.push(region);   // pushes the FULL RegionDTO (id, labelKey, cities[])
...
next.cities.push(city);      // pushes the FULL CityDTO (id, labelKey)
```

**Which state field:** the Zustand filter store slice
`selectedRegionsAndCities: SelectedRegionsAndCities` (`src/lib/store/useFilterStore.ts:19`,
default `{ regions: [], cities: [] }` at `:56`), mutated by the `setRegionCityValues` action
(`useFilterStore.ts:29` decl, `:162` impl `set({ selectedRegionsAndCities: values })`).

This is the exact analogue of web's `selectedRegionsAndCities`. The stored value type is
`SelectedRegionsAndCities = { regions: RegionDTO[]; cities: CityDTO[] }`
(`src/lib/types/filter/SelectedRegionsAndCities.ts:4-7`), where
`RegionDTO = { id; labelKey; cities: CityDTO[] }` (`RegionDTO.ts:3-7`) and
`CityDTO = { id; labelKey }` (`CityDTO.ts:1-4`). **Objects, not entity-id arrays.**

## Q2 — Where is the search REQUEST BODY assembled, and how is region/city copied in?

**Assembler:** `src/components/product/FilteredProductList.tsx:73-99` — a `useMemo<ProductsFilterDTO>`
that builds the `productsFilter` object directly from store fields destructured at `:53-71`.

The region/city copy (`:93-96`):

```ts
selectedRegionsAndCities: {
  regions: selectedRegionsAndCities?.regions,   // RegionDTO[]  — full objects
  cities:  selectedRegionsAndCities?.cities,    // CityDTO[]    — full objects
},
```

It re-wraps the store value under the same key and ships the **full objects**.

**Grep results (brief-mandated):**
- `selectedRegionAndCityValues` → **ZERO hits** in `src/`. The backend's field is never produced.
- `regionIds` / `cityIds` (wire sense) → **ZERO hits**. (The only `cityIds` occurrences are
  function-local variables in `RegionCityFilter.tsx:35,45,47,78,79` used to drop a region's cities
  from local state — unrelated to the wire.)
- `selectedRegionsAndCities` (the WRONG key) → present in the store, the UI, the assembler, the
  type, the analytics helpers, and the deep-link parser (`filtersUtil.ts`).

**Wire path confirms no transform downstream:** `getProducts` POSTs the assembled object verbatim —
`BACKEND_API.post(uri, { productsFilter, paging })` (`src/lib/services/productsSearchService.ts:93`).
There is no mapper between the `useMemo` and the POST. What `FilteredProductList` builds is exactly
what goes on the wire.

## Q3 — Exact failure mode

**(b) — copied but WRONG key/shape, so the backend ignores it. Same class as web.**

- Not (a): region/city IS mapped into the request (`FilteredProductList.tsx:93-96`).
- Not (c): the selection DOES land in state (`setRegionCityValues` → `useFilterStore.ts:162`;
  verified by `useFilterStore.test.ts:45-46` asserting `regions`/`cities` ids populate).
- Not (d): mobile does **not** send `{regionIds, cityIds}` — that shape exists nowhere.

**Exact line where the value is lost:** `src/components/product/FilteredProductList.tsx:93-96`.
The store carries the right data (entity objects with `.id`), but the assembler emits it under
`selectedRegionsAndCities` as `{regions: RegionDTO[], cities: CityDTO[]}` instead of
`selectedRegionAndCityValues` as `{regionIds: number[], cityIds: number[]}`. The backend
deserializer finds no `selectedRegionAndCityValues` field → null → no filtering.

## Q4 — The request TYPE (smoking gun)

`src/lib/types/filter/ProductsFilterDTO.ts:20`:

```ts
selectedRegionsAndCities?: SelectedRegionsAndCities;   // { regions: RegionDTO[]; cities: CityDTO[] }
```

The type declares **only** the wrong-named, wrong-shaped field. There is **no**
`selectedRegionAndCityValues`, no `regionIds`, no `cityIds` on the request DTO. The type itself
encodes the bug — identical to web. Note this is NOT the `RegionAndCityDTO = {region, city}`
user-profile shape (the brief's named trap); it's the `{regions[], cities[]}` object-array shape,
the same wrong shape web ships. Either way it is objects, not the flat entity-id arrays the backend
reads.

## Q5 — Does mobile build the request directly from the store? (vs web's SSR round-trip)

**Confirmed: yes, directly from the store. No SSR / URL / rebuild round-trip.** Mobile is a native
app; `FilteredProductList.tsx:53-71` reads store fields via a `useShallow` selector and `:73-99`
assembles `productsFilter` in-process, then `:182` hands it to `fetchPage` → `getPortalProducts`
(`UserProductsList.tsx:31`, `FavoritesProductList.tsx`) → `getProducts` POST. There is no analogue
of web's SSR `filtersHelper`. **The fix point is the client-side assembler `useMemo`, not an SSR
helper.**

(Autocomplete is a separate, NON-region/city path: `SearchInput.tsx:90-96` assembles its own filter
from `searchText` + categories + server-`preparedFilters` only — it never reads
`selectedRegionsAndCities`, so it neither sends nor needs region/city. Not a fix site.)

## Q6 — Is the mobile fix the same shape as the web fix?

**Yes — the web fix pattern (reshape region/city to flat `{regionIds, cityIds}` of entity IDs at
the wire-facing layer) transfers directly.** Only the LOCATION differs: web reshapes in its SSR
`filtersHelper`; mobile has no SSR layer, so the reshape belongs at the client-side wire boundary.

**Recommended minimal fix location:** `src/components/product/FilteredProductList.tsx:93-96` — emit

```ts
selectedRegionAndCityValues: {
  regionIds: selectedRegionsAndCities?.regions?.map((r) => r.id) ?? [],
  cityIds:   selectedRegionsAndCities?.cities?.map((c) => c.id) ?? [],
},
```

plus rename/replace the field on `ProductsFilterDTO.ts:20` to
`selectedRegionAndCityValues?: { regionIds: number[]; cityIds: number[] }` (a small new wire type;
keep `SelectedRegionsAndCities` for the store/UI/deep-link layer, which legitimately needs the full
objects to render labels and checkboxes). **Do NOT change the store or the UI** — they correctly
hold objects; only the wire-facing projection is wrong.

**One wrinkle to flag for the implementing brief (not a blocker):** the same `filtersData` useMemo
feeds the analytics `filterChangeSignature(filtersData)` (`FilteredProductList.tsx:151`). Renaming
the field on the request DTO changes that object's shape; confirm `filterChangeSignature` still
keys correctly (it reads category presence, not the region/city values, but verify) and that
`getActiveFilterCount`/`getActiveFilterCategories` (which read `selectedRegionsAndCities` straight
from the store, not from `filtersData`) are untouched. Cleanest implementation reshapes at the wire
boundary only, leaving the analytics-facing projection on the store shape — the implementing
engineer should decide between (i) reshape inside the useMemo vs (ii) a tiny `toWireFilter()` mapper
applied in `getProducts`, so analytics isn't disturbed.

## Q7 (bonus) — Does mobile hydrate region/city from persisted state on nav?

Mobile hydrates region/city **from inbound deep-link query params**, not from a persisted-across-nav
store. `filtersUtil.ts` `parseRegionsAndCities` (`:217-237`) matches `?regions=`/`?cities=` slugs
against `baseSite.regions` and returns `{regions: RegionDTO[], cities: CityDTO[]}` (full objects),
applied via `hydrateFilters` → the same `selectedRegionsAndCities` store slice. So a deep link DOES
land region/city correctly in state — but because the SEND path (Q1–Q6) is broken, a deep-link
region/city filter is equally ignored by the backend. This is independent of bug #1 (logo/back
hydrate), which Igor reports works on mobile; the region/city defect is entirely in the send-path,
not the hydrate-path. Not deep-dived per brief.

---

## Bottom line for Mastermind

Mobile and web share one root cause: the request DTO and assembler speak `selectedRegionsAndCities`
= objects; the backend reads `selectedRegionAndCityValues` = flat entity-id arrays. Fixing it is the
web fix transplanted to mobile's client-side assembler (`FilteredProductList.tsx:93-96` +
`ProductsFilterDTO.ts:20`), store and UI untouched. No backend change needed (backend contract is
the target). A wire-shaped test (assert the POST body carries `selectedRegionAndCityValues.regionIds`
/`.cityIds`) would lock it; today no test asserts the outbound region/city wire shape — the existing
tests assert store state only.
