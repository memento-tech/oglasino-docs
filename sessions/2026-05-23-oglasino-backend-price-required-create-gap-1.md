# Session summary

**Repo:** oglasino-backend
**Branch:** dev (Igor's checked-out branch at session start; brief expected `main` or `feature/qa-preparation` — read-only audit, so the branch choice does not affect findings)
**Date:** 2026-05-23
**Task:** Read-only audit. Trace the price-handling code path through product creation and decide what fix, if any, the create path needs to enforce a non-null price for non-free-zone categories.

## Implemented

- Read-only audit only. No code changes.
- Walked `DefaultProductService.createProduct`, `populateCategories`, `populateCurrency`, and `updatePriceCurrency` end-to-end. Confirmed the brief's gap.
- Counted call sites of each helper to confirm `updatePriceCurrency` is reachable only from the update path.
- Confirmed `NewProductRequestDTO.price` carries no Jakarta annotation.
- Verified `CategoryDTO.isFreeZone()` is present so Option B does not need an extra catalog lookup.
- Recommended Option B (server-side guard in `createProduct`).

## Files touched

- (none — read-only)

## Tests

- None run. Audit only.

## Cleanup performed

- None needed.

---

## Audit

### 1. Confirmation of the create-path gap

`DefaultProductService.createProduct` (`src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java`):

- Line 125: `product.setPrice(newProductRequest.getPrice());` — stores the client-supplied price verbatim. No null check, no `≥ 1` check.
- Lines 137–162: resolves `topCategory`, `subCategory`, `finalCategory` (as `CategoryDTO`) from the base-site catalog. Throws `CATEGORY_NOT_FOUND` on miss.
- Line 164: `populateCategories(topCategory, subCategory, finalCategory, product);`
- Line 169: `populateCurrency(newProductRequest, baseSite, product);`

`populateCategories` (lines 686–703):

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

- Lines 700–702: free-zone branch overwrites whatever price was set at line 125 to `BigDecimal.ZERO`.
- Non-free-zone branch: leaves the price untouched. **No `null` check, no `≥ 1` check on either side of the conditional.**

`populateCurrency` (lines 801–810): validates `CURRENCY_NOT_ALLOWED` against `baseSite.getAllowedCurrencies()`. Does not touch price.

`updatePriceCurrency` (lines 812–839):

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
  ...
}
```

- The `PRICE_REQUIRED` throw is at line 825, inside `updatePriceCurrency`.

Call-site grep across `src/`:

- `populateCategories` — called only at `DefaultProductService.java:164` (create path).
- `populateCurrency` — called only at `DefaultProductService.java:169` (create path).
- `updatePriceCurrency` — called only at `DefaultProductService.java:296` (update path).

**Confirmed:** `updatePriceCurrency` is not called from `createProduct`. The create path never runs the `PRICE_REQUIRED` enforcement.

`NewProductRequestDTO.java` (lines 13–45):

```java
private BigDecimal price;
```

No `@NotNull`, no `@DecimalMin`, no custom annotation. Jackson will deserialize a missing field or `"price": null` as `null` without complaint. The brief's call here is correct.

**Net effect:** a client posting `{"price": null, "topCategory": <non-free-zone>, …}` reaches line 125, stores `price = null`, then `populateCategories` sets `free = false` and leaves price as `null`. Product persists with `price = null, free = false`.

### 2. Confirmation of the free-zone branch

`populateCategories` lines 698 and 700–702:

- `destination.setFree(topCategory.isFreeZone());` — `Product.free` is derived from the catalog's `freeZone` flag (server-side, per spec Part 11).
- If the top category is free-zone, price is overwritten to `BigDecimal.ZERO` regardless of what the client sent.

This matches the QA audit's description verbatim. No correction needed.

### 3. Fix option A — Jakarta annotation on the DTO

Three sub-shapes:

1. Plain `@NotNull(message = "PRICE_REQUIRED")` on `NewProductRequestDTO.price` — would reject every free-zone create as well, because free-zone clients send `price` null or absent. Incorrect on its own.
2. `@NotNull` plus a class-level custom constraint (`@ValidNonFreeZonePrice` or similar) that reads `topCategory.id`, looks the category up in the catalog (`BaseSiteCacheService`), and decides whether to allow null. Pulls catalog DI into the Jakarta validator layer, which is a layering inversion (validators are stateless input checkers; catalog lookups are service-tier work).
3. A class-level custom constraint without catalog lookup — would need a `freeZone` flag on the wire. Reintroducing client-supplied `freeZone` re-opens a trust-boundary violation the spec explicitly closed (spec §"Trust boundary principles" → "`freeZone` is derived from `topCategory.isFreeZone()` server-side. Never trusted from client DTO."). Hard rule violation.

Cost summary:

- Sub-shape 1: simple but functionally broken (rejects free-zone creates).
- Sub-shape 2: ~30 LOC validator + DI of `BaseSiteCacheService` into a `ConstraintValidator`; mixes layers; doubles up with the existing `populateCategories` lookup.
- Sub-shape 3: trust-boundary regression.

**Plus a wire-contract complication for any Jakarta route:** spec §"HTTP status codes" assigns `PRICE_REQUIRED` to 422 (business rule). Jakarta constraint violations are 400 in the current handler chain. Enforcing via Jakarta would produce a 400 with `code=PRICE_REQUIRED`, contradicting the spec's status table for that code. Resolving that without breaking other rows means either special-casing this code in the global exception handler or accepting the contract drift.

### 4. Fix option B — server-side check in `createProduct`

Insert the guard between the `topCategory` resolve (line 144 — the `orElseThrow` for `CATEGORY_NOT_FOUND`) and the `populateCategories(...)` call (line 164). Brief-suggested shape:

```java
if (!topCategory.isFreeZone() && newProductRequest.getPrice() == null) {
  throw new ProductValidationException("price", ProductErrorCode.PRICE_REQUIRED);
}
```

For parity with `updatePriceCurrency`'s `< 1` enforcement (line 824) and the web client's Zod `≥ 1` rule, the simplest version that closes the full gap is:

```java
if (!topCategory.isFreeZone()
    && (newProductRequest.getPrice() == null
        || newProductRequest.getPrice().compareTo(BigDecimal.ONE) < 0)) {
  throw new ProductValidationException("price", ProductErrorCode.PRICE_REQUIRED);
}
```

Notes:

- `topCategory` at line 144 is a `CategoryDTO` (resolved from `baseSite.getCatalog().getCategories()`, which is `List<CategoryDTO>`). `CategoryDTO.isFreeZone()` exists (`CategoryDTO.java:64`), so no extra catalog lookup is needed.
- Wire contract: emits 422 with `field=price, code=PRICE_REQUIRED, translationKey=product.price.required`. Matches the spec's status table for `PRICE_REQUIRED` (422) and matches the existing update-path emission.
- The "price ≥ 1" half is identical to what `updatePriceCurrency` already enforces at line 824. The null half closes the new gap.

Cost: one ≤6-line guard in `createProduct`; one new test case in the create-path test suite (post a null price with a non-free-zone category, assert 422 + `PRICE_REQUIRED`); no DTO change, no wire-contract change, no translation change.

### 5. Fix option C — call `updatePriceCurrency` from `createProduct`

`updatePriceCurrency` (lines 812–839):

1. If free-zone: `setFree(true)`, `setPrice(ZERO)`, return.
2. If `source.getPrice() != null`: enforce `≥ 1`, `setPrice`, `setFree(false)`.
3. If `source.getCurrency() != null`: enforce `CURRENCY_NOT_ALLOWED`, `setCurrency`.

Side effects on the create path:

- **Signature mismatch.** Takes `UpdateProductRequestDTO`, not `NewProductRequestDTO`. Calling it from `createProduct` requires either (a) refactoring it to accept a common shape (extract an interface for `getPrice()` / `getCurrency()` or pass primitives), or (b) building a throwaway `UpdateProductRequestDTO` from the create request. Both expand scope beyond the gap.
- **Null price not enforced.** Step 2 is guarded by `if (source.getPrice() != null)`. If the client posts `price=null` on a non-free-zone create, the method falls through without setting price and without throwing. **Option C does not close the gap as written.** Closing it requires editing `updatePriceCurrency` itself — at which point the simpler edit is Option B.
- **Currency double-validation.** `populateCurrency` (line 169) already enforces `CURRENCY_NOT_ALLOWED` on the create path with `currency.id().equals(...)` semantics. Step 3 of `updatePriceCurrency` does the same check, wrapped in a `!= null` guard. On create, currency is `@NotNull` (handled at the Jakarta layer with `CURRENCY_REQUIRED`), so the inner null-guard is dead weight. Calling both is redundant.
- **`setFree(...)` double-fire.** `populateCategories` already sets `Product.free` from `topCategory.isFreeZone()` at line 698. `updatePriceCurrency` re-sets it inside its two branches. Two writers on one field — no correctness bug because the inputs agree, but it muddies the "who owns this field" boundary.

Cost: higher than B (refactor + the `null`-handling change), and doesn't even close the gap without that extra change. Net negative versus B.

### 6. Recommendation

**Option B**, with the `< 1` clause included for parity with the update path.

Reasoning:

- Smallest diff (≤6 lines), one new test.
- Honors the existing layering: business-rule validation lives in the service tier and emits 422 (per spec's status table).
- No DTO churn, no wire-contract churn, no translation churn.
- Matches the update path's enforcement shape, so a future reader sees one mental model on both sides.
- No layering inversion (Option A's catalog-lookup-in-validator), no scope expansion (Option C's refactor + null-handling fix).

Cost of being wrong (B):

- If a future flow legitimately needs to accept null price for a non-free-zone create (e.g. a draft-state listing), the guard would reject it with 422 `PRICE_REQUIRED`. Recoverable by removing the guard or scoping it by `ProductState`. No data-loss risk, no client breakage outside that hypothetical flow.
- Translation: the existing `product.price.required` key (per Conventions Part 6 Rule 4 mapping `PRICE_REQUIRED` → `product.price.required`) is already seeded in EN/SR/RU/CNR and is more naturally read as "missing" than as "too low." Extending the code's meaning to cover both is a Pareto improvement on the wire side — no translation work needed. Spec §"Out of scope" already records "Renaming `PRICE_REQUIRED` to something more accurate (it actually means 'price too low,' not 'missing')" as deferred-cosmetic; the rename is already not blocking.

---

## Trust-boundary check

**Chain restated for the create path's `price` field:**

- Client supply: `NewProductRequestDTO.price` is client-supplied (`BigDecimal`, no validator).
- Server derivation: none. The server stores the value verbatim at line 125 (modulo the free-zone overwrite to `BigDecimal.ZERO` at line 701).
- Identity layer: `currentUserService.getCurrentUserIdStrict()` reads `userId` from `SecurityContextHolder` (`OglasinoAuthentication`), which `FirebaseAuthFilter` populates from the verified Firebase ID token + `redisUserAuth` cache. Owner binding happens at line 108 (`product.setOwner(user)`), not from any field on the request.
- Authorization: identity-derived only (`USER_SETUP_INCOMPLETE` checks `user.getBaseSite/Region/City`). `baseSite` is read from the user (line 109) and re-resolved server-side (lines 127–129) via `baseSiteContext` / `baseSiteCacheService` — never from the request.
- `freeZone` decision: derived server-side from `topCategory.isFreeZone()` on the `CategoryDTO` resolved from `baseSite.getCatalog()` (line 698). Not client-trusted.
- The `Product.free` boolean flows from `topCategory.isFreeZone()`, not from `price`. Setting `price = null` does not flip `free` — `free` is independent.
- `price` is not used in any moderation, authorization, or state-transition decision. State transitions key on `ProductState` (`ACTIVE`/`INACTIVE`/`DELETED`), independent of price. Moderation runs on `name` and `description` only (`ContentModerator`, per spec § Content moderation chain).

**Verdict: clean** as a trust-boundary check. The gap is a validation gap, not a trust-boundary violation.

**Caveat (worth recording but not changing the verdict):** the downstream effect is that a product can persist with `price = null, free = false`. That malformed state is plausibly NPE-inducing in price formatters, currency conversion, and Elasticsearch indexing pipelines that assume `price != null` on non-free products. The risk is integrity, not trust. Tracked here for the next brief to weigh; not material to the labelled verdict.

**Labelled verdict: clean (trust boundary); concern (validation gap).** Fix recommendation: Option B.

---

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. The QA Preparation feature is `in-progress-web` and this audit was triggered from that work; the audit's output is intake for the next backend brief on `feature/validation-refactor` (or wherever the fix lands). State.md flips only when the fix ships, not on the audit.
- issues.md: no change required from this session. Option B is the recommended fix and is small enough to land as a one-shot brief; it does not need an `issues.md` entry as a long-tail tracker. If Mastermind defers the fix instead of scheduling it immediately, an entry under `2026-05-23` would be appropriate — drafted text in "For Mastermind" below for that case.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports, no debug logging, no TODO/FIXME added. Read-only session.
- Part 4a (simplicity): see structured evidence in "For Mastermind". The recommendation favors the simpler option (B) over the heavier alternatives (A's validator class + handler special-case, C's refactor + null-handling fix).
- Part 4b (adjacent observations): two observations flagged below in "For Mastermind". Did not fix because out of scope.
- Part 6 (translations): N/A this session. Recommendation reuses the existing `product.price.required` key already seeded in EN/SR/RU/CNR.
- Part 7 (error contract): confirmed. Recommended fix emits 422 with `{field: "price", code: "PRICE_REQUIRED", translationKey: "product.price.required"}`, matching the spec's status table and the existing update-path emission.
- Part 11 (trust boundaries): confirmed. Audit's trust-boundary verdict is "clean"; gap is validation, not trust.

## Known gaps / TODOs

- None — audit only.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: Option A (DTO-layer Jakarta enforcement) and Option C (call `updatePriceCurrency` from create). Both expand scope without closing the gap more cleanly than Option B's ≤6-line guard. Rejection rationale is in §3 and §5 above.
  - Simplified or removed: nothing.

- **Recommendation: schedule Option B as the next brief.** Single-session backend brief; insert the guard between line 144 and line 164 in `DefaultProductService.createProduct`, add one test, run `./mvnw spotless:check` + `./mvnw test`. No DTO change, no wire change, no translation change. Estimated <30 minutes of work plus test.

- **Adjacent observation 1 — update path silently accepts null price on non-free-zone updates.** `DefaultProductService.updatePriceCurrency` line 823 (`if (source.getPrice() != null)`) is intentional — update is partial and a null `price` on the DTO means "don't change price." But combined with the create-path gap, a product created with `price = null` cannot be repaired through the update path (the persisted null is preserved). Once Option B closes the create gap, this becomes moot for new products. For products created under the historical gap (if any exist), a one-off migration or admin tool would be needed.
  - File: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:823`
  - Severity: low (conditional on whether any current products have `price = null` — Igor / DB query can confirm).
  - Did not fix this because it is out of scope.

