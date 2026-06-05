# Audit — Region/City Filter Bug (backend, read-only)

**Repo:** oglasino-backend · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Bug:** region and city filters do not work — broken identically on web (Next.js) AND mobile (Expo).
**Date:** 2026-06-05

## TL;DR verdict

The backend region/city filter path is **correct end to end** — data serving, predicate
building, ES field mapping, handler registration, and data population all check out, and I
**reproduced the generated query** to confirm it matches. The filter applies correctly **whenever
the request body carries `productsFilter.selectedRegionAndCityValues.{regionIds,cityIds}`**.

There is exactly one place the backend drops the filter silently: when
`selectedRegionAndCityValues` deserializes to **null** (`RegionAndCityQueryGenerator.java:19`), the
generator no-ops with no error. That is the symptom's only backend-side cause. Because two
**independent** frontends are broken identically and the backend predicate is provably correct, the
most-likely root cause is the **wire contract** — the clients are not sending region/city in the
exact `selectedRegionAndCityValues.{regionIds,cityIds}` shape the backend reads — or, secondarily,
a **stale/never-rebuilt ES index**. Neither is decidable from backend code alone; both must be
cross-checked against the web/expo audits (Phase 3) and the live index. I did **not** audit the
frontend repos (per the no-cross-repo rule).

---

## Q1 — Endpoints

### Data-serving (the list of regions/cities a user picks from)

- **`GET /api/public/baseSite/regions/{targetBaseSiteCode}`** →
  `BaseSiteController.getRegionsForBaseSite` (`controller/BaseSiteController.java:49-55`). Returns
  `List<RegionDTO>`, each with nested `List<CityDTO>`.
- Same data is also embedded in **`GET /api/public/baseSite/{targetBaseSiteCode}`** →
  `getBaseSiteDTO` (`BaseSiteController.java:25-31`) as `BaseSiteDTO.regions`.
- Both resolve through `BaseSiteFacade` → `DefaultBaseSiteService.mapToDTO`
  (`service/impl/DefaultBaseSiteService.java:124-128`): `baseSite.getRegions()` →
  `modelMapper.map(region, RegionDTO.class)`. The regions come from the `base_site_regions`
  many-to-many join (`entity/BaseSite.java:57-62`).

### Apply (turning a selected region/city into a product-search filter)

All product searches share one mechanism — a `SearchProductsDTO` body → `buildQuery` → generator
list:

- **`POST /api/public/product/search`** → `ProductSearchController.getAllProducts`
  (`controller/ProductSearchController.java:64-78`); also `/autocomplete` (`:52-61`).
- **`POST /api/secure/products`** + `/autocomplete` → `DashboardProductController`
  (`controller/DashboardProductController.java:70-72, 152-154`).
- **`POST /api/secure/admin/products`** + `/autocomplete` → `AdminProductSearchController`
  (`admin/controller/AdminProductSearchController.java:42-44, 50-52`).

Chain: controller → `ProductsSearchFacade.getProductOverviews`
(`elasticsearch/facade/impl/DefaultProductsSearchFacade.java:52-58`) →
`DefaultProductsSearchService.getProductsFor`
(`elasticsearch/service/impl/DefaultProductsSearchService.java:44-50`) →
`DefaultProductsFilterQueryBuilder.buildQuery`
(`elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java:76-93`). The facade and
service pass `searchProducts` through **untouched** — no rebuild, no field dropped.

The catalog SSR path (`catalog/service/CatalogToJsonService.java`) does **not** do region/city
filtering — it only emits catalog/category JSON. There is no second, non-ES apply path.

---

## Q2 — Trace the APPLY path; is the predicate added or silently dropped?

`buildQuery` → `buildBaseQuery` iterates every `QueryGenerator` bean
(`DefaultProductsFilterQueryBuilder.java:102-112`), calling `generator.addQuery(boolBuilder,
productsFilter)`. One of those beans is `RegionAndCityQueryGenerator`.

