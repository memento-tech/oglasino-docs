# Audit — web region/city wire shape

**Repo:** oglasino-web · **Branch:** dev (confirmed checked out) · **Type:** read-only, no code changed.
**Date:** 2026-06-01
**Question:** For a PORTAL catalog search with ≥1 region AND ≥1 city selected, what exact JSON does
web POST to `/public/product/search` for the region/city portion of the body?

---

## TL;DR

Web puts **full objects** on the wire under the outer key **`selectedRegionsAndCities`**, with inner
arrays **`regions`** and **`cities`**. It does **not** send `selectedRegionAndCityValues`, and it does
**not** send `regionIds` / `cityIds` number arrays. There is **no transform step** anywhere between the
UI/store objects and the POST body — the objects are serialized as-is. The web TypeScript type and the
serialized wire body **agree with each other**; both **disagree** with the backend read's claim of
`selectedRegionAndCityValues.{regionIds,cityIds}`. That divergence is the parity finding.

---

## 1. Outer key name

**`selectedRegionsAndCities`** (plural "Regions", "And", "Cities").

- Declared on the request DTO: `src/lib/types/filter/ProductsFilterDTO.ts:20`
  → `selectedRegionsAndCities?: SelectedRegionsAndCities;`
- It is **not** `selectedRegionAndCityValues`. A repo grep for `selectedRegionAndCityValues` returns
  zero hits in web source.

## 2. Inner shape — full objects, not ID arrays

Inner type `SelectedRegionsAndCities` (`src/lib/types/filter/SelectedRegionsAndCities.ts:4-7`):

```ts
type SelectedRegionsAndCities = {
  regions: RegionDTO[];
  cities:  CityDTO[];
};
```

The element objects carried on the wire (no field stripping happens before POST — see §3):

- `RegionDTO` (`src/lib/types/catalog/RegionDTO.ts`):
  `{ id: number; labelKey: string; cities: CityDTO[] }`
  — note each region object **also carries its own nested `cities` array** on the wire.
- `CityDTO` (`src/lib/types/catalog/CityDTO.ts`):
  `{ id: number; labelKey: string }`

So for one region + one city selected, the region/city portion of the POST body is approximately:

```json
{
  "selectedRegionsAndCities": {
    "regions": [
      {
        "id": 12,
        "labelKey": "region.podgorica.label",
        "cities": [
          { "id": 30, "labelKey": "city.podgorica.label" },
          { "id": 31, "labelKey": "city.tuzi.label" }
        ]
      }
    ],
    "cities": [
      { "id": 30, "labelKey": "city.podgorica.label" }
    ]
  }
}
```

The inner arrays are arrays of **objects** carrying `id` + `labelKey` (regions additionally carry a nested
full `cities[]`). They are **not** `regionIds` / `cityIds` number arrays.

## 3. Where the mapping from UI objects to the posted shape happens

**Nowhere — there is no object→ID mapping step.** The store/SSR `{regions,cities}` objects are sent as-is.

Build (server-side, the canonical hydration path the brief named):

- `filterHydrationSSR` (`src/components/client/initializers/FilterHydrationSSRInit.tsx:7-25`) flattens
  `searchParams` to key/value pairs and delegates to `parseFiltersFromQueryParams`.
- `parseFiltersFromQueryParams` (`src/lib/utils/filtersHelper.ts:38`) sets
  `selectedRegionsAndCities: getRegionAndCitiesData(params, baseSite, t)` at **`filtersHelper.ts:58`**.
- `getRegionAndCitiesData` (`src/lib/utils/filtersHelper.ts:223-244`) resolves the URL `regions=` /
  `cities=` slug values back to the **full `RegionDTO` / `CityDTO` objects** off `baseSite.regions`
  (matched by `toQueryParam(t(labelKey))`), and returns `{ regions, cities }` — objects, not ids.

POST (both transport variants — identical region/city handling):

- **nextCalls (SSR / server action, first page):** `getProducts`
  (`src/lib/service/nextCalls/productsSearchService.ts:87-92`) POSTs the body
  `{ productsFilter, paging }`. `productsFilter` is the `ProductsFilterDTO` straight from hydration —
  **passed through verbatim, no reshape.**
- **reactCalls (client pagination, page ≥ 1):** `SelectableFilterProductListWrapper.tsx:56` calls
  `getPortalProducts({ ...filtersData }, paging, signal)` — a **shallow spread only**; the
  `selectedRegionsAndCities` object reference is carried unchanged. reactCalls `getProducts`
  (`src/lib/service/reactCalls/productsSearchService.ts:142`) then POSTs `{ productsFilter, paging }`.
  Same outer key, same inner objects. The two variants do **not** differ for region/city.

(The autocomplete path, reactCalls `getAutocompleteSuggestions`
`productsSearchService.ts:88-94`, uses the same `{ productsFilter, paging }` shape — also no transform.)

---

## Type-vs-wire verdict (the brief's explicit ask)

The web TS `ProductsFilterDTO.selectedRegionsAndCities` and the actually-serialized body **match** — web
genuinely sends `selectedRegionsAndCities: { regions: <objects>, cities: <objects> }`. There is no
rename/reshape between the type and the wire. So the divergence is **not** web-type-vs-web-wire; it is
**web (type and wire, both objects under `selectedRegionsAndCities`) vs. the backend read's claim
(`selectedRegionAndCityValues` with `regionIds`/`cityIds` `List<Long>`)**. Since web region/city filtering
works in production, the backend must be consuming the object shape under `selectedRegionsAndCities` (e.g.
deserializing the objects and reading their `id` fields), or the backend read was of a different/renamed
DTO than the one this endpoint actually binds. Resolving which is a backend-side question, outside this
read-only web audit.

---

## Files inspected (read-only)

- `src/lib/types/filter/ProductsFilterDTO.ts`
- `src/lib/types/filter/SelectedRegionsAndCities.ts`
- `src/lib/types/catalog/RegionDTO.ts`, `src/lib/types/catalog/CityDTO.ts`
- `src/lib/utils/filtersHelper.ts` (`parseFiltersFromQueryParams`, `getRegionAndCitiesData`)
- `src/components/client/initializers/FilterHydrationSSRInit.tsx` (`filterHydrationSSR`)
- `src/lib/service/nextCalls/productsSearchService.ts` (`getProducts` / `getPortalProducts`)
- `src/lib/service/reactCalls/productsSearchService.ts` (`getProducts` / `getPortalProducts` / autocomplete)
- `src/components/client/product/SelectableFilterProductListWrapper.tsx` (pagination call sites)
- `src/components/client/product/FilteredProductList.tsx`

No edits made. No git writes.
