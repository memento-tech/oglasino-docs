# Audit — Filter query-param vocabulary (home + catalog) for deep-link hydration

**Repo:** oglasino-web · **Branch:** `dev` (read-only, no edits) · **Date:** 2026-06-04
**Feature:** filter deep-linking — web is the contract; mobile must parse exactly what web emits.
**Scope:** home (`/{locale}`) and catalog (`/{locale}/catalog[/…]`) only.

## TL;DR

- **The authoritative emitter** is the SYNC-TO-URL effect in `FilterManager.tsx:230-355`. It writes a flat, single-level query string with `URLSearchParams` and navigates with `router.replace`. Twelve possible keys; only seven are in scope for public deep-links (the two `…States` keys are owner/admin-only, and category travels in the **path**, not the query).
- **`randomSeed` is NOT in the URL** — a shared link does **not** pin a friend's shuffle. (Q7)
- **There are TWO inbound parsers and they do not agree on region/city and search_text decoding.** The SSR `parseFiltersFromQueryParams` (filtersHelper.ts) is the clean, accent-robust reference mobile should mirror. The client-store `FilterManager` hydrate effect uses a lossy `fromQueryParam` round-trip that breaks on accents/casing — **mobile must not copy that one.** (Q3, see "For Mastermind".)
- **`URLSearchParams` percent-encodes the structural chars** in range filters: `:` → `%3A`, `,` → `%2C`. Mobile must URL-decode each value before splitting on `,` / reading the `from:`/`to:` tokens. (Q4)

---

## Q1 — The full filter query-param vocabulary

Every key the emitter (`FilterManager.tsx:230-355`) can write, with value encoding:

| # | Key | Value format | Encoding fn | Emitter line |
|---|-----|--------------|-------------|--------------|
| 1 | `search_text` | single slug value | `toQueryParam(searchText)` | `FilterManager.tsx:236` |
| 2 | `f_<filterKey>` (OPTION) | comma-joined option slugs | `toQueryParam(t(opt.labelKey))` per option, `.join(',')` | `:244-247` |
| 3 | `f_<filterKey>` (RANGE/DATE) | `from:<prefix><n><suffix>` and/or `to:<prefix><n><suffix>`, comma-joined | raw number with the filter's `rangePrefix`/`rangeSuffix` bolted on | `:255-269` |
| 4 | `from` | raw string (NOT slugified) | none — `params.set('from', selectedPriceRange.from)` | `:275` |
| 5 | `to` | raw string (NOT slugified) | none | `:276` |
| 6 | `free` | literal `true` (key omitted when false) | none | `:277` |
| 7 | `currency` | currency `code` (e.g. `EUR`) — only when a price bound is set | none | `:281-286` |
| 8 | `order` | order `code` (backend-defined `OrderTypeDTO.code`) | none | `:288-290` |
| 9 | `regions` | comma-joined region-label slugs | `toQueryParam(t(r.labelKey))`, `.join(',')` | `:292-297` |
| 10 | `cities` | comma-joined city-label slugs | `toQueryParam(t(c.labelKey))`, `.join(',')` | `:299-304` |
| 11 | `productStates` | comma-joined raw enum (`ProductState`) | none | `:306-308` |
| 12 | `moderationStates` | comma-joined raw enum (`ModerationState`) | none | `:309-311` |

**Scope note:** keys 11–12 are set only on `/owner/products` and `/admin/products` (the shared filter store only carries product/moderation states on those surfaces). On home and catalog they are always empty, so they never appear in a public deep-link. They are listed for completeness; the frozen mobile contract for home/catalog is keys 1–10.

**`toQueryParam` (filtersHelper.ts:14-23) — the slug encoder used for keys 1, 2, 9, 10:**
1. NFD-normalize, strip combining accents (`/[̀-ͯ]/g`) — **lossy** (`Niš` → `nis`).
2. literal `-` → `_d_` (dash-escape token).
3. whitespace `\s+` → `-`, then `.toLowerCase()`.

Examples: `"BMW X5"` → `bmw-x5`; `"Novi Sad"` → `novi-sad`; `"Pré-owned"` → `pre_d_owned`; `"Niš"` → `nis`.

Note the **price** `from`/`to` (keys 4–5) are emitted **raw**, not through `toQueryParam` — they are numeric strings off the price input.