`RegionAndCityQueryGenerator.addQuery`
(`elasticsearch/generator/impl/RegionAndCityQueryGenerator.java:16-28`):

```java
var selectedRegionCityValues = productsFilter.getSelectedRegionAndCityValues();
if (Objects.nonNull(selectedRegionCityValues)) {                       // :19  ← the only silent guard
  if (CollectionUtils.isNotEmpty(selectedRegionCityValues.getRegionIds()))
    addShouldListTermFilter(queryBuilder, "regionId", ...getRegionIds());  // :21
  if (CollectionUtils.isNotEmpty(selectedRegionCityValues.getCityIds()))
    addShouldListTermFilter(queryBuilder, "cityId", ...getCityIds());      // :25
}
```

`addShouldListTermFilter` (`:30-46`) builds a `filter` clause containing a `bool` of `should` term
clauses with `minimumShouldMatch("1")` — the correct "OR within a filter" idiom (region 5 OR
region 7).

**The predicate IS added to the query** whenever `selectedRegionAndCityValues` is non-null with a
non-empty list. **It is silently dropped only when `selectedRegionAndCityValues == null`** (the
`:19` guard) — i.e. when the request body did not carry that field (or carried it under a different
key/shape, which Jackson ignores). No validation, no log, no error — exactly the "silent drop"
described in the brief.

**I verified the generated ES query is well-formed and field-typed.** The generator differs
cosmetically from its siblings: it builds the term value via `FieldValue.of(obj)` where `obj` is the
`Object`-typed element of a `List<?>` (`:41`), which binds to `FieldValue.of(java.lang.Object)`
(it wraps the value as `JsonData`), whereas `CategoryQueryGenerator`/`BaseSiteQueryGenerator` pass a
boxed `Long` straight to `.value(...)`. I compiled both against the project's
`elasticsearch-java 9.2.6` jar and serialized them — **they are byte-identical**:

```
of(Object) :  {"term":{"regionId":{"value":5}}}
value(Long):  {"term":{"regionId":{"value":5}}}
```

So the `of(Object)` style is a cosmetic inconsistency (Part 4b, low), **not** a defect — a numeric
term matches the `long`-mapped field.

---

## Q3 — Field names + mapping; client↔backend mismatch?

**Filter request shape (what the backend reads):**

- `ProductsFilterDTO.selectedRegionAndCityValues` (`dto/ProductsFilterDTO.java:19, 111-118`), type
  `SelectedRegionsAndCitiesFilterDTO`.
- `SelectedRegionsAndCitiesFilterDTO` (`dto/SelectedRegionsAndCitiesFilterDTO.java:6-23`):
  `List<Long> regionIds`, `List<Long> cityIds`. **These are entity IDs, not labels.**
  (Corroborated by the prior `audit-product-filtering-backend-reference.md:275`.)

**ES document fields (what the predicate targets):**

- `ProductDocument.regionId` (`long`, `@Field FieldType.Long`) and `cityId`
  (`elasticsearch/documents/ProductDocument.java:62-72`). Field names default to the Java property
  name (no custom ES naming strategy), so the query field strings `"regionId"`/`"cityId"`
  (`RegionAndCityQueryGenerator.java:21,25`) **match the mapping exactly**.

**ID provenance — server-derived, IDs line up:**

- Indexer populates `regionId`/`cityId` from `source.getRegion().getId()` /
  `source.getCity().getId()` (`elasticsearch/converters/DocumentProductConverter.java:87-90`).
- The data-serving endpoint serves the **same** `Region`/`City` entity IDs as `RegionDTO.id` /
  `CityDTO.id` (`dto/RegionDTO.java`, `dto/CityDTO.java` — each `{id, labelKey}`). So a picked
  `RegionDTO.id` sent back as `regionIds` matches a product's indexed `regionId`. No ID-space
  mismatch.