- **Adjacent observation 2 — `isNoOpUpdate` treats two nulls as equal.** `DefaultProductService.isNoOpUpdate` lines 327–329: `if (dto.getPrice() == null ? persistedPrice != null : persistedPrice == null) return false; if (dto.getPrice() != null && dto.getPrice().compareTo(persistedPrice) != 0) return false;`. If both the persisted product and the incoming DTO have `price = null` on a non-free-zone product, the method considers the field unchanged and short-circuits. A product persisted in the gap state remains persisted in the gap state on update (no validation error surfaces, no fix path triggers). Same caveat as observation 1 — only relevant if the create-path gap has historically produced malformed products.
  - File: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:327`
  - Severity: low (downstream consequence of the create-path gap, not an independent defect).
  - Did not fix this because it is out of scope.

- **Drafted `issues.md` entry — only if Mastermind defers the Option B fix.** If the fix is scheduled as the next brief, no entry needed. If deferred:

  ```markdown
  ### 2026-05-23 — Create path missing PRICE_REQUIRED enforcement (medium)

  `DefaultProductService.createProduct` at
  `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:125`
  stores `request.getPrice()` verbatim. `populateCategories` (line 686) overwrites
  to `BigDecimal.ZERO` for free-zone categories but does not validate price for
  non-free-zone. The `PRICE_REQUIRED` throw at line 825 lives in
  `updatePriceCurrency`, called only from the update path. A client that bypasses
  the web Zod check at `productValidator.ts:76–82` (e.g. mobile pre-adoption,
  curl, future flows) can create a product with `price = null, free = false`.

  Audit: `.agent/2026-05-23-oglasino-backend-price-required-create-gap-1.md`.
  Recommended fix: Option B — guard in `createProduct` between the
  `topCategory` resolve and the `populateCategories` call.
  ```

  Target file: `oglasino-docs/issues.md`, append under the latest 2026-05-23 section if one exists, else as a new dated entry.

- **Branch note.** Igor's checked-out branch was `dev`, not the `main` or `feature/qa-preparation` the brief expected. Audit is read-only, so the discrepancy did not affect findings. Surfacing in case Mastermind wants the fix brief to specify a branch (most likely `feature/validation-refactor`, where the surrounding work landed).
