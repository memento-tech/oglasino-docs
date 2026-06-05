# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-27
**Task:** READ-ONLY AUDIT — verify whether `PRICE_REQUIRED` enforcement exists on the `createProduct` path (issues.md 2026-05-23 entry)

## Audit answers

### Q1 — `createProduct` method body

**File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java`
**Lines:** 91–188

```java
@Override
public Product createProduct(NewProductRequestDTO newProductRequest) throws Exception {
  Objects.requireNonNull(newProductRequest);

  var user = userService.getUserById(currentUserService.getCurrentUserIdStrict()).orElseThrow();

  if (Objects.isNull(user.getBaseSite())
      || Objects.isNull(user.getRegion())
      || Objects.isNull(user.getCity())) {
    throw new ProductValidationException(ProductErrorCode.USER_SETUP_INCOMPLETE);
  }

  var product = new Product();
  product.setProductState(ProductState.ACTIVE);
  product.setModerationState(ModerationState.APPROVED);

  product.setOwner(user);
  product.setBaseSite(user.getBaseSite());
  product.setRegion(user.getRegion());
  product.setCity(user.getCity());

  updateTranslations(
      newProductRequest.getName(),
      product,
      languageContext.getCurrentUserPreferredLanguage(),
      ProductTranslation::setName);

  updateTranslations(
      newProductRequest.getDescription(),
      product,
      languageContext.getCurrentUserPreferredLanguage(),
      ProductTranslation::setDescription);

  var baseSite = getBaseSite();

  var imageKeys =
      CollectionUtils.emptyIfNull(newProductRequest.getImageKeys()).stream()
          .filter(StringUtils::isNoneBlank)
          .collect(Collectors.toSet());
  product.setImageKeys(imageKeys);

  var topCategory =
      baseSite.getCatalog().getCategories().stream()
          .filter(cat -> cat.getId().equals(newProductRequest.getTopCategory().getId()))
          .findFirst()
          .orElseThrow(
              () ->
                  new ProductValidationException(
                      "topCategory", ProductErrorCode.CATEGORY_NOT_FOUND));

  var subCategory =
      topCategory.getSubcategories().stream()
          .filter(cat -> cat.getId().equals(newProductRequest.getSubCategory().getId()))
          .findFirst()
          .orElseThrow(
              () ->
                  new ProductValidationException(
                      "subCategory", ProductErrorCode.CATEGORY_NOT_FOUND));

  var finalCategory =
      subCategory.getSubcategories().stream()
          .filter(cat -> cat.getId().equals(newProductRequest.getFinalCategory().getId()))
          .findFirst()
          .orElseThrow(
              () ->
                  new ProductValidationException(
                      "finalCategory", ProductErrorCode.CATEGORY_NOT_FOUND));

  if (!topCategory.isFreeZone()) {
    if (Objects.isNull(newProductRequest.getPrice())
        || newProductRequest.getPrice().doubleValue() <= 0) {
      throw new ProductValidationException("price", ProductErrorCode.PRICE_REQUIRED);
    }

    product.setPrice(newProductRequest.getPrice());
  } else {
    product.setPrice(BigDecimal.ZERO);
  }

  populateCategories(topCategory, subCategory, finalCategory, product);

  var allFilters = getFilters(baseSite, topCategory, subCategory, finalCategory);

  populateFilters(newProductRequest.getFilters(), allFilters, product);
  populateCurrency(newProductRequest, baseSite, product);

  var savedProduct = productRepository.save(product);

  userService.evictUserInfoCache(savedProduct.getOwner());

  applicationEventPublisher.publishEvent(new ProductUpdateEvent(savedProduct.getId()));

  return savedProduct;
}
```

**Price handling:** Lines 159–168. The `if (!topCategory.isFreeZone())` branch validates `newProductRequest.getPrice()` is non-null and > 0, throwing `PRICE_REQUIRED` on failure. The `else` branch (free-zone) sets price to `BigDecimal.ZERO`. Price is never stored verbatim from the request — it goes through this guard first.

### Q2 — `populateCategories` method body

**File:** same, **Lines:** 692–709

```java
private void populateCategories(
    CategoryDTO topCategoryDTO,
    CategoryDTO subCategoryDTO,
    CategoryDTO finalCategoryDTO,
    Product destination) {
  var topCategory = entityManager.getReference(Category.class, topCategoryDTO.getId());
  var subCategory = entityManager.getReference(Category.class, subCategoryDTO.getId());
  var category = entityManager.getReference(Category.class, finalCategoryDTO.getId());

  destination.setTopCategory(topCategory);
  destination.setSubCategory(subCategory);
  destination.setCategory(category);
  destination.setFree(topCategory.isFreeZone());

  if (topCategory.isFreeZone()) {
    destination.setPrice(BigDecimal.ZERO);
  }
}
```

**Behavior:** Sets `destination.setFree(topCategory.isFreeZone())` unconditionally at line 704. Overwrites price to `BigDecimal.ZERO` for free-zone categories at lines 706–708. Does **not** enforce non-null for non-free-zone — but that enforcement now happens upstream in `createProduct` at lines 159–162 before `populateCategories` is called at line 170.

### Q3 — `updatePriceCurrency` method body

**File:** same, **Lines:** 818–845

```java
private void updatePriceCurrency(
    UpdateProductRequestDTO source,
    BaseSiteDTO baseSite,
    CategoryDTO topCategory,
    Product destination) {
  if (topCategory.isFreeZone()) {
    destination.setFree(true);
    destination.setPrice(BigDecimal.ZERO);
    return;
  }

  if (source.getPrice() != null) {
    if (source.getPrice().compareTo(BigDecimal.ONE) < 0) {
      throw new ProductValidationException("price", ProductErrorCode.PRICE_REQUIRED);
    }
    destination.setPrice(source.getPrice());
    destination.setFree(false);
  }

  if (source.getCurrency() != null) {
    if (baseSite.getAllowedCurrencies().stream()
        .noneMatch(curr -> curr.id().equals(source.getCurrency().id()))) {
      throw new ProductValidationException("currency", ProductErrorCode.CURRENCY_NOT_ALLOWED);
    }
    destination.setCurrency(
        entityManager.getReference(Currency.class, source.getCurrency().id()));
  }
}
```

**Confirmed:** `PRICE_REQUIRED` throw at line 831 is reachable from the update path only (called at the update-path's `updateProductData` method). The guard at line 829 (`source.getPrice() != null`) means updates can omit price entirely — intentional for partial updates.

### Q4 — Search for `PRICE_REQUIRED` and price guards on the create path

**All `PRICE_REQUIRED` occurrences in `DefaultProductService.java`:**

| Line | Location |
|------|----------|
| 162 | `createProduct` — `throw new ProductValidationException("price", ProductErrorCode.PRICE_REQUIRED);` |
| 831 | `updatePriceCurrency` — `throw new ProductValidationException("price", ProductErrorCode.PRICE_REQUIRED);` |

**All `getPrice()` occurrences:**

| Line | Context |
|------|---------|
| 160 | `Objects.isNull(newProductRequest.getPrice())` — create path null check |
| 161 | `newProductRequest.getPrice().doubleValue() <= 0` — create path positivity check |
| 165 | `product.setPrice(newProductRequest.getPrice())` — create path assignment (inside guard) |
| 333 | `var persistedPrice = persisted.getPrice()` — `isNoOpUpdate` (update path) |
| 334 | `dto.getPrice() == null ? persistedPrice != null : persistedPrice == null` — `isNoOpUpdate` |
| 335 | `dto.getPrice() != null && dto.getPrice().compareTo(persistedPrice) != 0` — `isNoOpUpdate` |
| 829 | `source.getPrice() != null` — `updatePriceCurrency` |
| 830 | `source.getPrice().compareTo(BigDecimal.ONE) < 0` — `updatePriceCurrency` |
| 833 | `destination.setPrice(source.getPrice())` — `updatePriceCurrency` |

**Null checks on price in create path:** Lines 159–161 form the guard: `if (!topCategory.isFreeZone()) { if (Objects.isNull(newProductRequest.getPrice()) || newProductRequest.getPrice().doubleValue() <= 0) { throw ... } }`.

### Q5 — Create-path call graph, one level deep

Methods called directly by `createProduct` and their price-related guards:

| Line | Method | Price validation? |
|------|--------|-------------------|
| 95 | `userService.getUserById(...)` | No |
| 112, 118 | `updateTranslations(...)` | No |
| 124 | `getBaseSite()` | No |
| 170 | `populateCategories(...)` | Free-zone only: sets price to ZERO (lines 706–708). No non-free guard. |
| 172 | `getFilters(...)` | No |
| 174 | `populateFilters(...)` | No |
| 175 | `populateCurrency(...)` | Validates currency, not price (lines 807–816) |
| 177 | `productRepository.save(...)` | No |
| 183 | `userService.evictUserInfoCache(...)` | No |
| 185 | `applicationEventPublisher.publishEvent(...)` | No |

The **only** price guard on the create path is the inline block at lines 159–168 in `createProduct` itself. No helper method contributes a price guard.

### Q6 — Git log for recent changes

```
4c2c5e4 Ready for prod
498ab30 Lets role this user deletion feature......
8ff27e0 Feature validation rest
7ebec6e Previous changes from other agents.... new incoming
e2c0fa1 Product validation refactor
a0fcf9e Fully finished images pipeline on backend
a5132f2 IMAGE_PIPELINE_FINISHED
9a4a3cf Production-readiness: Flyway, rate limiting, catalog sync, CI workflows
e0a9617 Pre-lunch commit
a810d20 First commit
```

**Git blame on lines 159–168:** all attributed to commit `4c2c5e4` ("Ready for prod"), authored 2026-05-25 by Igor Stojanovic. This is two days after the issues.md entry (2026-05-23). No commit message explicitly mentions "PRICE_REQUIRED" or "price validation" — the guard was folded into the broader "Ready for prod" commit.

### Q7 — Create-path tests

**No test for `createProduct` exists.** Grep across `src/test/java/` for `createProduct`, `PRICE_REQUIRED`, and `NewProductRequest` returned one unrelated match:

- `src/test/java/com/memento/tech/oglasino/dto/UpdateProductRequestDTOTest.java:45` — asserts `UpdateProductRequestDTO` does not extend `NewProductRequestDTO`. No price logic tested.

No test case asserts that creating with `price=null, free=false` throws `PRICE_REQUIRED`.

### Q8 — Verdict

**Already fixed.** The `PRICE_REQUIRED` guard exists on the create path at `DefaultProductService.java:159–162`. It sits between the `topCategory` resolve (line 132) and the `populateCategories` call (line 170), matching the issues.md entry's recommended Option B placement. Added in commit `4c2c5e4` ("Ready for prod", 2026-05-25).

## Verdict for B5

**Already fixed.** Guard at `DefaultProductService.java:159–162`, commit `4c2c5e4` (2026-05-25).

## Implemented

- Read-only audit only. No code changes.

## Files touched

- None (read-only audit).

## Tests

- No tests run (read-only audit per brief).
- No existing test covers `createProduct` with `price=null, free=false`. See "For Mastermind" section.

## Cleanup performed

- None needed (read-only audit).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: the 2026-05-23 `PRICE_REQUIRED` entry status should flip from `open` to `fixed` with a note referencing commit `4c2c5e4` (2026-05-25). Draft below in "For Mastermind."

## Obsoleted by this session

- The 2026-05-23 issues.md entry's recommended fix (Option B) is now implemented. The entry itself is not obsoleted — it should be updated to `fixed`.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, no cleanup needed
- Part 4a (simplicity): N/A — read-only audit
- Part 4b (adjacent observations): three observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 11 (trust boundaries) — confirmed, the guard is a validation gap fix, not a trust-boundary issue. `price` is not used in moderation, authorization, or state-transition decisions.

## Known gaps / TODOs

- No test covers `createProduct` + `price=null` + non-free-zone category → `PRICE_REQUIRED`. The guard is in place but untested. See "For Mastermind."

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **issues.md status flip (draft for Docs/QA):**

  The 2026-05-23 entry "Create path missing `PRICE_REQUIRED` enforcement" should be updated:

  ```
  **Status:** fixed (2026-05-25, commit 4c2c5e4 "Ready for prod")
  ```

  Append to the entry's detail:

  > **Fix:** Guard added at `DefaultProductService.java:159–162` in commit `4c2c5e4` (2026-05-25). Non-free-zone products now require `price != null && price > 0` on the create path, throwing `PRICE_REQUIRED` (422) on failure. Free-zone products get `BigDecimal.ZERO`. Guard sits between `topCategory` resolve and `populateCategories` call, matching the recommended Option B placement.

- **Adjacent observation 1 — No test coverage for `createProduct` `PRICE_REQUIRED` guard.**
  - File: `src/test/java/` (no file — test does not exist)
  - Severity: medium
  - The guard at lines 159–162 is untested. A test asserting `price=null, free=false → PRICE_REQUIRED` and `price=0, free=false → PRICE_REQUIRED` and `price=1, free=false → passes` would lock the fix against regression. I did not write this test because the brief is read-only.

- **Adjacent observation 2 — Asymmetric price comparison between create and update paths.**
  - File: `DefaultProductService.java:161` vs `:830`
  - Severity: low
  - Create path uses `newProductRequest.getPrice().doubleValue() <= 0` (rejects 0 and negative).
  - Update path uses `source.getPrice().compareTo(BigDecimal.ONE) < 0` (rejects anything < 1).
  - Functionally: create rejects `price=0.5` as valid (passes `> 0`), update rejects `price=0.5` as invalid (fails `>= 1`). A product created with `price=0.50` cannot be updated to the same price. This is a minor asymmetry — both paths reject 0 and negative, but the boundary between 0 and 1 differs. Not a bug per se, but worth knowing about if the threshold is ever discussed.

- **Adjacent observation 3 — `populateCategories` redundantly sets price to ZERO for free-zone after `createProduct` already did.**
  - File: `DefaultProductService.java:706–708` (called from line 170, after line 167 already set `BigDecimal.ZERO`)
  - Severity: low
  - On the create path, `createProduct` sets `product.setPrice(BigDecimal.ZERO)` at line 167 for free-zone categories, then `populateCategories` sets it again at line 707. Harmless — double-set to the same value. On the update path, `populateCategories` is not called (update uses `updatePriceCurrency` instead). The redundancy is a natural consequence of `populateCategories` being shared infrastructure that doesn't know its caller already handled price. Not worth changing.
