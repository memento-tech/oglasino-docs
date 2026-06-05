# Audit — Four backend low-severity findings

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-27
**Task:** READ-ONLY AUDIT of four issues.md / Risk Watch entries (B1–B4)

---

## B1 — `TestCreateJSON.java` anonymous public endpoint

### 1. Controller quote

File: `src/main/java/com/memento/tech/oglasino/controller/test/TestCreateJSON.java:1-19`

```java
package com.memento.tech.oglasino.controller.test;

import com.memento.tech.oglasino.catalog.service.CatalogToJsonService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/public/filter_gen/init")
public class TestCreateJSON {

  @Autowired private CatalogToJsonService catalogToJsonService;

  @GetMapping
  public void createJson() throws Exception {
    catalogToJsonService.createJSONForAllBaseSites();
  }
}
```

Class-level: `@RestController`, `@RequestMapping("/api/public/filter_gen/init")`.
Method-level: `@GetMapping`, `public void createJson() throws Exception`.
Package: `com.memento.tech.oglasino.controller.test`.

### 2. What `createJSONForAllBaseSites()` does

File: `src/main/java/com/memento/tech/oglasino/catalog/service/CatalogToJsonService.java:254-266`

```java
@Transactional
public void createJSONForAllBaseSites() {
  try {
    Files.createDirectories(ROOT);
    Files.createDirectories(CATEGORY_DIR);
  } catch (Exception e) {
    throw new RuntimeException(e);
  }

  baseSiteRepository.findAll().forEach(this::generateBaseSiteJSON);

  writeIconRegistryFile(baseSiteRepository.findById(1000L).orElseThrow());
}
```

Where `ROOT = Paths.get("catalogJSON")` and `CATEGORY_DIR = ROOT.resolve("categories")` (lines 170-172).

**What "expensive" means concretely:**
- Opens a `@Transactional` database session.
- Creates directories on the local filesystem.
- Reads all base sites from Postgres (`baseSiteRepository.findAll()`).
- For each base site: loads the full catalog tree from DB (`catalogService.getEntityCatalogTree`), loads catalog category assignments, iterates every category and subcategory, writes JSON files to disk per category, writes a per-base-site catalog JSON file.
- Writes icon registry files to disk.
- Cost: multiple DB round-trips (base sites, catalog trees, assignments), Jackson serialization for each file, filesystem I/O. Not catastrophic for a single call, but repeatable by an unauthenticated caller at arbitrary frequency.

### 3. Caller inventory

Grep of `src/main/java/` and `src/test/java/` for `createJSONForAllBaseSites`:

- `CatalogToJsonService.java:255` — the method definition itself
- `TestCreateJSON.java:17` — the sole caller

**No other Java caller exists. No test caller exists. `TestCreateJSON` is the only HTTP entry point.**

### 4. Security configuration

File: `src/main/java/com/memento/tech/oglasino/security/config/SecurityConfig.java:73-82`

```java
.authorizeHttpRequests(
    auth ->
        auth.requestMatchers(HttpMethod.OPTIONS, "/**")
            .permitAll()
            .requestMatchers("/api/public/**", "/api/auth/**", "/internal/**")
            .permitAll()
            .requestMatchers("/api/secure/**")
            .authenticated()
            .anyRequest()
            .permitAll())
```

`/api/public/filter_gen/init` matches `/api/public/**` → genuinely `permitAll`. No other mechanism gates it. The `RateLimitFilter` does not cover this path (it matches only `/api/public/verify-recaptcha`, `/api/public/suggestion`, `/api/public/product/search`, and `/api/public/product/seen`).

### 5. Dev-vs-prod existence check

The file lives in `src/main/java/` — it ships to all environments (dev, stage, prod). There is no `@Profile` annotation on the class or any method. Spring Boot's autowiring does not exclude it on any profile today. **It is active in every deployment.**

### 6. Real-world usage signal