**The brief's `RegionAndCityDTO` / "KeyLabelPair" note — important seam clue.**
`RegionAndCityDTO` (`dto/RegionAndCityDTO.java`) is the **user-profile** shape (`{region, city}`
objects), used in `UserInfoDTO`, `UpdateUserDTO`, `AuthUserDTO` — **not** the search filter. The
filter uses the flat `{regionIds:[…], cityIds:[…]}` shape. The pick-list items are `RegionDTO`/
`CityDTO` (`{id, labelKey}` — the "KeyLabelPair"). **If a client serializes the chosen region/city
as `RegionAndCityDTO`-style objects, or sends the IDs under any key other than
`selectedRegionAndCityValues.regionIds` / `.cityIds`, Jackson silently ignores it and
`selectedRegionAndCityValues` stays null → Q2's silent drop.** This is the single most plausible
cross-frontend cause and is the seam to confirm against the web/expo audits.

---

## Q4 — Does region/city data exist + is it indexed?

**Seeded and linked — data exists:**

- Regions and cities are seeded: `resources/data/location/0001-data-locations-rs.sql`,
  `0002-data-locations-me.sql` (`INSERT INTO region …`, `INSERT INTO city …`).
- Base sites are linked to regions via `base_site_regions`:
  `resources/data/basesite/0001-data-rs.sql:7`, `0002-data-me.sql:6`, `0003-data-rs-moto.sql:7`.
  Schema: `db/migration/V1__init_schema.sql:106` (`base_site_regions` table). So the data-serving
  endpoint returns real, non-empty region/city options.

**Products carry region/city, and they are indexed:**

- `Product.region` / `Product.city` are `@ManyToOne` FKs (`entity/Product.java:46-52`).
- `DocumentProductConverter` writes `regionId`/`cityId` (+ `regionLabelKey`/`cityLabelKey`) into the
  document on every index (`DocumentProductConverter.java:87-90`), invoked by the indexer via
  `modelMapper.map` (`ProductIndexer.toIndexQuery:341-344`, `indexOne:371`).

**Caveats (operational, not code-correctness):**

