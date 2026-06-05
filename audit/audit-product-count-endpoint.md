# Audit — Product Count Endpoint

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-27
**Task:** Read-only audit to inventory the surface for a count-only product endpoint, enabling the sitemap to avoid fetching a full product payload just to read a count.

---

## 1. Existing `/api/public/product/*` endpoints

### Controller: `ProductSearchController`

**File:** `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java`
**Base mapping:** `@RequestMapping("/api/public/product/search")`

| HTTP | Path | Method | Signature |
|------|------|--------|-----------|
| GET | `/api/public/product/search` | `getProductDetails` | `@RequestParam @NotNull Long productId` → `ResponseEntity<ProductDetailsDTO>` |
| POST | `/api/public/product/search/autocomplete` | `autocomplete` | `@RequestBody SearchProductsDTO` → `ResponseEntity<List<SearchProductData>>` |
| POST | `/api/public/product/search` | `getAllProducts` | `@RequestBody SearchProductsDTO` → `ResponseEntity<ProductOverviewsDataDTO>` |

### Controller: `PublicProductController`

**File:** `src/main/java/com/memento/tech/oglasino/controller/PublicProductController.java`
**Base mapping:** `@RequestMapping("/api/public/product")`

| HTTP | Path | Method | Signature |
|------|------|--------|-----------|
| GET | `/api/public/product/seen/{productId}` | `setProductAsSeen` | `@PathVariable @NotNull Long productId, HttpServletRequest` → `ResponseEntity<Void>` |
| GET | `/api/public/product/views/{productId}` | `getNumberOfViews` | `@PathVariable @NotNull Long productId` → `ResponseEntity<Long>` |

### Auth annotations

No `@PreAuthorize` on any of these controllers. Security is handled at the filter-chain level:

- **`SecurityConfig.java:77`** — `/api/public/**` is in the `permitAll()` matcher. All public product endpoints are anonymous-accessible.
- No per-method security annotations exist on either controller.

### X-Base-Site header handling

**`BaseSiteFilter.java:15-37`** — a `@Component` `OncePerRequestFilter` that runs on all `/api/**` paths. It reads:

1. `request.getHeader("X-Base-Site")` (line 25)
2. Falls back to `request.getParameter("baseSite")` if header is null/blank (line 27)

Resolves to a `BaseSiteDTO` via `BaseSiteCacheService.getBaseSiteForCode(baseSiteCode)` and stores in `BaseSiteContext` — a `@RequestScope` `@Component` (line 8-9 of `BaseSiteContext.java`).

**Downstream consumption:** `BaseSiteQueryGenerator.java:18-22` adds a `term` filter on `baseSiteId` to the ES bool query when `baseSiteContext.getCurrentBaseSite()` is non-null. This means base-site scoping on search queries is driven entirely by the request-scoped `BaseSiteContext`, populated from the header/query-param.

### Rate limiting

**`RateLimitFilter.java:128-129`** — any path starting with `/api/public/product/search` maps to `RateLimitCategory.SEARCH`.

**`RateLimitCategory.SEARCH`** (line 18-20):
- 60 requests per 1 minute (burst)
- 500 requests per 1 hour (sustained)

The categorization uses `path.startsWith("/api/public/product/search")`, so a new endpoint at `/api/public/product/count` would **not** be caught by this rule. It would need its own categorization entry, or could reuse the `SEARCH` category if added under `/api/public/product/search/count`.

---

## 2. The existing search endpoint internals

### Request DTO shape

**`SearchProductsDTO.java`** — the POST body:
- `ProductsFilterDTO productsFilter` — filters (categories, search text, price range, regions, product/moderation states, owner, random seed, order, exclude IDs, dynamic filters)
- `PagingDTO paging` — page number + page size; defaults to `Pageable.ofSize(10)` when null

### Controller → service delegation

```
ProductSearchController.getAllProducts()          (controller)
  → ProductsSearchFacade.getProductOverviews()    (facade)
    → ProductsSearchService.getProductsFor()      (service)
      → ElasticsearchOperations.search()          (ES client)
    → getResponseData(searchHits)                 (facade, maps results)
```