---

## Q2 — Where the URL is built (the authoritative emitter)

`src/components/client/initializers/FilterManager.tsx`, the **"2. SYNC TO URL"** `useEffect` (lines 230-355). It runs only after hydration on an allowed path (`isAllowedPath()` — `/`, `/catalog*`, `/owner/products`, `/admin/products`). The serializer verbatim:

```ts
const params = new URLSearchParams();

if (searchText) params.set('search_text', toQueryParam(searchText));

selectedFilters.forEach((f) => {
  switch (f.filter.filterType) {
    case FilterType.SINGLE_OPTION:
    case FilterType.MULTI_OPTION: {
      if (!f.options.length) return;
      params.set(`f_${f.filter.filterKey}`,
        f.options.map((o) => toQueryParam(t(o.labelKey))).join(','));
      break;
    }
    case FilterType.DATE:
    case FilterType.RANGE: {
      if (!f.rangeFrom && !f.rangeTo) return;
      const filterRangeData = f.filter.filterRange;
      const final = [];
      const prefix = (filterRangeData && filterRangeData.rangePrefix) || '';
      const suffix = (filterRangeData && filterRangeData.rangeSuffix) || '';
      if (f.rangeFrom) final.push(`from:${prefix}${f.rangeFrom}${suffix}`);
      if (f.rangeTo)   final.push(`to:${prefix}${f.rangeTo}${suffix}`);
      params.set(`f_${f.filter.filterKey}`, final.join(','));
      break;
    }
  }
});

if (selectedPriceRange.from) params.set('from', selectedPriceRange.from);
if (selectedPriceRange.to)   params.set('to', selectedPriceRange.to);
if (selectedPriceRange.free) params.set('free', 'true');

const hasValue = (v?: string) => !!v?.trim();
if (selectedPriceRange.selectedCurrency &&
    (hasValue(selectedPriceRange.from) || hasValue(selectedPriceRange.to))) {
  params.set('currency', selectedPriceRange.selectedCurrency.code);
}

if (selectedOrder) params.set('order', selectedOrder.code);

if (selectedRegionsAndCities.regions.length) {
  params.set('regions',
    selectedRegionsAndCities.regions.map((r) => toQueryParam(t(r.labelKey))).join(','));
}
if (selectedRegionsAndCities.cities.length) {
  params.set('cities',
    selectedRegionsAndCities.cities.map((c) => toQueryParam(t(c.labelKey))).join(','));
}

if (selectedProductStates.length > 0)   params.set('productStates', selectedProductStates.join(','));
if (selectedModeratioStates.length > 0) params.set('moderationStates', selectedModeratioStates.join(','));

const pathSegment = pathname === '/' ? '' : pathname;
const newUrl = `/${locale}${pathSegment}` + (params.toString() ? `?${params}` : '');
// …
router.replace(newUrl, { scroll: false });
```

**Encoding caveat (verified with Node):** `URLSearchParams.toString()` percent-encodes the structural characters inside values. `f_year=from:2010,to:2020` is emitted on the wire as `f_year=from%3A2010%2Cto%3A2020`; `regions=novi-sad,beograd` as `regions=novi-sad%2Cbeograd`. The mobile parser must decode (`%3A`→`:`, `%2C`→`,`) before splitting.