- No comments in the file.
- No README references anywhere.
- The only sibling file in the `controller/test/` package is `TestCreateJSON.java` itself (directory listing shows one file, last modified Apr 15).
- The method writes to local filesystem paths (`catalogJSON/`) — this is a build-time/dev-time utility, not a production feature.
- No other endpoint references `filter_gen` or `catalogJSON`.

### 7. Triage option weights

- **(a) Gate behind `@Profile("dev")`:** ~2 lines of code, zero risk of breaking a real workflow in production. If Igor uses this locally, it still works. If someone else runs it in prod, it 404s.
- **(b) Move under `/api/secure/admin/**` with `@PreAuthorize("hasRole('ADMIN')")`:** ~5 lines (annotation + path change), but the method writes to local filesystem which has no role in production. Admin gating is overengineering for a filesystem utility.
- **(c) Delete entirely:** ~1 file deletion. Risk: if Igor has a local shell alias or script that hits `GET /api/public/filter_gen/init`, it breaks silently. But the catalog JSON generation logic in `CatalogToJsonService.createJSONForAllBaseSites()` remains available for any future caller.

### Verdict for B1

**Delete.** Zero evidence of legitimate use. The endpoint writes JSON files to the local filesystem (`Paths.get("catalogJSON")`), which has no role in a production deployment. The package name `controller/test` and the absence of any documentation, `@Profile` annotation, or sibling files all point to an abandoned dev utility. The underlying `CatalogToJsonService.createJSONForAllBaseSites()` method is preserved; only the unauthenticated HTTP trigger is removed.

**If Igor confirms he actively uses this endpoint in dev,** `@Profile("dev")` is the fallback (2 lines, preserves dev workflow, kills production exposure).

---

## B2 — Dead `baseCurrency` variable in `PriceQueryGenerator`

### 1. Full method body of `getPriceRangeQuery2`

File: `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/PriceQueryGenerator.java:62-98`

```java
private BoolQuery.Builder getPriceRangeQuery2(
    BoolQuery.Builder queryBuilder, PriceRangeFilterDTO priceRange) {
  if (priceRange.getFrom() == null && priceRange.getTo() == null) {
    return null;
  }

  BigDecimal fromConverted =
      priceRange.getFrom() != null
          ? baseCurrencyService.convertToBaseCurrency(
              priceRange.getSelectedCurrency().code(), priceRange.getFrom())
          : null;

  BigDecimal toConverted =
      priceRange.getTo() != null
          ? baseCurrencyService.convertToBaseCurrency(
              priceRange.getSelectedCurrency().code(), priceRange.getTo())
          : null;

  return queryBuilder.filter(
      f ->
          f.range(
              r ->
                  r.number(
                      n -> {
                        n.field("basePrice");

                        if (fromConverted != null) {
                          n.gte(fromConverted.doubleValue());
                        }

                        if (toConverted != null) {
                          n.lte(toConverted.doubleValue());
                        }

                        return n;
                      })));
}
```

No `getPriceRangeQuery1` exists.

### 2. `baseCurrency` is no longer present

**The dead variable has already been removed.** The line `var baseCurrency = baseCurrencyService.getBaseCurrency()` that the Risk Watch entry describes no longer exists in the file. Grep for `baseCurrency` in `PriceQueryGenerator.java` returns zero matches (only `baseCurrencyService` appears as a field name).

Git history confirms: commit `fc2d03d` (2026-05-17, "Resolved pom, cache and other bugs") removed the line:

```diff
-    var baseCurrency = baseCurrencyService.getBaseCurrency();
-
     BigDecimal fromConverted =
```

### 3. Callers

- `PriceQueryGenerator.addQuery:42` calls `getPriceRangeQuery2(queryBuilder, priceRange)` directly
- `PriceQueryGenerator.addBoth:55` calls `getPriceRangeQuery2(boolQuery, priceRange)` in a lambda

No external callers — `getPriceRangeQuery2` is `private`.

### 4. Side effects of the fetch (moot — call is gone)