**Facade layer (`DefaultProductsSearchFacade.java:53-58`):** calls `productsSearchService.getProductsFor(searchProducts, PORTAL_SEARCH)` and passes the `SearchHits<ProductDocument>` result to `getResponseData()`.

**`getResponseData` (line 74-89):** does two things:
1. Maps each `SearchHit<ProductDocument>` → `ProductOverviewDTO` via ModelMapper (line 77-84)
2. Reads `searchHits.getTotalHits()` → sets `result.setTotalNumberOfProducts(...)` (line 85-86)

### Count-only method — does one exist?

**No.** There is no count-only method on `ProductsSearchService`, `ProductsSearchFacade`, or `ProductSearchController`. Counting is folded into the full search: the count is extracted from `SearchHits.getTotalHits()` as a side effect of a full Elasticsearch search query.

However, `ElasticsearchOperations.count(Query, IndexCoordinates)` **is used elsewhere** in the codebase:

- **`DefaultEsStateService.java:125-133`** — `countDocumentsViaAlias()` calls `elasticsearchOperations.count(NativeQuery.builder().withQuery(q -> q.matchAll(m -> m)).build(), IndexCoordinates.of(ALIAS_NAME))` to count all documents in the products index. This is an admin/health-check path, not filtered by base site or state.

### The exact response shape

**`ProductOverviewsDataDTO.java`:**
```java
{
  "products": [ ...List<ProductOverviewDTO>... ],
  "totalNumberOfProducts": <long>
}
```

`totalNumberOfProducts` comes from Elasticsearch's `SearchHits.getTotalHits()` — this is the ES total hit count, which includes all matched documents regardless of pagination. When `withTrackTotalHits(true)` is set (and it is — line 86 of `DefaultProductsFilterQueryBuilder.java`), ES returns the exact count rather than a lower-bound approximation.

### What the count represents

For `PORTAL_SEARCH` mode, the count is: all products where `productState = ACTIVE` AND `moderationState = APPROVED` AND `baseSiteId = <current base site>` (if `X-Base-Site` header is present), AND any additional filters from `ProductsFilterDTO`. The state hard-pin to ACTIVE + APPROVED is a trust-boundary enforcement in `ProductStateQueryGenerator.java:52-55` — client-supplied `productStates`/`moderationStates` are **ignored** for `PORTAL_SEARCH`.

---

## 3. Count-only path candidates

### Shape A: `GET /api/public/product/count?baseSite=<code>`

**Backend code path:**
- `BaseSiteFilter` populates `BaseSiteContext` from the `baseSite` query param (already supported — line 27 of `BaseSiteFilter.java`).
- New controller method builds a count query with ACTIVE + APPROVED state filters + `baseSiteId` term from `BaseSiteContext`, calls `ElasticsearchOperations.count()`.
- Returns `ResponseEntity<Long>` (or a thin wrapper like `{ "count": <long> }`).

**Precedent:**
- `PublicProductController.getNumberOfViews` returns `ResponseEntity<Long>` — lightweight GET → `long` response. Same controller base path (`/api/public/product`).
- `VersionController.getVersions` is a `GET /api/public/versions` with no body, returning a DTO with `Cache-Control` headers. Similar structure.
- No existing `/count` endpoints in the codebase.

**Trust-boundary implications:**
- Base site comes from `baseSite` query param → resolved server-side via `BaseSiteCacheService.getBaseSiteForCode()`. If the code is invalid, `BaseSiteContext` is null, and `BaseSiteQueryGenerator` adds no base-site filter — the count would span all base sites. The controller must validate that `BaseSiteContext.getCurrentBaseSite()` is non-null and return 400 if not.
- Product state hard-pin (ACTIVE + APPROVED) is enforced server-side in the query, not from client input. Clean.

### Shape B: `POST /api/public/product/count` with optional `ProductsFilterDTO` body

**Backend code path:**
- Accepts the same `ProductsFilterDTO` (or a subset) as search. Builds the same ES bool query via the existing `QueryGenerator` chain + `ProductStateQueryGenerator`, but calls `elasticsearchOperations.count()` instead of `.search()`.
- Returns `ResponseEntity<Long>` or `{ "count": <long> }`.

