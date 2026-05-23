# Session summary

**Repo:** oglasino-backend
**Branch:** dev (current checkout — read-only session, no branch change)
**Date:** 2026-05-19
**Task:** Audit `oglasino-backend` for what it emits that affects SEO. Code is ground truth.

## Findings

Severity legend per brief: `critical` (page won't index or will mis-rank) / `high` (significant visibility loss) / `medium` (optimization gap) / `low` (cosmetic). Status legend: `correct` / `broken` / `partial` / `missing` / `n/a`.

### 1. Product and category slug semantics

- **No product slug field exists.** Product entity has no `slug`/`urlSlug` column. Fields are `productState, moderationState, owner, baseSite, topCategory, subCategory, category, region, city, currency, price, free, translations, filterValues, imageKeys, numberOfViews, numberOfFavorites, updateCount, deactivatedBySystem`.
  - `src/main/java/com/memento/tech/oglasino/entity/Product.java:8–109` (no slug field anywhere in 1–262)
  - `src/main/resources/db/migration/V1__init_schema.sql` — only Flyway migration in the repo; `grep -niE '\bslug\b' V1__init_schema.sql` returns no hits. No `slug` column on any table.
  - Status: `missing`. Severity: `critical`. Why: with no canonical slug source, every product URL is keyed off `productId` (numeric Long). Public detail fetch is `GET /api/public/product/search?productId={Long}` (see §3). Whatever slug the web layer renders into a URL is a web-side derivation; backend has no opinion on stability, uniqueness, or canonical form.

- **Product translated name is per-language.** Names live in `ProductTranslation(product, language, name, description)`. The same product carries one name per (product, language) pair, but neither row is a "slug source" — name is free-text moderated content. Renames change the row in place; no slug derived from it; no slug history.
  - `src/main/java/com/memento/tech/oglasino/entity/Product.java:64–69` (translations OneToMany)
  - `src/main/java/com/memento/tech/oglasino/entity/ProductTranslation.java`
  - Status: `n/a` (no slug derived).

- **Category has `route`, not slug.** `Category.route` is a path-segment string seeded from `catalogJSON/categories/*.json`. Indexed (not unique).
  - `src/main/java/com/memento/tech/oglasino/entity/Category.java:20–24` — `@Column(length = 255) private String route;`
  - `src/main/java/com/memento/tech/oglasino/entity/Category.java:10–14` — `@Index(name = "idx_category_route", columnList = "route")` (non-unique)
  - Status: `partial`. Severity: `high`. Why: category URLs do have a stable per-category path segment (`route`), but uniqueness is not DB-enforced — only `Category.key` is `unique = true` (line 17). Two categories with the same `route` under different parents are allowed by the schema; uniqueness is implicit in the JSON seed, not enforced.

- **Catalog entity has no slug.** Catalog is a per-base-site container; route information lives on Category.
  - `src/main/java/com/memento/tech/oglasino/entity/Catalog.java` — fields: `id, name, baseSite, topFilters, currentVersion`. No route/slug field.
  - Status: `n/a`.

- **Product "catalog route" is a concatenation of three category routes.** ES indexer computes `topCategory.route + subCategory.route + category.route` at index time and stores it in `ProductDocument.catalogRoute`. Returned verbatim in `ProductDetailsDTO.catalogRoute` and `SearchProductData.catalogRoute`.
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/DocumentProductConverter.java:77–82`
    ```
    var productCatalogRoute =
        source.getTopCategory().getRoute()
            + source.getSubCategory().getRoute()
            + source.getCategory().getRoute();
    destination.setCatalogRoute(productCatalogRoute);
    ```
  - `src/main/java/com/memento/tech/oglasino/dto/ProductDetailsDTO.java:11` — `private String catalogRoute;`
  - `src/main/java/com/memento/tech/oglasino/dto/SearchProductData.java:7` — `private String catalogRoute;`
  - Status: `partial`. Severity: `high`. Why: this is the SEO-relevant URL prefix for a product, but (a) it is computed at index time and only refreshed when the product is re-indexed, and (b) if any of the three category routes is renamed, every indexed product still carries the old concatenation until the next reindex — no automatic re-emission. There is also no record of the previous concatenation, so a category rename silently 404s old URLs.

- **Slug stability across edits.** Public product detail is fetched by numeric `productId`, never by slug. A product rename has no effect on its `productId` URL key. Category renames change `catalogRoute` only after reindex; product IDs persist.
  - `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java:34–45` — `@GetMapping public ResponseEntity<ProductDetailsDTO> getProductDetails(@RequestParam @NotNull Long productId)`
  - Status: `correct` (product IDs are stable). Severity: `low` (in isolation); becomes `high` if web layer uses name-derived slugs in the URL without a server-side redirect for changed slugs.

### 2. Redirect semantics

- **No 301/302/410/308/307 status codes returned anywhere in the codebase.** Grepped `MOVED_PERMANENTLY|HttpStatus\.MOVED|HttpStatus\.FOUND|HttpStatus\.SEE_OTHER|HttpStatus\.GONE|HttpStatus\.PERMANENT_REDIRECT|HttpStatus\.TEMPORARY_REDIRECT|\b301\b|\b302\b|\b410\b|\b307\b|\b308\b` across `src/main/**/*.java` — zero hits. (The two `301`/`302` hits found were unrelated: `SnowflakeUtil.java:33` "Clock moved backwards" and `EsIndexerController.java:26` "endpoint moved from GET to POST" — both comments.)
  - Status: `missing`. Severity: `high`. Why: there is no server-side redirect path for renamed categories, renamed products (if name-derived slugs are used by web), or moved content. If web persists a name-derived slug after a rename, the old URL has no recovery path.

- **No "previous slug" / "alias slug" / slug-history concept.** Grepped `alias|canonical|previous_slug|previousSlug|rename|slugHistory` across `src/main` — only hits are in unrelated contexts (Elasticsearch alias `products`, ImagePaths canonical extension, `gpgsign=false` etc.). No product- or category-slug history.
  - Status: `missing`. Severity: `high`.

- **Deleted / banned / inactive product URL behavior.** A product in any state other than `(productState=ACTIVE, moderationState=APPROVED)` is filtered out of public ES queries (see §3). Result: `ProductsSearchFacade.getProductDetailsForId` returns `null`; the controller emits **404** via `ResponseEntity.notFound()`.
  - `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java:40–42`
    ```
    if (Objects.isNull(result)) {
      return ResponseEntity.notFound().build();
    }
    ```
  - Status: `partial`. Severity: `medium`. Why: 404 is acceptable for "not found"; **410 Gone** would be a stronger signal for a previously-indexed product that has been deleted/banned (Google treats 410 as a faster deindex than 404). Backend cannot distinguish "never existed" from "existed and was removed" — both go through the same null path.

### 3. Status codes for SEO-relevant paths

- **Public single-product fetch:** `GET /api/public/product/search?productId={id}` returns **200** on hit, **404** on miss (whether the product doesn't exist or doesn't satisfy `ACTIVE + APPROVED`).
  - `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java:34–45`
  - Status: `correct` for the 200 path; `partial` for the 404 path (no 410 distinction).

- **PORTAL_SEARCH filter:** the public search mode injects two mandatory ES filters; nothing else can leak through.
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/ProductStateQueryGenerator.java:34–38`
    ```
    case PORTAL_SEARCH ->
        queryBuilder
            .filter(f -> f.term(t -> t.field("productState").value("ACTIVE")))
            .filter(f -> f.term(t -> t.field("moderationState").value("APPROVED")));
    ```
  - `ProductState` enum: `ACTIVE, INACTIVE, DELETED` (`src/main/java/com/memento/tech/oglasino/entity/ProductState.java:3–6`).
  - `ModerationState` enum: `APPROVED, BANNED` (`src/main/java/com/memento/tech/oglasino/entity/ModerationState.java:3–5`).
  - Status: `correct`. Severity: `n/a`.

- **No expiry/expired state.** There is no `EXPIRED` enum value, no expiry timestamp on Product, no scheduled-deactivation job that flips `productState` based on time. "Expired" as a concept does not exist in the backend; the brief's mention of expired products has no corresponding state.
  - Status: `n/a` (the concept is absent). Severity: `low` if web layer renders an "expired" badge from elsewhere — flag as cross-repo seam (see "For Mastermind").

- **Category-slug resolution failure path.** There is no public endpoint that resolves a category route to a category. The catalog tree is delivered up-front, embedded in `BaseSiteDTO.catalog` via `GET /api/public/baseSite/{baseSiteCode}` (see §8). Web resolves category routes to category IDs locally against this tree. Backend never returns 404 for an unknown category route because backend never receives the route.
  - `src/main/java/com/memento/tech/oglasino/dto/BaseSiteDTO.java:14` — `private CatalogDTO catalog;`
  - `src/main/java/com/memento/tech/oglasino/dto/CatalogDTO.java:7` — `private List<CategoryDTO> categories;`
  - `src/main/java/com/memento/tech/oglasino/dto/CategoryDTO.java:12` — `private List<CategoryDTO> subcategories;`
  - Status: `n/a` (delegated to web). Severity: `n/a` for backend; flag as seam in "For Mastermind."

- **`GlobalExceptionHandler` mapping.** Unhandled `NoResourceFoundException` → 404; uncaught generic `Exception` → 500. No SEO-aware mapping (no 410, no 308 for renamed resources).
  - `src/main/java/com/memento/tech/oglasino/exception/GlobalExceptionHandler.java:55–59` (404), `142–147` (500).
  - Status: `correct` for routing failures; `missing` for SEO-aware mappings (410/308).

### 4. Sitemap data endpoints

- **No sitemap endpoint, no robots.txt endpoint.** Grepped `sitemap|robots\.txt` across `src/main/**/*.java` — zero hits.
  - Status: `missing`. Severity: `high`. Why: a real sitemap requires a backend endpoint that emits all indexable product URLs with their last-modified timestamps. Today there is no such thing.

- **No `lastModified` exposure on public product/category data.** `ProductDocument` and `ProductDetailsDTO` only carry `createdAt`, not `updatedAt`/`lastModified`.
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/documents/ProductDocument.java:36–37`
    ```
    @Field(type = FieldType.Date, format = DateFormat.date_time)
    private Instant createdAt;
    ```
  - `src/main/java/com/memento/tech/oglasino/dto/ProductDetailsDTO.java:12` — `private String createdAt;` (no `updatedAt`).
  - `src/main/java/com/memento/tech/oglasino/dto/ProductOverviewDTO.java:7–21` — no `updatedAt`.
  - Status: `missing`. Severity: `high`. Why: search engines use last-modified to prioritize recrawl; absent freshness signal pessimizes recrawl frequency.

- **`updatedAt` exists on every entity but is not propagated to ES or public DTOs.** `BaseEntity.updatedAt` is maintained via `@UpdateTimestamp`. It is not copied into `ProductDocument` and not surfaced anywhere on the public API.
  - `src/main/java/com/memento/tech/oglasino/entity/BaseEntity.java:30` — `@UpdateTimestamp private LocalDateTime updatedAt;`
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/DocumentProductConverter.java:75` — `destination.setCreatedAt(source.getCreatedAt().toInstant(ZoneOffset.UTC));` (no `setUpdatedAt`).
  - Status: `partial` (data exists; surface absent). Severity: `high`.

- **No bulk product-listing endpoint suitable for sitemap generation.** `POST /api/public/product/search` is paginated, requires a `SearchProductsDTO` body, and is intended for user search. There is no `GET /api/public/products/sitemap` or similar. To produce a sitemap, web today must page through the search endpoint, which has no total-count discipline tuned for sitemaps and no last-modified field on the page items.
  - `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java:57–68`
  - `src/main/java/com/memento/tech/oglasino/dto/ProductOverviewsDataDTO.java:5–8` — `products` + `totalNumberOfProducts`. Pageable but heavy for sitemap.
  - Status: `missing` (dedicated endpoint). Severity: `high`.

### 5. `BACKEND_TRANSLATIONS` namespace and SEO-visible text

- **`BACKEND_TRANSLATIONS` is defined and indexed, but is not consumed by any public response on the SEO surface.** The namespace serves push-notification / email bodies — strings the backend emits directly to a user (per conventions Part 6). It is not used for product/category page text.
  - `src/main/java/com/memento/tech/oglasino/entity/TranslationNamespace.java:38–39` — enum value `BACKEND_TRANSLATIONS,`.
  - `src/main/java/com/memento/tech/oglasino/service/impl/DefaultTranslationService.java:42` — `private Map<String, Map<String, String>> backendTranslations;`
  - `src/main/java/com/memento/tech/oglasino/service/impl/DefaultTranslationService.java:265–276` — `indexBackendTranslations()` builds the in-memory map from `TranslationNamespace.BACKEND_TRANSLATIONS` rows.
  - `src/main/java/com/memento/tech/oglasino/service/impl/DefaultTranslationService.java:199–203` — `getBackendTranslation(code, langCode)` is the only consumer-facing accessor.
  - Consumer search: `grep -rn 'getBackendTranslation' src/main` shows callers in notification services only (push bodies). Not in product/category response pipeline.
  - Status: `correct` (namespace contains only out-of-band copy). Severity: `n/a` for SEO directly.

- **What public product responses do emit as text vs. as keys:**
  - **Text (translated, server-side):** `ProductOverviewDTO.name`, `ProductDetailsDTO.description`. These are translated server-side from `ProductTranslation` rows via `LanguageContext`.
    - `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/ProductOverviewConverter.java:56–59`
      ```
      var nameTranslated =
          source.getTranslatedValue(
              source.getNameTranslations(), languageContext.getCurrentLanguageCode(), "PRODUCT NAME");
      destination.setName(nameTranslated);
      ```
    - Description follows the same translation pattern (see `DocumentProductConverter.java:64–73` for the indexer side).
    - Status: `correct`. Severity: `n/a`.
  - **Keys (NOT translated):** `cityLabelKey`, `regionLabelKey`, category IDs only (no name/label). Web must translate label keys client-side.
    - `src/main/java/com/memento/tech/oglasino/dto/ProductDetailsDTO.java:7` — `private String regionLabelKey;`
    - `src/main/java/com/memento/tech/oglasino/dto/ProductOverviewDTO.java:14` — `private String cityLabelKey;`
    - Status: `partial`. Severity: `medium`. Why: SSR can resolve labelKeys via `/api/public/translations`, but it's an extra round-trip and creates a dependency where missing translations leak raw keys into rendered HTML.
  - **Raw enum values:** `productState` and `moderationState` are returned as raw enum names (`ACTIVE`, `APPROVED`). Not directly SEO-visible if web ignores them on detail pages, but if web ever renders them, they'll appear as untranslated tokens.
    - `src/main/java/com/memento/tech/oglasino/dto/ProductOverviewDTO.java:8–9`.
    - Status: `correct` for current intent; `potential trap` if web ever surfaces them.

- **`METADATA` namespace consumed only by web.** `TranslationNamespace.METADATA` (line 36) is meant for page titles, descriptions, OG tags per conventions. Grepped `TranslationNamespace\.METADATA` across `src/main/**/*.java` — zero hits. Backend never reads METADATA; it only seeds the rows and exposes them via `/api/public/translations`.
  - Status: `correct` (by design — separation of concerns). Severity: `n/a` for backend.

### 6. `Last-Modified` and `ETag` headers

- **Neither header is set on any response.** Grepped `Last-Modified|setLastModified|HttpHeaders\.LAST_MODIFIED|ETag|setETag|HttpHeaders\.ETAG|eTag\(|lastModified\(` across `src/main/**/*.java`. Only hits are in `DefaultR2Service.java:181, 191` — and those are inbound: the R2 SDK's `HeadObjectResponse.lastModified()` method, used to read R2 object metadata, not to set an outgoing HTTP header.
  - `src/main/java/com/memento/tech/oglasino/images/service/impl/DefaultR2Service.java:181` — `resp.lastModified()` (inbound R2 metadata).
  - Status: `missing`. Severity: `high`. Why: with no `Last-Modified` or `ETag`, conditional requests (`If-Modified-Since`, `If-None-Match`) cannot return 304; every recrawl is a full body fetch. Combined with the missing freshness signal (§4), this leaves the SEO crawler with neither sitemap freshness nor per-response freshness.

- **`Cache-Control` is set on reference-data endpoints only.** `/api/public/baseSite/**`, `/api/public/translations`, `/api/public/config` carry `CacheControl.maxAge(5min).cachePublic().staleWhileRevalidate(1d)`. Product endpoints (`/api/public/product/**`, `/api/public/product/search/**`) carry no `Cache-Control`.
  - `src/main/java/com/memento/tech/oglasino/controller/BaseSiteController.java:29–32`
  - `src/main/java/com/memento/tech/oglasino/controller/TranslationsController.java:23–26`
  - `src/main/java/com/memento/tech/oglasino/controller/ConfigurationController.java:19–22`
  - `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java` — no `cacheControl(...)` call anywhere.
  - `src/main/java/com/memento/tech/oglasino/controller/PublicProductController.java` — explicit `/* TO DYNAMIC TO CACHE */` comment at line 27 above `views/{productId}`. No `Cache-Control` set.
  - Status: `partial`. Severity: `medium`. Why: edge caching for product detail responses is implicit (Cloudflare default for missing headers, depending on its origin rule), and SSR cache behavior for SEO crawlers depends on what web does. Backend does not state an opinion.

- **No filter or interceptor sets `Last-Modified`/`ETag` globally.** Filters under `filter/` are `BaseSiteFilter`, `CurrentLanguageFilter`, `RateLimitFilter`, `FirebaseAuthFilter` — none manipulates response cache headers. No `OncePerRequestFilter` sets `Last-Modified`.
  - Status: `missing`. Severity: `high`.

### 7. Image URL semantics

- **Backend returns R2 keys (relative paths), not full URLs.** Public DTOs carry `imageKeys` (a `Set<String>`) and `topImageKey` (a `String`). Both are raw R2 keys with the bucket-internal prefix, e.g. `public/products/9f3e1c20-….jpg`.
  - `src/main/java/com/memento/tech/oglasino/entity/Product.java:79–88` — javadoc on `imageKeys` says "R2 keys for product photos, full path including prefix — e.g. `public/products/9f3e1c20-….jpg`. Returned verbatim by `POST /api/secure/images/upload-tokens` and stored as-is."
  - `src/main/java/com/memento/tech/oglasino/dto/ProductDetailsDTO.java:10` — `private Set<String> imageKeys;`
  - `src/main/java/com/memento/tech/oglasino/dto/ProductOverviewDTO.java:18` — `private String topImageKey;`
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/ProductOverviewConverter.java:61–63` — `var topImageKey = CollectionUtils.emptyIfNull(source.getImageKeys()).stream().findFirst().orElse("");` (note: first-by-iteration of a `Set` — not deterministic; see §7 follow-up).
  - `src/main/java/com/memento/tech/oglasino/images/path/ImagePaths.java:14–15` — `public static final String PUBLIC_PRODUCTS = "public/products/";`
  - Status: `partial`. Severity: `high` for `og:image` correctness. Why: web must prepend the CDN base (Cloudflare R2 public bucket / Cloudflare image transformations / equivalent) to produce a fetchable absolute URL. Backend is opinion-free on what that base is. This is a cross-repo seam — see "For Mastermind".

- **`imageKey` is stable for the lifetime of an image.** New uploads get new UUID keys; existing keys are not regenerated on product edit. Image deletion removes the key from the product's `imageKeys` set and (eventually, via the image-deletion job) deletes the R2 object.
  - `src/main/java/com/memento/tech/oglasino/images/path/ImagePaths.java:33–36` — `productKey(uuid, ext)` is the only key-construction site; UUID is generated once at upload-token time.
  - `src/main/java/com/memento/tech/oglasino/entity/Product.java:85–88` — `@ElementCollection ... product_images` join table; rows are added/removed, never regenerated.
  - Status: `correct`. Severity: `n/a`.

- **`topImageKey` selection is non-deterministic.** `getImageKeys()` returns a `Set<String>` (entity field is `Set<String>`, the join table has no ordering column), and the converter takes the first iteration element. Two indexings of the same product can pick different `topImageKey`s. Web's `og:image` picks the first of `imageKeys` if it doesn't trust `topImageKey`, but the backend itself doesn't preserve order.
  - `src/main/java/com/memento/tech/oglasino/entity/Product.java:85–88` — `Set<String> imageKeys` (no `@OrderColumn` / `@OrderBy`).
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/ProductOverviewConverter.java:61–63` — `findFirst()` over a `Set`.
  - Status: `broken`. Severity: `medium`. Why: an inconsistent `og:image` across crawls/renders can hurt CTR and rich-result eligibility. Even within one render, the selected image is whichever the JVM/HashSet returns first.

### 8. Public vs secure endpoint shapes for SEO consumers

Inventory of every `/api/public/**` endpoint and its SEO relevance:

| Endpoint | Method | DTO | SEO surface? |
| --- | --- | --- | --- |
| `/api/public/baseSite/{code}` | GET | `BaseSiteDTO` (incl. nested `CatalogDTO`) | yes — base-site bootstrap, includes full category tree |
| `/api/public/baseSite/details` | GET | `List<BaseSiteDTO>` | yes — bulk variant |
| `/api/public/baseSite/overviews` | GET | `List<BaseSiteOverviewDTO>` | yes — for site switcher |
| `/api/public/baseSite/regions/{code}` | GET | `List<RegionDTO>` | yes — region pages |
| `/api/public/product/search?productId={id}` | GET | `ProductDetailsDTO` | **yes — primary product page** |
| `/api/public/product/search` | POST | `ProductOverviewsDataDTO` | yes — listing/search pages |
| `/api/public/product/search/autocomplete` | POST | `List<SearchProductData>` | n/a (typeahead, not crawlable) |
| `/api/public/product/seen/{id}` | GET | `void` | n/a (telemetry) |
| `/api/public/product/views/{id}` | GET | `Long` | n/a (counter) |
| `/api/public/user?id={id}` | GET | `UserInfoDTO` | yes — seller profile pages |
| `/api/public/review?page={i}&productId={id}` | GET | `PageDTO<ReviewDTO>` | yes — review section |
| `/api/public/translations?namespace=&lang=` | GET | `List<TranslationDTO>` | yes — text content for SSR |
| `/api/public/config` | GET | `Map<String,String>` | yes (bootstrap) |
| `/api/public/maintenance/active` | GET | maintenance state | n/a |
| `/api/public/verify-recaptcha` | POST | — | n/a |
| `/api/public/health/check` | GET | — | n/a |
| `/api/public/app/version` | GET | app version (admin/internal) | n/a |
| `/api/public/filter_gen/init` | — | test controller (`controller/test/TestCreateJSON`) | n/a — should not be exposed in prod |
| `/api/public/notification/test` | — | test controller (`controller/test/NotificationsControllerTest`) | n/a — should not be exposed in prod |

Per-endpoint completeness for SEO:

- **`GET /api/public/product/search?productId={id}` (single product detail).** Returns `ProductDetailsDTO extends ProductOverviewDTO`. Combined fields (from `ProductDetailsDTO.java` + `ProductOverviewDTO.java`):
  - `productState, moderationState, id, ownerId, baseSiteOverview, name, cityLabelKey, price, currency, free, topImageKey, isFavorite, condition` (parent)
  - `regionLabelKey, description, imageKeys, catalogRoute, createdAt, numberOfFavorites, topCategoryId, subCategoryId, finalCategoryId, availability, deliveries, otherFilters` (child)
  - **Sufficient for:** title (`name`), description (`description`), price (`price` + `currency`), free flag (`free`), OG image (`topImageKey` / `imageKeys`), URL prefix (`catalogRoute`), seller link (`ownerId`).
  - **Missing for full schema.org `Product` + `Offer`:** `availability` is a filter id, not `https://schema.org/InStock`; `condition` is a filter id, not `https://schema.org/NewCondition`; no `gtin`/`mpn`; no `aggregateRating` summary (must call `/api/public/review` separately and aggregate); no seller name/rating embedded (must call `/api/public/user`).
  - **Missing for breadcrumbs:** only category IDs are returned, no labelKeys, no parent chain. Web must walk its locally-cached catalog tree (from `BaseSiteDTO.catalog`) and resolve labelKeys via `/api/public/translations`.
  - Status: `partial`. Severity: `high`. Why: producing a complete `Product` + `BreadcrumbList` schema.org JSON-LD requires three API calls (`product/search` + `baseSite` + `translations`) and one client-side join. SSR latency and correctness both suffer.

- **`POST /api/public/product/search` (listing/search).** Returns `ProductOverviewsDataDTO { products: List<ProductOverviewDTO>, totalNumberOfProducts: long }`. Same per-product shape as the single endpoint, minus the child-only fields. Sufficient for listing-card metadata; no breadcrumb / category names.
  - `src/main/java/com/memento/tech/oglasino/dto/ProductOverviewsDataDTO.java:5–8`
  - Status: `partial`. Severity: `medium`.

- **`POST /api/public/product/search/autocomplete`.** Returns `List<SearchProductData { id, name, catalogRoute }>`. No image, no price, no state. Fine for typeahead, not SEO-relevant.
  - `src/main/java/com/memento/tech/oglasino/dto/SearchProductData.java:5–7`.
  - Status: `correct` for its purpose. Severity: `n/a`.

- **`GET /api/public/user?id={id}` (seller profile).** Returns `UserInfoDTO`. Fields: `id, firebaseUid, displayName, profileImageKey, shortBio, rating, isVerified, activeProducts, iamActive, allowPhoneCalling, isFollowingCurrent, baseSiteOverview, regionAndCity, state, scheduledDeletionAt`.
  - Sufficient for a Person/Organization-like schema: `displayName, profileImageKey, shortBio, rating, isVerified`.
  - **Trust-boundary note:** `iamActive` and `isFollowingCurrent` are caller-context fields; both default to `false` from the cache and are populated by `DefaultUserFacade.applyCallerContext` after lookup. For unauthenticated SSR requests these will be `false`, which is correct — but if the web layer SSR-fetches with an authenticated cookie, the rendered page may differ per-user, which is hostile to caching. (Out of scope for this brief, flagged in adjacent observations.)
  - `src/main/java/com/memento/tech/oglasino/dto/UserInfoDTO.java:26–50`
  - Status: `correct` for SEO data; `partial` for SSR cache discipline. Severity: `medium`.

- **`GET /api/public/review?page=&productId=&userId=` (reviews).** Returns `PageDTO<ReviewDTO>`. `ReviewDTO { id, reviewer, reviewedProduct, comment, rating, imageKeys }`. Sufficient for a per-review `Review` schema; no `aggregateRating` summary returned in a single call.
  - `src/main/java/com/memento/tech/oglasino/controller/PublicReviewController.java:22–34` — page size hardcoded to 20, sorted by `createdAt ASC`.
  - `src/main/java/com/memento/tech/oglasino/dto/ReviewDTO.java:5–11`.
  - Status: `partial`. Severity: `medium`. Why: schema.org `aggregateRating` is best served by a single call; today web must request page 0 and either trust 20 reviews' average or page until exhausted to compute the true average rating. No dedicated `/api/public/review/summary?productId={id}` endpoint exists.

### 9. Category tree and breadcrumb data

- **Full category tree is delivered up-front via base-site bootstrap.** `BaseSiteDTO.catalog: CatalogDTO` contains `categories: List<CategoryDTO>`; each `CategoryDTO` has `subcategories: List<CategoryDTO>`. Recursion means the entire tree is in one payload, keyed by `labelKey` (translation key) and `route`.
  - `src/main/java/com/memento/tech/oglasino/dto/BaseSiteDTO.java:14` — `private CatalogDTO catalog;`
  - `src/main/java/com/memento/tech/oglasino/dto/CatalogDTO.java:7` — `private List<CategoryDTO> categories;`
  - `src/main/java/com/memento/tech/oglasino/dto/CategoryDTO.java:6–14` — fields `id, labelKey, route, iconId, subcategories, filters, freeZone`.
  - Status: `partial`. Severity: `medium`.

- **A product response does NOT include the ancestor chain.** `ProductDetailsDTO` returns `topCategoryId, subCategoryId, finalCategoryId` — three IDs — and the concatenated `catalogRoute` string. It does NOT return category labelKeys, names, or parent objects.
  - `src/main/java/com/memento/tech/oglasino/dto/ProductDetailsDTO.java:15–17`
  - To produce a `BreadcrumbList`, web must (a) read its locally-cached `BaseSiteDTO.catalog`, (b) find each of the three category IDs in the tree, (c) collect their `labelKey`s, (d) translate each via `/api/public/translations` (METADATA or category labels namespace).
  - Status: `partial`. Severity: `high`. Why: SSR-rendered breadcrumb JSON-LD depends on a multi-step client-side join. If web's category-tree cache is missing or stale, breadcrumb JSON-LD is wrong.

- **No `CategoryAncestor` table or `CategoryTreeService` for per-product ancestor lookup.** Ancestors are implicit in `CatalogCategoryAssignment.parent_id` (admin-side data) and `Product.{topCategory,subCategory,category}` (three explicit columns). There is no public endpoint returning ancestors for a given category ID.
  - `src/main/java/com/memento/tech/oglasino/entity/CatalogCategoryAssignment.java` — `parent` field for admin tree traversal.
  - Status: `missing` for public endpoint. Severity: `medium`.

### 10. Free-zone semantics

- **`freeZone` is a flag on `Category`.** All categories under a free-zone top-category are part of the free zone; the flag is on the category, not the product.
  - `src/main/java/com/memento/tech/oglasino/entity/Category.java:29–30` — `@Column(nullable = false, columnDefinition = "boolean default false") private boolean freeZone;`
  - `src/main/java/com/memento/tech/oglasino/dto/CategoryDTO.java:14, 64–70` — `freeZone` is exposed on each `CategoryDTO` returned via base-site bootstrap.
  - Status: `correct`. Severity: `n/a`.

- **A product's `free` flag (in DTOs) is derived from its top-category's `freeZone`, not from its price.** This is a critical semantics gotcha:
  - `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/DocumentProductConverter.java:95` — `destination.setFree(source.getTopCategory().isFreeZone());`
  - i.e. `ProductDocument.free` and therefore `ProductOverviewDTO.free` are **category-membership-derived**, not price-derived.
  - The entity `Product.free` field (line 62 in Product.java) is a separate column, intended for product-level "this is a giveaway" semantics — **but the DocumentProductConverter overwrites the DTO-side `free` with the category flag**, so the entity-level `Product.free` is invisible on public responses.
  - Status: `partial / broken-as-named`. Severity: `medium`. Why: a field called `free` on the public API actually means "is in a free-zone category," not "the price is zero." A web layer that uses `free` to choose between `https://schema.org/Offer` (price ≥ 0) and a non-`Offer` schema may misclassify products that are in a free-zone category but happen to carry a non-zero price (or vice versa).

- **Free-zone products are served from the same endpoints with the same DTO shape.** No `/api/public/free-zone/**` endpoints exist; no separate DTO; the only differentiator is the `free` boolean and the membership in a `freeZone == true` top-category in the catalog tree.
  - Status: `correct` (uniformity is the right call). Severity: `n/a`.

- **`Product.price` is nullable and is decoupled from `Product.free`.** `Product.price BigDecimal` (nullable), `Product.free boolean` (entity-level, not exposed on public API per above). Free-zone products may carry a `price` value, a null price, or a zero price — the schema does not enforce any of these.
  - `src/main/java/com/memento/tech/oglasino/entity/Product.java:58–62`.
  - Status: `partial`. Severity: `medium`. Why: schema.org `Offer.price` requires a numeric. If web SSR'd a free-zone product page as an `Offer` with a null price, structured-data validators flag it.

## Files touched

- none (read-only audit)

## Tests

- n/a

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code touched.
- Part 4a (simplicity): N/A — see structured evidence in "For Mastermind" (all three categories "nothing" per brief).
- Part 4b (adjacent observations): two items flagged in "For Mastermind".
- Part 6 (translations): confirmed — `BACKEND_TRANSLATIONS` is consumed only by the notification path, not by SEO-visible responses. `METADATA` is consumed only by the web frontend.
- Part 7 (error contract): touched indirectly — `GlobalExceptionHandler` 404/500 mapping reviewed, conforms to the contract.
- Part 11 (trust boundaries): noted — `UserInfoDTO.iamActive` / `isFollowingCurrent` are caller-context fields populated post-cache, correctly read from `SecurityContextHolder` upstream.

## Known gaps / TODOs

- none

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — read-only audit.
  - Simplified or removed: nothing — read-only audit.

- **Cross-repo seams discovered (per brief):**

  1. **Backend assumes web constructs absolute image URLs.** Backend returns R2 keys verbatim (`public/products/{uuid}.{ext}`). Backend has no opinion on the CDN base URL, signing scheme, or transformation pipeline (Cloudflare Images, R2 public, etc.). For `og:image` correctness, the web layer must own a stable, absolute URL contract that mirrors what crawlers fetch. If web changes its CDN base, no backend code changes — but every previously-rendered `og:image` URL silently 404s. *Assumption: web prepends a fixed Cloudflare R2 / CDN base.* Severity for SEO: high.
  2. **Backend assumes web walks the category tree for breadcrumbs.** `ProductDetailsDTO` returns three category IDs (`topCategoryId, subCategoryId, finalCategoryId`) but no labelKeys, names, or parent objects. Web must hold a `CatalogDTO` from `/api/public/baseSite` and join locally to build `BreadcrumbList` JSON-LD. *Assumption: web SSR fetches `/api/public/baseSite/{code}` before rendering any product/category page, and caches the tree.* If web doesn't, breadcrumb JSON-LD will be empty or wrong. Severity for SEO: high.
  3. **Backend assumes web resolves category routes locally.** There is no `GET /api/public/category/by-route?...` endpoint. A user typing a category URL hits web first, web resolves the route against its tree, web makes a search call with the resolved category IDs. *Assumption: web owns route → category-ID resolution and 404s unknown routes itself.* If web is missing or stale, the backend has no path to return a meaningful 404 for an unknown category route. Severity for SEO: medium.
  4. **Backend's `catalogRoute` may go stale after a category rename without a reindex.** `ProductDocument.catalogRoute` is concatenated at index time. A category-route edit in admin requires a full ES reindex (or per-product re-index of affected products) before public detail pages reflect the new URL. Until then, web receives the old `catalogRoute` for old products and the new `catalogRoute` for newly-indexed products — silent inconsistency. *Assumption: admin/sync code triggers an ES reindex on category-route changes.* If it doesn't, half the product URLs will mismatch the new category tree. Severity for SEO: high.
  5. **`free` boolean naming.** `ProductOverviewDTO.free` is derived from `topCategory.isFreeZone()`, not from `Product.price == 0`. A web layer reading `free` as "price is zero" will misclassify. *Assumption: web treats `free` as a free-zone membership flag, not a price-zero flag.* Severity: medium.
  6. **No `Last-Modified`, no `ETag`, no per-product `updatedAt`.** The web layer cannot emit a meaningful sitemap `<lastmod>` from current backend responses, and conditional crawler requests can't return 304. *Assumption: web today emits no `<lastmod>` or estimates it from `createdAt`.* If correct, recrawl frequency is suboptimal. Severity: high.
  7. **"Expired" is not a backend concept.** ProductState has only `ACTIVE | INACTIVE | DELETED`. If the product spec mentions expiry, that semantics lives somewhere else (frontend? scheduled job that hasn't shipped?). *Flag for cross-repo confirmation: does web display "expired" badges and, if so, where does the signal come from?* Severity: medium.

- **Part 4b adjacent observations (out of scope, flagged for triage):**

  - **`topImageKey` selection is non-deterministic** because `Product.imageKeys` is a `Set<String>` with no explicit ordering, and `ProductOverviewConverter` does `findFirst()` over it. Two reindexings can pick different `og:image` candidates. File: `src/main/java/com/memento/tech/oglasino/elasticsearch/converters/ProductOverviewConverter.java:61–63` and `src/main/java/com/memento/tech/oglasino/entity/Product.java:85–88`. Severity guess: medium. Did not fix — out of scope (audit is read-only).
  - **`UserInfoDTO` carries caller-context fields that vary per request** (`iamActive`, `isFollowingCurrent`). For an SSR cache that keys by `userId` alone, this leaks one user's view of another to anyone. File: `src/main/java/com/memento/tech/oglasino/dto/UserInfoDTO.java:36, 38`. Note the existing javadoc warns "the cached representation has them at default `false` on purpose" — but the response itself still varies post-cache. Severity guess: medium for SEO (small in practice if SSR is unauthenticated), high for general SSR-cache discipline. Did not fix — out of scope.
  - **`PublicReviewController.getReviews` returns reviews sorted by `createdAt ASC` with hardcoded page size 20.** SEO crawl will see the oldest reviews first on page 0; the newest reviews live on the last page. For "showcase recent reviews" SEO patterns, this is the wrong order. File: `src/main/java/com/memento/tech/oglasino/controller/PublicReviewController.java:33`. Severity guess: low. Did not fix — out of scope.
  - **Two test controllers are exposed under `/api/public/`.** `controller/test/TestCreateJSON.java` (`/api/public/filter_gen/init`) and `controller/test/NotificationsControllerTest.java` (`/api/public/notification/test`). If production routes them, they're unauthenticated test surface; if not, they shouldn't carry `@RequestMapping("/api/public/...")`. Severity guess: medium. Did not fix — out of scope, and the security implications need a separate look.