**Second emitter (search only):** `SearchInput.tsx:141` does its own
`router.push(\`${targetPath}?search_text=${toQueryParam(rawTerm)}\`)`
when the user commits a search. Same `search_text` slug encoding as key 1, so no contract divergence — it just seeds the search param on navigation; `FilterManager` then takes over the full serialization. (`targetPath` is `/catalog/...` when already on catalog, else `/`.) No other component writes filter params to the URL (grep-confirmed; the `favorites` page's `applyRandom:true` is an internal request flag, not a URL write).

---

## Q3 — Where the URL is parsed back (web's own inbound)

**There are two inbound parsers. They consume the same vocabulary but produce different shapes and — critically — decode `regions`/`cities`/`search_text` differently.**

### 3a. `parseFiltersFromQueryParams` (filtersHelper.ts:38-73) — the request-DTO builder (SSR)

Called by `filterHydrationSSR` (`FilterHydrationSSRInit.tsx:24`), which is invoked server-side by the home page (`app/[locale]/(portal)/(public)/page.tsx:40`, `applyRandom=true`) and the catalog page (`catalog/[[...slugs]]/page.tsx:113`, `applyRandom=true`, with resolved `categories`). It produces a **`ProductsFilterDTO`** (the request body for the product query):

```ts
let result: ProductsFilterDTO = {
  searchText: getSearchText(params),                       // raw URL value, NOT decoded
  filters: getSelectedFilters(params, baseSite.catalog, t, categories),
  orderBy: getOrderData(params, baseSite.catalog),         // matched by .code
  priceRange: getPriceRangeData(params, baseSite),         // {from, to, free, selectedCurrency}
  selectedRegionsAndCities: getRegionAndCitiesData(params, baseSite, t),
  productStates: getProductStatesData(params),
  moderationStates: getModerationStatesData(params),
};
```

`ProductsFilterDTO` shape (`ProductsFilterDTO.ts`):

```ts
interface ProductsFilterDTO {
  applyRandom?: boolean;  randomSeed?: number;
  ownerId?: number;  topCategoryId?: number;  subCategoryId?: number;  finalCategoryId?: number;
  excludeIds?: number[];
  searchText?: string;
  filters?: RequestSelectedFilterDTO[];     // {filterId, filterType, optionIds?, rangeFrom?, rangeTo?}
  orderBy?: OrderTypeDTO;                    // {code, field, labelKey, direction}
  priceRange?: PriceRange;                  // {from: string, to: string, selectedCurrency?, free: boolean}
  selectedRegionsAndCities?: SelectedRegionsAndCities;  // {regions: RegionDTO[], cities: CityDTO[]}
  productStates?: ProductState[];  moderationStates?: ModerationState[];
}
```

This parser resolves URL slugs back to **backend IDs/objects** (option `id`s, filter `id`, currency/region/city objects, order object). Region/city matching is **accent-robust** because both sides are run through `toQueryParam`:
```ts
// getRegionAndCitiesData, filtersHelper.ts:228-233
const regionValues = params.find(p => p.key === 'regions')?.value?.split(',') || [];
const regions = baseSite.regions?.filter(r => regionValues.includes(toQueryParam(t(r.labelKey)))) || [];
```

### 3b. `FilterManager` "1. HYDRATE FROM URL" effect (FilterManager.tsx:77-225) — the client store populator

Reads `window.location.search` and sets the Zustand `FilterState` (display state: chips, inputs). It is the **inverse for the UI**, not for the request. It decodes differently:

- `search_text`: `setSearchText(fromQueryParam(params.get('search_text')))` (`:117-119`) — applies `fromQueryParam` (slug → Title Case).
- `regions`/`cities`: `params.get('regions')?.split(',').map(fromQueryParam)` then matched against **raw** `t(r.labelKey)` (`:206-214`) — `fromQueryParam` round-trip, **lossy on accents/casing**.
- option/range filters: same slug/`from:`/`to:` logic as the SSR parser.

**The mobile contract should mirror 3a (`parseFiltersFromQueryParams`), not 3b.** See "For Mastermind".

---

## Q4 — Dynamic per-category filters (the hard case)

**Keying.** Every dynamic filter is keyed `f_<filterKey>` (literal `f_` prefix + the filter's `filterKey`). Emitted at `FilterManager.tsx:245,269`; parsed by stripping `f_` and resolving the filter against, in order: `baseSite.catalog.topFilters` → `categories.topCategory.filters` → `subCategory.filters` → `finalCategory.filters` (`FilterManager.tsx:130-133`, `filtersHelper.ts:150-153`). There is **no SELECT type** — the enum is exactly `SINGLE_OPTION | MULTI_OPTION | RANGE | DATE` (`FilterType.ts`).

**OPTION filters (SINGLE_OPTION and MULTI_OPTION — identical encoding):**
- Wire: `f_<filterKey>=<slug1>,<slug2>,…` where each slug = `toQueryParam(t(option.labelKey))`.
- Multi-select → multiple comma-joined values; single-select → one value. **Same `f_` key, same comma-join for both types** (there is no per-type key difference).
- Example: `f_condition=new,used`, `f_color=black`.
- Parse (`filtersHelper.ts:157-172`): split on `,`, match each slug against `toQueryParam(t(option.labelKey))`, collect matching `option.id` → `{filterId, filterType, optionIds:[…]}`.

**RANGE and DATE filters (identical encoding — this is the pair dropped in the mobile catalog-filters bug):**
- Wire: `f_<filterKey>=from:<prefix><number><suffix>` and/or `to:<prefix><number><suffix>`, comma-joined. Either bound may be absent.
- `<prefix>`/`<suffix>` come from the filter definition's `filterRange.rangePrefix` / `rangeSuffix` (`FilterRangeDTO`, both are `string`, may be empty). The `from:`/`to:` tokens are literal.
- Examples: `f_year=from:2010,to:2020` (no affixes); `f_price_range=from:$100,to:$500` (prefix `$`); `f_mileage=to:100000km` (suffix `km`, no lower bound).
- On the wire (percent-encoded): `f_year=from%3A2010%2Cto%3A2020`.
- Parse (`extractRangeData` / `extractRangeNumber`, `filtersHelper.ts:92-131`): split on `,`; a token `startsWith('from')` → `from`, else → `to`; then the value is stripped of `prefix`/`suffix` and **everything non-digit/non-dot** is removed (`token.replace(/[^\d.]/g, '')`), so the `from:`/`to:` label and the `:` separator are discarded by the regex regardless. Result: `{filterId, filterType, rangeFrom: number, rangeTo: number}` (`filtersHelper.ts:182-189`).
- **Mobile parser checklist for RANGE/DATE:** (1) URL-decode the value; (2) split on `,`; (3) classify each piece by `from`/`to` prefix; (4) strip the filter's `rangePrefix`/`rangeSuffix`; (5) extract the numeric portion; (6) emit `rangeFrom`/`rangeTo` as numbers. Dropping `rangeFrom`/`rangeTo` here is exactly the prior bug — both bounds must survive into the request DTO.

---

## Q5 — Region/city encoding (URL)

- **Regions:** `regions=<slug1>,<slug2>` — comma-joined `toQueryParam(t(region.labelKey))` (`FilterManager.tsx:292-297`). Example `regions=beograd,novi-sad`.
- **Cities:** `cities=<slug1>,<slug2>` — comma-joined `toQueryParam(t(city.labelKey))` (`FilterManager.tsx:299-304`). Example `cities=zemun,vracar`.
- Both are **flat label-slug lists**. This is **not** the request-body shape: the body sends `selectedRegionsAndCities: {regions: RegionDTO[], cities: CityDTO[]}` (full objects), which `getRegionAndCitiesData` reconstructs by matching the URL slugs against `baseSite.regions` and the flattened `baseSite.regions[].cities`. So the URL carries label slugs; the request carries resolved objects.
- **Collapse-to-region:** the auto-collapse (when all cities of a region are selected, the UI collapses to the region) is **store-side behavior** — it mutates `selectedRegionsAndCities` before serialization, so the URL simply reflects whatever the store holds at emit time (collapsed → a region appears in `regions` and the individual cities drop out of `cities`). There is no separate collapse flag in the URL; the collapse is already baked into the emitted `regions`/`cities` lists.
- City→region association is **not** encoded — cities are a flat global list matched across all regions (`baseSite.regions.flatMap(r => r.cities)`). A city slug that is ambiguous across regions would match the first hit; not observed to be a problem with current data, flagged as a latent edge.

---

## Q6 — Home vs catalog differences

- **Identical param vocabulary and encoding.** Same `FilterManager` serializer runs on both; same `parseFiltersFromQueryParams` parses both.
- **Path differs, not the query:** home URL = `/{locale}` (`pathSegment` empty, `FilterManager.tsx:313`); catalog = `/{locale}/catalog/<topRoute>[/<subRoute>[/<finalRoute>]]`. **Category is carried in the PATH, never the query** — segments are matched to `baseSite.catalog.categories[].route` (`FilterManager.tsx:84-112`). So a deep-link's category context comes from the path, and `topCategoryId`/`subCategoryId`/`finalCategoryId` in `ProductsFilterDTO` are derived from the path, not parsed from the query.
- **Available dynamic filters differ by surface:** on home, only `baseSite.catalog.topFilters` resolve (no category in path), so only top-level `f_<key>` filters can appear. On catalog, `topFilters` **plus** the path-resolved category filters resolve. The *encoding* of any `f_<key>` is identical; the *set* of resolvable keys is larger on catalog.
- **`applyRandom`** is `true` for both home and catalog (both SSR callers pass `true`); `false` for owner/admin. But `applyRandom` is **not in the URL** (Q7), so this does not affect the link contract.
- **`productStates`/`moderationStates`** are emitted only on owner/admin surfaces, never home/catalog (Q1 scope note).

---

## Q7 — The random-seed param

**`randomSeed` is NOT in the shareable URL, and neither is `applyRandom`.** The emitter never writes them (grep-confirmed; they are absent from the `FilterManager` SYNC-TO-URL effect). They are **synthesized at parse time**, server-side, in `parseFiltersFromQueryParams`:

```ts
const randomAddOn = { applyRandom: true, randomSeed: Date.now() };
if (params.length === 0 && applyRandom) return { ...randomAddOn };
// …
if (applyRandom && !result.searchText && !result.orderBy) {
  result = { ...result, ...randomAddOn };   // randomSeed = Date.now(), fresh each request
}
```

So a shared link does **not** pin a friend's shuffle — each open computes a fresh `Date.now()` seed (and only when `applyRandom` is on **and** there is no `searchText`/`orderBy`; an explicit sort or search suppresses random entirely). **Product implication for mobile:** mobile should likewise synthesize its own seed on open rather than expecting one in the link. (See "For Mastermind".)

---

## Q8 — What's NOT in the URL

Fields that live only in the request DTO / store / path and can therefore **not** be deep-linked via the query string:

- **`applyRandom` / `randomSeed`** — synthesized at parse time (Q7).
- **`ownerId`** — derived from auth/route (owner/admin surfaces), never a public-link concern.
- **`topCategoryId` / `subCategoryId` / `finalCategoryId`** — derived from the **path** (catalog) — not lost, just carried in the path rather than the query.
- **`excludeIds`** — pagination/infinite-scroll exclusion list, request-only.
- **Currency without a price bound** — `currency` is suppressed unless `from` or `to` has a trimmed value (`FilterManager.tsx:281-284`). A currency selection with empty price bounds does **not** round-trip.

Everything else relevant to home/catalog (search, option filters, range/date filters, price from/to, free, order, regions, cities) **is** in the URL and round-trips.

---

## Filter URL contract (frozen — mobile parses against this)

For home and catalog public deep-links. Values shown **pre-percent-encoding**; on the wire `:`→`%3A`, `,`→`%2C`.

| Param key | Value encoding | Surface | Example (decoded) |
|-----------|----------------|---------|-------------------|
| `search_text` | single slug, `toQueryParam(text)` (lowercased, accents stripped, `-`→`_d_`, spaces→`-`) | both | `search_text=bmw-x5` |
| `f_<filterKey>` (OPTION) | comma-joined option slugs, `toQueryParam(t(labelKey))` each | both¹ | `f_condition=new,used` |
| `f_<filterKey>` (RANGE/DATE) | `from:<prefix><n><suffix>` and/or `to:<prefix><n><suffix>`, comma-joined | both¹ | `f_year=from:2010,to:2020` |
| `from` | raw numeric string (no encoding) | both | `from=100` |
| `to` | raw numeric string (no encoding) | both | `to=500` |
| `free` | literal `true` (omitted when false) | both | `free=true` |
| `currency` | currency `code`; only when a price bound is set | both | `currency=EUR` |
| `order` | `OrderTypeDTO.code` | both | `order=price_asc` |
| `regions` | comma-joined region-label slugs, `toQueryParam(t(labelKey))` | both | `regions=beograd,novi-sad` |
| `cities` | comma-joined city-label slugs, `toQueryParam(t(labelKey))` | both | `cities=zemun,vracar` |
| `productStates` | comma-joined raw `ProductState` enum | owner/admin only² | `productStates=ACTIVE,INACTIVE` |
| `moderationStates` | comma-joined raw `ModerationState` enum | admin only² | `moderationStates=APPROVED` |
| *(category)* | **path segments**, not query: `/catalog/<topRoute>/<subRoute>/<finalRoute>` matched on `category.route` | catalog | `/sr/catalog/electronics/phones` |

¹ Home resolves only `topFilters`; catalog resolves `topFilters` + path-category filters. Encoding identical.
² Out of scope for public home/catalog deep-links; listed for completeness.

**Not in the query (cannot be deep-linked):** `applyRandom`, `randomSeed`, `ownerId`, `excludeIds`, currency-without-price. Category IDs travel in the path.

---

## For Mastermind

**Part 4a simplicity evidence (required):**
- Added (earned complexity): nothing — read-only audit, no code.
- Considered and rejected: nothing.
- Simplified or removed: nothing.

**Findings / risks for the mobile filter-contract work:**

1. **(high — the central contract decision) Web has two inbound parsers that disagree; mobile must mirror the SSR one, not the client-store one.**
   - `parseFiltersFromQueryParams` (filtersHelper.ts) is **accent-robust** for regions/cities — it compares `toQueryParam(t(labelKey))` on both emit and parse, so the lossy slug is matched against an equally-slugged candidate. This is the clean reference.
   - `FilterManager`'s "HYDRATE FROM URL" effect (FilterManager.tsx:206-214) instead does `fromQueryParam` on the URL value and compares against the **raw** `t(labelKey)`. `fromQueryParam` cannot recover stripped accents or original casing (`Niš` → slug `nis` → `Nis` ≠ `Niš`), so a deep-linked region/city with an accent fails to re-select in the client store. Mobile copying *this* path would reintroduce exactly the region/city drift class the brief warns about.
   - **Recommendation:** freeze the mobile parser against `parseFiltersFromQueryParams`'s logic (slug-vs-slug comparison via `toQueryParam`), and flag FilterManager.tsx:206-214 as a likely **pre-existing web bug** (accented region/city deep-links don't re-hydrate the chips, though the request still filters correctly because the SSR parser is right). Out of scope for this read-only audit; logged for a follow-up web brief.

2. **(medium — quirk to decide) `search_text` is slugified into the request, lossily, and the two parsers disagree on whether to reverse it.**
   - Emitter writes `toQueryParam(searchText)` (e.g. `bmw-x5`). The SSR parser (`getSearchText`, filtersHelper.ts:75-85) sends that slug **verbatim** to the backend as `searchText` (so the backend searches on `bmw-x5`, spaces already collapsed to dashes and lowercased). The client store reverses it with `fromQueryParam` to `Bmw X5` for the chip display.
   - This is lossy (a search for `BMW X5` and `bmw-x5` collapse to the same wire value) and the front-of-house display vs. wire value diverge. It's how web works today. **Mobile should mirror the SSR behavior** (send the slug as `searchText`) to stay byte-identical with web's request, but Mastermind should confirm this is intended rather than a latent web bug before freezing it.

3. **(product question — Q7) `randomSeed` is correctly absent from shared links.** A shared link does not pin a shuffle; each open reseeds (`Date.now()`), and only when there's no search/sort. No bug — but confirm the desired mobile behavior is "reshuffle on open" (it should synthesize its own seed, matching web). No change needed web-side.

4. **(low — latent edge) City→region association is not encoded.** `cities` is a flat global slug list matched across all regions (`flatMap`). An ambiguous city name shared across regions would match the first hit. Not observed to bite with current data; flagged for the mobile parser author's awareness.

5. **Encoding reminder for the mobile parser:** values are percent-encoded on the wire (`:`→`%3A`, `,`→`%2C`). Decode before splitting. Price `from`/`to` are raw (not slugified); option/region/city/search values are slugified via `toQueryParam`; states are raw enum strings.

**Is web's `parseFiltersFromQueryParams` a clean reference for mobile to mirror?** Yes — it is the authoritative inbound counterpart to the emitter and is accent-robust. The only quirk to confirm before freezing is the `search_text` slug-as-wire-value behavior (finding 2). Do **not** mirror the `FilterManager` client-hydrate region/city decode (finding 1).

**Config-file impact:**
- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change required by this audit itself. Findings 1 and 2 are candidate `issues.md` entries (pre-existing web parser quirks) **if** Mastermind/Igor decide to log them — drafted above for that purpose, not written here (Docs/QA is the sole writer per conventions Part 3).

**Conventions check:** Part 4 (cleanliness): N/A — read-only audit, no code touched. Part 6 (translations): N/A. Part 11 (trust boundaries): N/A to a URL-vocabulary audit (the server remains the moderation/auth boundary; query params are display/filter criteria only).