**Precedent:**
- Mirrors the existing `POST /api/public/product/search` shape minus the paging and product list.
- No existing POST-to-count pattern in the codebase.

**Trust-boundary implications:**
- Same as search — `PORTAL_SEARCH` mode ignores client-supplied `productStates`/`moderationStates`. Base site from `X-Base-Site` header (POST convention). Clean.
- Risk: the filter surface is wide (12 fields on `ProductsFilterDTO`). For a count-only use case, this is over-engineering. The sitemap caller uses empty filters.

### Shape C: Reuse search endpoint with `countOnly: true` flag

**Backend code path:**
- Add `boolean countOnly` to `SearchProductsDTO`. When true, the facade skips the ModelMapper loop and returns `{ "products": [], "totalNumberOfProducts": <count> }`.
- ES query still runs as a full search (`.search()`), but the response mapping is short-circuited.

**Precedent:**
- No existing boolean-flag-driven short-circuit pattern in the codebase.
- Modifies the existing DTO contract — the field would need to be Optional/nullable.

**Trust-boundary implications:**
- No new trust-boundary concern — same code path, same state enforcement. But the flag is a "mode switch" on a hot endpoint, which is a coupling smell.

### Recommendation: **Shape A**

One-sentence reasoning: the sitemap needs one number per base site with no filters, so the simplest shape — a GET with a query param that returns a `long` — is the right match; Shape B over-engineers for a caller that passes empty filters, and Shape C couples a mode switch onto a hot endpoint.

Shape A's placement should be on `PublicProductController` (`/api/public/product/count`) or a new thin controller, **not** under `ProductSearchController` (`/api/public/product/search/count`), because the count endpoint has no search semantics. Placing it on `PublicProductController` keeps the path prefix at `/api/public/product/count` and avoids accidentally inheriting the `SEARCH` rate-limit category via the `startsWith("/api/public/product/search")` match in `RateLimitFilter`.

---

## 4. Web caller context

**Caller:** `oglasino-web/app/sitemap.ts:54-78` (`getProductsPage`).

The web call posts `{ productsFilter: {}, paging: { page: 0, perPage: 1 } }` with `X-Base-Site` header. It reads only `totalNumberOfProducts` from the response. The `productsFilter` is empty — no filters are applied.

**Should filtering be expected at sitemap-generation time?**

No. The backend code confirms this:

1. `ProductStateQueryGenerator.java:52-55` hard-pins `PORTAL_SEARCH` to `ACTIVE + APPROVED`. Client-supplied state filters are ignored. This is the trust-boundary enforcement.
2. `BaseSiteQueryGenerator.java:18-22` scopes by base site from `BaseSiteContext` (populated from `X-Base-Site` header).
3. No other filter is structurally required for "how many products does this base site have."

"All products for a base site, in ACTIVE + APPROVED state, no other filter" is sufficient for sitemap generation. The product state hard-pin is applied server-side regardless of what the client sends, so the count endpoint needs only the base-site scope to reproduce the same count.

---

## 5. Caching

### Current state: `/api/public/product/search`

- **No Redis cache** on the search response. The controller, facade, and service layer have zero `@Cacheable`/`@CacheEvict` annotations. No custom Redis template usage.
- **No HTTP cache headers.** The controller returns `ResponseEntity.ok(result)` with no `Cache-Control`, `ETag`, or `Last-Modified` headers.
- Every call hits Elasticsearch directly.

### Count endpoint caching recommendation

A count-only endpoint could benefit from caching at two levels:

1. **HTTP `Cache-Control` headers.** The sitemap runs on-demand (typically daily or on-deploy). A `Cache-Control: max-age=300, public, stale-while-revalidate=86400` header (matching `ReferenceDataCacheControl.INSTANCE` used by `VersionController`) would be invisible to the sitemap but cheap for any repeat caller. The 5-minute TTL means the count is at most 5 minutes stale.