1. **The filter only works against an index that was (re)built after `regionId`/`cityId` existed
   in `ProductDocument`.** If the live `products` index predates those fields or was never
   rebuilt, the fields are absent and the term matches nothing — silently. (These fields are
   long-standing — **not** in the current working-tree diff — so this is only a concern if the
   environment's index is stale.) Verifying this requires inspecting the live index mapping/contents
   (`GET products/_mapping`, a sample doc) — out of reach from source.
2. **Null region/city on a product breaks indexing for that product.** `Product.region`/`city` are
   nullable (no `nullable=false` on the `@JoinColumn`, `Product.java:46-52`), but
   `DocumentProductConverter.java:87,89` dereferences `source.getCity().getId()` /
   `getRegion().getId()` unguarded → NPE. In `indexOne` that aborts that product; in a full reindex
   it aborts the batch and leaves the alias untouched (`ProductIndexer.doReindex`). This is a hard
   failure, not the silent symptom — flagged for completeness.
3. **Minor:** `ProductIndexer.indexOne` (`:361-367`) `Hibernate.initialize`s translations/baseSite/
   owner/categories/imageKeys but **not** `region`/`city`. Harmless today because `indexOne` is
   `@Transactional` (`:349`) so the lazy `getRegion()`/`getCity()` access inside the converter still
   loads within the open session. Worth a note only if that method is ever made non-transactional.

---

## Q5 — Recent change, or pre-existing? Is the handler registered?

**Handler IS registered.** `RegionAndCityQueryGenerator` is `@Service implements QueryGenerator`
(`:12-13`). `DefaultProductsFilterQueryBuilder` autowires `List<QueryGenerator> generators` (`:25`)
and invokes every one for every search mode (`:102-112`). There is **no filter-type dispatcher /
switch** that could miss a case — it is a fixed bean list, and region/city is always processed.
Nothing is missing or unmatched.

**Pre-existing, not broken by the current batch.** The uncommitted working-tree changes in the ES
package are an unrelated **translations refactor** (`ProductDocument` name/description translation
references, `getTranslatedValue` → `static`, the three/four converters) plus a **+6-line security
change** in `DefaultProductsFilterQueryBuilder` (the `DASHBOARD_SEARCH` `UserQueryGenerator` skip,
`:104-112`). `git diff` on `ProductDocument.java` touches **only** the translation fields/methods —
`regionId`/`cityId` are untouched. `RegionAndCityQueryGenerator.java` is unchanged (only the two
repo-wide commits ever touch it). So whatever is failing is **pre-existing behavior**, consistent
with "broken on both frontends" rather than a regression from this batch.

---

## Q6 — Verdict: where is it failing?

**Not in the backend predicate, field mapping, handler registration, or data serving** — all four
are verified correct, and the generated query was reproduced as a valid numeric term against the
correctly-mapped `regionId`/`cityId` fields. Given `productsFilter.selectedRegionAndCityValues`
with `regionIds`/`cityIds`, the backend filters correctly.

**Most-likely root cause (high confidence): the request contract / seam.** The only backend path
that produces the exact reported symptom — filter ignored, no error — is
`selectedRegionAndCityValues == null` at `RegionAndCityQueryGenerator.java:19`, which happens when
the request body does not carry that field in the precise nested shape the backend reads:

```json
{ "productsFilter": { "selectedRegionAndCityValues": { "regionIds": [<id>], "cityIds": [<id>] } } }
```

Two independent frontends failing identically, against a provably-correct backend predicate, points
to a shared contract error in what the clients serialize (wrong key name, or sending
`RegionAndCityDTO`-style objects / `labelKey`s instead of `{regionIds,cityIds}` of entity IDs — see
Q3). **This must be confirmed by the web and expo audits** — it is not decidable from backend code.

**Secondary candidate (operational): stale ES index** lacking `regionId`/`cityId` for the live
documents (Q4 caveat 1) — verify the live `products` mapping + a sample document and trigger a
reindex if absent.

**What is NOT the cause:** missing handler, unmatched dispatcher case, field-name mismatch, ID-space
mismatch, the `FieldValue.of(Object)` style, the facade/service layer, or this batch's changes.

### Recommended next steps (for Mastermind / Phase 3)
1. Diff the web + expo region/city filter request bodies against the exact contract above (key name
   and `{regionIds,cityIds}` of entity IDs). This is the prime suspect.
2. Inspect the live `products` index: `GET products/_mapping` (confirm `regionId`/`cityId` present
   as `long`) and a sample doc (confirm populated). Reindex if stale.
3. Lowest priority: normalize `RegionAndCityQueryGenerator` to the sibling `.value(Long)` style
   (cosmetic) and guard null region/city in `DocumentProductConverter` (Q4 caveat 2).

---

## Adjacent observations (Part 4b)

- **(low, cosmetic)** `RegionAndCityQueryGenerator.java:41` builds term values via
  `FieldValue.of(Object)` (JsonData-wrapped) while sibling generators pass typed `Long`. Proven
  byte-identical output; pure inconsistency. Not fixed (read-only; out of scope).
- **(low/medium)** `DocumentProductConverter.java:87,89` dereferences `source.getCity()`/
  `getRegion()` unguarded though `Product.region`/`city` are nullable
  (`entity/Product.java:46-52`) → NPE aborts indexing for a region/city-less product. Not fixed
  (read-only; out of scope).
- **(low)** `ProductIndexer.indexOne` (`:361-367`) omits `Hibernate.initialize` for `region`/`city`;
  benign only because the method is `@Transactional`. Not fixed (read-only; out of scope).
- **Zero test coverage** of the region/city filter path (no test references
  `RegionAndCityQueryGenerator`, `selectedRegionAndCityValues`, `regionIds`, or `cityIds`). A single
  contract test asserting the generated query (or a slice test posting the documented body) would
  have caught a contract drift immediately. Not fixed (read-only; out of scope).