`BaseCurrencyService.getBaseCurrency()` at `DefaultBaseCurrencyService.java:53` is `@Cacheable("redisBaseCurrency")`. The call was a Redis cache hit returning `CurrencyDTO`. No side effects beyond reading cache. Now that the call is gone, the wasted Redis hit is gone too.

### 5. No shadowing

The class declaration at `PriceQueryGenerator.java:15` has only one field: `@Autowired private BaseCurrencyService baseCurrencyService`. No `baseCurrency` field on the class or any parent.

### Verdict for B2

**Already fixed.** The dead variable was removed in commit `fc2d03d` (2026-05-17, Igor's commit "Resolved pom, cache and other bugs"). The Risk Watch entry in `state.md:326` is stale and should be closed with status `fixed`.

---

## B3 — `backendTranslations` field is non-`volatile`

### 1. Field declaration

File: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultTranslationService.java:28`

```java
@Service
public class DefaultTranslationService implements TranslationService {

  private Map<String, Map<String, String>> backendTranslations;       // line 28

  @Autowired private LanguageRepository languageRepository;            // line 30
  @Autowired private TranslationRepository translationRepository;      // line 31
  @Autowired private LanguageService languageService;                  // line 32

  @Lazy @Autowired private DefaultTranslationService self;             // line 34
  @Lazy @Autowired private VersionChecksumService versionChecksumService; // line 35
```

No `volatile`, no `final`, no `AtomicReference`. Plain mutable instance field.

**Note:** The Risk Watch entry references line 42, but the version-checksums feature (commit `59b159b`) heavily refactored this file (131 lines removed per `decisions.md`). The field is now at line 28.

### 2. Every assignment to `backendTranslations`

**Assignment 1 — `@PostConstruct` boot path:**

`DefaultTranslationService.java:37-39`

```java
@PostConstruct
public void initBackendTranslations() {
  indexBackendTranslations();
}
```

Which calls `indexBackendTranslations()` at lines 130-141:

```java
private void indexBackendTranslations() {
  backendTranslations =
      CollectionUtils.emptyIfNull(
              translationRepository.findByTranslationNamespace(
                  TranslationNamespace.BACKEND_TRANSLATIONS))
          .stream()
          .collect(
              Collectors.groupingBy(
                  Translation::getTranslationKey,
                  Collectors.toMap(
                      t -> t.getLanguage().getCode(), Translation::getTranslationValue)));
}
```

**Full reassignment** — `backendTranslations = <new map>`. Not mutation.

**Assignment 2 — admin edit path:**

`DefaultTranslationService.java:125-127` (inside `updateTranslation`):

```java
if (translationUpdateRequest.getNamespace() == TranslationNamespace.BACKEND_TRANSLATIONS) {
  indexBackendTranslations();
}
```

Same `indexBackendTranslations()` method. Same full reassignment pattern. Called from `AdminTranslationsController.updateTranslation` on an admin request thread.

**Both writes are full reassignment. No mutation (`backendTranslations.put(...)`) exists.**

### 3. Every read of `backendTranslations`

**Read site — `getBackendTranslation()`:**

`DefaultTranslationService.java:101-104`

```java
@Override
public String getBackendTranslation(String code, String langCode) {
  return Optional.ofNullable(backendTranslations.get(code))
      .map(m -> m.get(langCode))
      .orElseThrow();
}
```

**Callers (all on request threads — Tomcat worker threads):**

- `DefaultAdminReviewService.java:89,100,103,115,124,127,154,157` — 8 call sites for push notification content during admin product review
- `DefaultFavoriteProductFacade.java:109,113` — 2 call sites for push notification content when a favorited product changes

Thread context: admin review notifications and favorite-product notifications. Both run on HTTP request threads (Tomcat workers), not on the boot thread or the admin-edit thread.

### 4. Sibling fields

No other mutable fields in `DefaultTranslationService` face the same concern:
- `languageRepository`, `translationRepository`, `languageService` — injected once at construction, never reassigned
- `self` — `@Lazy` injected, never reassigned
- `versionChecksumService` — `@Lazy` injected, never reassigned

`backendTranslations` is the only field that is reassigned after construction.

### 5. `AtomicReference` precedent

One use in the codebase:

- `DefaultOpenAIFacade.java:46` — `AtomicReference<String> suggestion = new AtomicReference<>(StringUtils.EMPTY)` — local variable inside a method, not a field-level pattern.

No field-level `AtomicReference` precedent exists. No `volatile` fields exist anywhere in `src/main/java/`.

### 6. Concurrency cost

**Write path is cold:** boot (`@PostConstruct`, once) + admin edits to `BACKEND_TRANSLATIONS` namespace (rare — only when an admin changes a push notification or email translation value via the admin translations page).

**Read path is warm:** every push notification for product reviews (8 reads per notification) and every favorite-product notification (2 reads per notification). These are on request threads.

The window for JMM visibility problems: an admin edits a BACKEND_TRANSLATIONS value → `indexBackendTranslations()` runs on the admin's request thread → a different request thread reads `backendTranslations` without `volatile` → that thread may see the stale reference.

### Verdict for B3

**Add `volatile`.** One keyword, minimal change. The write path is cold (boot + rare admin edits). Full reassignment means readers see either the old map or the new — no torn map — but JMM does not guarantee visibility of the new reference without `volatile` or a happens-before relationship. No codebase precedent for `AtomicReference` at field level; `volatile` is the simpler fix for a read-heavy, write-cold field.

---

## B4 — `PORTAL_SEARCH` silently drops body `productStates` / `moderationStates`

### 1. `PORTAL_SEARCH` branch

File: `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/ProductStateQueryGenerator.java:34-51`

```java
switch (productsSearchMode) {
  case PORTAL_SEARCH ->
      queryBuilder
          .filter(f -> f.term(t -> t.field("productState").value("ACTIVE")))
          .filter(f -> f.term(t -> t.field("moderationState").value("APPROVED")));

  case DASHBOARD_SEARCH -> {
    addProductStatesFilterTerms(queryBuilder, productStates, ALLOWED_DASHBOARD_PRODUCT_STATES);
    addModerationStatesFilterTerms(queryBuilder, moderationStates);
  }

  case ADMIN_SEARCH -> {
    addProductStatesFilterTerms(queryBuilder, productStates, null);
    addModerationStatesFilterTerms(queryBuilder, moderationStates);
  }

  case null, default -> throw new IllegalStateException();
}
```

Branch selected by `switch (productsSearchMode)` at line 34. `productsSearchMode` is passed from controllers.

`productStates` and `moderationStates` are extracted from `productsFilter` at lines 26-32:

```java
var productStates =
    Optional.ofNullable(productsFilter).map(ProductsFilterDTO::getProductStates).orElse(null);

var moderationStates =
    Optional.ofNullable(productsFilter)
        .map(ProductsFilterDTO::getModerationStates)
        .orElse(null);
```

These extracted values are **never touched** in the `PORTAL_SEARCH` branch.

### 2. Shared wire DTO

File: `src/main/java/com/memento/tech/oglasino/dto/ProductsFilterDTO.java:20-21`

```java
private List<ProductState> productStates;
private List<ModerationState> moderationStates;
```

Both declared as plain `List` — nullable by default, no validation annotations, no `@JsonIgnore`. Wrapped inside `SearchProductsDTO.productsFilter`:

```java
public class SearchProductsDTO {
  private ProductsFilterDTO productsFilter;
  private PagingDTO paging;
  ...
}
```

### 3. Other branches

- **`DASHBOARD_SEARCH`** (lines 40-43): Calls `addProductStatesFilterTerms` with `ALLOWED_DASHBOARD_PRODUCT_STATES` (`Set.of(ProductState.ACTIVE, ProductState.INACTIVE)`) — intersects client-supplied states with the allowed set (clamping to ACTIVE/INACTIVE). Calls `addModerationStatesFilterTerms` passing client states through. **Client-supplied states are consulted but restricted.**

- **`ADMIN_SEARCH`** (lines 45-48): Calls `addProductStatesFilterTerms` with `null` allowed set — no restriction on client-supplied states. Calls `addModerationStatesFilterTerms` passing client states through. **Client-supplied states pass through unrestricted.**

- **`PORTAL_SEARCH`** is the **only branch** that drops the body fields entirely.

### 4. Trust boundary verification

**Confirmed: the hard-pinning IS the security mechanism.**

`PORTAL_SEARCH` branch at lines 36-38:

```java
case PORTAL_SEARCH ->
    queryBuilder
        .filter(f -> f.term(t -> t.field("productState").value("ACTIVE")))
        .filter(f -> f.term(t -> t.field("moderationState").value("APPROVED")));
```

Even if a client sends `productStates=[PENDING]` via the public product search endpoint, the server hard-pins `ACTIVE` and `APPROVED`. The client-supplied values are extracted into local variables at lines 26-32 but never read in this branch.

Additionally, in `DefaultProductsFilterQueryBuilder`, two of the three call sites (lines 38 and 65 — single-product and multi-product ID queries) pass `null` as the `productsFilter` parameter, so state fields cannot reach `ProductStateQueryGenerator` at all for those paths. The third call site (line 112 — the main search query builder) passes `productsFilter` through, but `PORTAL_SEARCH` ignores it.

### 5. Three fix options

- **(a) Add Javadoc comment:** ~3 lines. Zero risk. Documents the intentional drop for future readers. Cost: trivial.
- **(b) Split the wire DTO:** New `PortalProductsFilterDTO` without state fields, wire it through `ProductSearchController` → `ProductsSearchFacade` → `DefaultProductsFilterQueryBuilder`. Moderate work (~10 files), potential mobile/web client impact. Cost: disproportionate to the problem — the security boundary is correct, only readability is at issue.
- **(c) Reject with 400:** Add a check in the `PORTAL_SEARCH` branch or in the controller: if `productStates` or `moderationStates` is non-empty, return 400. Risk: breaks any client that sends these fields even though they're harmlessly ignored today. Would require a new error code and translation key per Part 7. Cost: moderate work + client-facing breakage risk.

### 6. Public endpoint surface

File: `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java`

- `GET /api/public/product/search` (line 34) — single product by ID, `PORTAL_SEARCH`
- `POST /api/public/product/search/autocomplete` (line 48) — autocomplete, `PORTAL_SEARCH`
- `POST /api/public/product/search` (line 57) — main product list, `PORTAL_SEARCH`

Also used internally (not HTTP):
- `DefaultFavoriteProductFacade.java:77` — fetches favorited product IDs using `PORTAL_SEARCH`

All three public endpoints are under `/api/public/product/search` and use `ProductsSearchMode.PORTAL_SEARCH`.

### Verdict for B4

**Add Javadoc comment.** The security boundary is correct — `PORTAL_SEARCH` hard-pins `ACTIVE+APPROVED` regardless of client input. The silent drop is the intentional trust-boundary enforcement. A comment explaining that these fields are intentionally ignored (and why) is the right-sized fix. Splitting the DTO or rejecting with 400 are disproportionate — the security mechanism works, only the documentation is missing.

---

## Verdict per finding

| Finding | Verdict |
|---------|---------|
| B1 — `TestCreateJSON.java` | **Delete.** Abandoned dev utility writing to local filesystem. No evidence of legitimate use. Fallback: `@Profile("dev")` if Igor confirms active use. |
| B2 — Dead `baseCurrency` variable | **Already fixed.** Removed in commit `fc2d03d` (2026-05-17). Risk Watch entry is stale → close as `fixed`. |
| B3 — `backendTranslations` non-volatile | **Add `volatile`.** One keyword. Write path is cold (boot + rare admin edits); read path is on request threads. JMM visibility not guaranteed without it. |
| B4 — `PORTAL_SEARCH` silent drop | **Add Javadoc comment.** Security boundary is correct. Only documentation is missing. |

---

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing (read-only audit)
  - Simplified or removed: nothing (read-only audit)

- **B2 Risk Watch entry is stale.** `state.md:326` ("Dead `baseCurrency` variable in `PriceQueryGenerator` (accepted-and-known)") should be closed as `fixed`. The variable was removed in commit `fc2d03d` (2026-05-17). Draft for Docs/QA: append "(CLOSED — removed in commit fc2d03d, 2026-05-17)" to the Risk Watch row, or delete the row.

- **B3 line number drift.** The Risk Watch entry at `state.md:325` references `DefaultTranslationService.java:42`. The version-checksums feature (commit `59b159b`) heavily refactored this file; the field is now at line 28. If the Risk Watch entry is updated before the fix lands, correct the line number.

- **B1 adjacent observation (Part 4b):** `CatalogToJsonService.createJSONForAllBaseSites()` writes to local filesystem paths (`Paths.get("catalogJSON")`). This is clearly a dev/build-time utility (generates JSON catalog files from DB). The method itself is fine, but its placement in `src/main/java` (not `src/test/java`) means the full catalog-generation code ships to production even if the HTTP trigger is removed. Low severity — the code is inert without a caller, but a future developer could wire it up unintentionally. Not fixing because it's out of scope.

- **B1 adjacent observation (Part 4b):** The `controller/test/` package exists under `src/main/java/` and currently contains only `TestCreateJSON.java`. If B1 is deleted, the package becomes empty and should be deleted too. If other test-utility controllers exist on other branches, consider whether the `controller/test/` pattern should be `@Profile("dev")`-gated at the package level. Low severity.

- **B4 adjacent observation (Part 4b):** Two of the three `wrapWithStateSearchMode` call sites in `DefaultProductsFilterQueryBuilder` (lines 38, 65) pass `null` for `productsFilter`. The method at `ProductStateQueryGenerator:26-32` extracts `productStates` and `moderationStates` from a `null` filter, producing `null` for both. In the `DASHBOARD_SEARCH` and `ADMIN_SEARCH` branches, this means `addProductStatesFilterTerms` is called with `null` states. For `DASHBOARD_SEARCH`, the method falls through to using the full `ALLOWED_DASHBOARD_PRODUCT_STATES` set as the filter — correct behavior for single-product lookups. For `ADMIN_SEARCH`, the method skips the filter entirely — also correct (admin sees everything). Not a bug, but the null-propagation chain is subtle. Low severity; the Javadoc comment recommended for B4 could note this.

### Config-file impact

- `conventions.md`: no change
- `decisions.md`: no change
- `state.md`: B2 Risk Watch row should be closed as `fixed` (draft above in For Mastermind)
- `issues.md`: no change (the four entries remain as-is; the B2 Risk Watch entry is in state.md, not issues.md)

## Obsoleted by this session

- The Risk Watch entry for B2 ("Dead `baseCurrency` variable in `PriceQueryGenerator`") at `state.md:326` is obsoleted by commit `fc2d03d` (2026-05-17). Left for Docs/QA to close — not deleting because this session is read-only and the entry is in a cross-repo file.

## Conventions check

- Part 4 (cleanliness): confirmed — read-only audit, no code changes
- Part 4a (simplicity): N/A — read-only audit
- Part 4b (adjacent observations): three observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 11 (trust boundaries) — confirmed for B1 (unauthenticated trigger) and B4 (portal state hard-pin)

## Cleanup performed

None needed — read-only audit.

## Known gaps / TODOs

- B1: need Igor's confirmation on whether the endpoint is actively used in dev before choosing delete vs `@Profile("dev")`.

## Implemented

- Read-only audit of four findings (B1–B4) with verbatim code quotes, caller inventories, and security configuration analysis.
- Discovered B2 is already fixed (commit `fc2d03d`, 2026-05-17).
- Confirmed B3 and B4 are still present as described (with line-number drift on B3).
- Confirmed B1 is still present and genuinely `permitAll` with no rate limiting.

## Files touched

None — read-only audit.

## Tests

None — read-only audit.