2. **Redis cache.** If desired, the count per base site could be cached in Redis with a short TTL (e.g., 5-10 minutes). The cache key would be `productCount::<baseSiteCode>`. Eviction would piggyback on the existing product create/update/delete paths (which already reindex to ES; a cache eviction there is cheap). However, for a single-caller scenario (sitemap), the Redis layer may be over-engineering — the ES count query itself is fast (metadata-only, no document fetch).

**Cost comparison:**

| Path | What it costs |
|------|--------------|
| Current (search with `perPage: 1`) | Full ES search query + deserialize 1 `ProductDocument` + ModelMapper mapping to `ProductOverviewDTO` + serialize full response DTO |
| Count via `ElasticsearchOperations.count()` | ES `_count` API call — metadata-only, no document fetch, no deserialization, no mapping |

The `_count` API is significantly cheaper than a search query even with `perPage: 1`, because it skips all document fetch, scoring, and source retrieval.

---

## 6. Tests

### Test classes that exercise `/api/public/product/search`

1. **`FirebaseAuthFilterTest.java:117`** — uses `/api/public/product/search` as a fixture path for testing `PENDING_DELETION` behavior on public routes. Tests auth filter behavior, not search logic.

2. **`CurrentLanguageFilterTest.java:47,63,142,160`** — uses `/api/public/product/search` as the request path for four tests covering X-Lang header validation on public routes. Tests language filter behavior, not search logic.

3. **`DefaultProductsFilterQueryBuilderTest.java`** — 12+ unit tests (Mockito, `@ExtendWith(MockitoExtension.class)`) covering the ES query builder. Tests trust-boundary fixes: `UserQueryGenerator` skipped for `DASHBOARD_SEARCH`, `applyRandom` short-circuit on search text / explicit sort, `DASHBOARD_SEARCH` throws without auth. Uses `ReflectionTestUtils` for field injection. Does not test the controller or HTTP layer.

**No integration tests exist for `ProductSearchController`.** No `@WebMvcTest` or `@SpringBootTest` test class targets the search controller directly.

### Test pattern for a new endpoint

The codebase has two test patterns:

**Pattern 1 — Standalone MockMvc (preferred for controller tests):**
- `ImageDeletionControllerTest.java` / `ImageTokensControllerTest.java` / `UsersControllerAdminExtensionTest.java`
- Uses `MockMvcBuilders.standaloneSetup(controller).setControllerAdvice(exceptionHandler).build()`
- Mocks the facade/service layer
- Tests HTTP verbs, status codes, response shapes, error handling

**Pattern 2 — Pure Mockito unit tests (preferred for service/query builder tests):**
- `DefaultProductsFilterQueryBuilderTest.java` / `DefaultProductServiceTest.java` / `DefaultProductServiceCreateTest.java`
- `@ExtendWith(MockitoExtension.class)` with `@Mock` dependencies
- Tests business logic, query shapes, validation

**Recommended test shape for the count endpoint:**
- A standalone MockMvc test for the controller (Pattern 1) verifying:
  - `GET /api/public/product/count?baseSite=rs` returns 200 with a count
  - Missing `baseSite` param returns 400
  - Invalid `baseSite` returns 400
  - Response shape (`long` or wrapper DTO)
  - Cache-Control headers if applied
- A Mockito unit test for the count query builder (Pattern 2) verifying the bool query has the ACTIVE + APPROVED state filters and the baseSiteId term

---

## Summary

The simplest path is **Shape A** — `GET /api/public/product/count?baseSite=<code>` on `PublicProductController`. It needs:

- A new method on `PublicProductController` (or a thin new controller)
- A count method on `ProductsSearchService` that builds the ACTIVE + APPROVED + baseSiteId query and calls `elasticsearchOperations.count()`
- Validation that `BaseSiteContext.getCurrentBaseSite()` is non-null (reject missing/invalid base site)
- Optional: `Cache-Control: max-age=300, public, stale-while-revalidate=86400` header
- A new `RateLimitFilter.categorize` entry (or no rate limit — the endpoint is cheap)
- Standalone MockMvc test + Mockito unit test for the query
