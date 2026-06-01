# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** Add /public/product/seen/* to CurrentLanguageFilter allowlist (VALIDATE-THEN-FIX)

## Phase 1 — VALIDATE (read-only)

**Q1 — Is `/api/public/product/seen/*` in the allowlist? ABSENT.**

Controller registers the route at `PublicProductController.java:20,27`:

```java
@RequestMapping("/api/public/product")
…
@GetMapping("/seen/{productId}")
```

Effective path: `/api/public/product/seen/{productId}`.

Allowlist at `CurrentLanguageFilter.java:35-44` (pre-fix):

```java
private static final List<String> ALLOWLIST_PREFIXES =
    List.of("/api/public/baseSite/", "/api/public/maintenance/", "/api/public/app/version/");

private static final List<String> ALLOWLIST_EXACT =
    List.of(
        "/api/public/config",
        "/api/public/translations",
        "/api/public/versions",
        "/api/public/health/check",
        "/api/public/verify-recaptcha");
```

Neither prefix nor exact entry covers `/api/public/product/seen/{productId}`. Confirmed absent.

**Q2 — Does the `/seen` request path consume `X-Lang` anywhere downstream? NO.**

Controller delegates: `productFacade.handleProductView(productId, request)` → `DefaultProductSeenService.handleProductView` (`DefaultProductSeenService.java:28-63`):

```java
public void handleProductView(Long productId, HttpServletRequest request) {
  var currentUserId = currentUserService.getCurrentUserId().orElse(null);
  Long productOwnerId =
      Optional.ofNullable(redis.opsForValue().get("product:owner:" + productId))
          .map(Long::getLong)
          .orElse(null);
  if (Objects.isNull(productOwnerId)) {
    productOwnerId = productService.getOwnerIdForProductId(productId).orElseThrow();
    redis.opsForValue().set("product:owner:" + productId, …);
  }
  if (Objects.nonNull(currentUserId) && currentUserId.equals(productOwnerId)) return;
  String deviceId = request.getHeader("X-Device-Id");
  String remoteAddr = request.getRemoteAddr();
  String viewerKey = … productId … deviceId / remoteAddr …;
  ThreadUtil.virtualThread("IncreaseProductView", () -> handleProductViewInternally(productId, viewerKey));
}
```

Inputs: authenticated identity, Redis owner cache, DB owner lookup, `X-Device-Id`/remote-addr (rate-limit keying), product id. No `X-Lang`, no `LanguageContext`, no language-keyed branching. No language dependence.

**Q3 — Does the `/seen` path need `X-Base-Site`? NO.**

`setProductAsSeen` does not reference `baseSiteContext` — the only `baseSiteContext` use in `PublicProductController` is `getProductCount` at line 43, on a different endpoint. `DefaultProductSeenService` does not inject or read `BaseSiteContext`. The counter is keyed on the globally-unique `productId`. No base-site requirement.

## Gate

Route ABSENT and no language dependence and no base-site requirement → **Phase 2 gate passes.**

## Phase 2 — FIX

## Implemented

- Added `/api/public/product/seen/` to `ALLOWLIST_PREFIXES` in `CurrentLanguageFilter` (prefix style with trailing slash, matching the existing `/api/public/baseSite/`, `/api/public/maintenance/`, `/api/public/app/version/` entries — the right shape for a path with a `{productId}` segment). The 400 `LANG_MISSING_OR_INVALID` rule and the rest of the filter are unchanged.
- Added one test `seenRouteWithMissingXLangContinuesChain` in `CurrentLanguageFilterTest`, mirroring the existing `allowlistedPrefixRouteWithMissingXLangContinuesChain` shape: `GET /api/public/product/seen/42` with no `X-Lang` returns 200 and continues the chain.

## Files touched

- src/main/java/com/memento/tech/oglasino/filter/CurrentLanguageFilter.java (+6 / -1)
- src/test/java/com/memento/tech/oglasino/filter/CurrentLanguageFilterTest.java (+27 / -0)

## Tests

- Ran: `./mvnw -Dtest=CurrentLanguageFilterTest test`
- Result: 11 passed, 0 failed (was 10 before; +1 new `seenRouteWithMissingXLangContinuesChain`).
- Ran: `./mvnw test`
- Result: 665 passed, 0 failed.
- Spotless: `./mvnw spotless:apply` then `./mvnw spotless:check` clean.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. The 2026-05-14 "Redis caching & filter control-flow fixes" entry (point 3) framed the allowlist's intent as "confirmed language-independent public routes." This session adds one route fitting that same rationale; the existing entry remains the authoritative description and does not need an amendment.
- state.md: no change. The "Product view count not incrementing (open bug)" Risk Watch row and the 2026-05-28 `issues.md` entry remain open because manual smoke on stage is the end-to-end gate per the brief's Definition of Done. Mastermind may decide to amend that entry / Risk Watch after stage smoke confirms the count moves; that is a Mastermind/Docs/QA call, not this session's.
- issues.md: no change in this session. The 2026-05-28 entry "Product view count always 0 across the portal (increment never fires)" lists hypothesis (c) (`owner.iamActive` gate suppression after the #8 fix) as the most likely cause. This session's fix removes hypothesis-class "backend rejects with 400 at the language filter," which the entry explicitly does not list but which web's diagnosis flagged. Stage smoke determines which (a)/(b)/(c)/(d) actually fires.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): see "For Mastermind" — one observation logged.
- Part 6 (translations): N/A this session (no translation work).
- Other parts touched:
  - Part 7 (error contract): confirmed — the 400 `LANG_MISSING_OR_INVALID` body emits `{field, code, translationKey}` codes-only; no message text. Untouched in this session.
  - Part 11 (trust boundaries): confirmed — the seen path consumes the authenticated identity via `SecurityContextHolder` (through `CurrentUserService`) and the unforgeable `X-Device-Id`/remote-addr for rate-limit keying; nothing else. No client-supplied "before"/"previous" values. The allowlist addition does not introduce or weaken any trust boundary — the route is a counter side-effect keyed on a server-validated product id.

## Known gaps / TODOs

- The web `markAsSeen` change is already shipped (`after()` wrap + `skipAuth: true`) — out of scope. Whether the count actually increments on stage is the end-to-end gate (per the brief's Definition of Done). This fix only makes the route reachable through the language filter.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): one new entry in `ALLOWLIST_PREFIXES` and one new test method. Neither is a new abstraction; both reuse the existing prefix-list shape and the existing test pattern. Matches Part 4a "Match the surrounding code's style."
  - Considered and rejected: (a) a parameterized test for all allowlisted routes instead of one method per route — rejected, the file's existing style is one method per route (`allowlistedPrefixRouteWithMissingXLangContinuesChain`, `allowlistedExactRouteWithMissingXLangContinuesChain`, `versionsRouteWithMissingXLangContinuesChain`); changing to parameterized would create a parallel pattern next to the existing one. (b) Extending the `LANG_MISSING_OR_INVALID` 400 body to include a hint about which routes are exempt — rejected as out of scope and outside the error-contract shape.
  - Simplified or removed: nothing.
- **Adjacent observation (Part 4b, low):** The `Long::getLong` reference at `DefaultProductSeenService.java:34` (`Optional.ofNullable(redis.opsForValue().get(…)).map(Long::getLong)`) is `Long.getLong(String)`, which reads from system properties — not what is meant for converting a Redis string value to `Long`. It will return `null` for any product whose owner-id key isn't also a system property, defeating the Redis hit and falling through to `productService.getOwnerIdForProductId(productId).orElseThrow()` on every call. File path: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductSeenService.java:34`. Severity: medium (correctness bug; defeats the owner-id Redis cache so every `/seen` call hits the DB for owner lookup; not a security or trust-boundary issue). I did not fix this because it is out of scope for this brief (the brief is strictly the allowlist addition). Likely relevant to the open Risk Watch on view-count once stage smoke runs — if cache hits are still rare even with the route reachable, this is one reason.
- **Config-file impact summary:** no drafted edits this session. The 2026-05-14 decisions.md entry (point 3) carries the allowlist's stated intent and remains authoritative as written. The `issues.md` entry and Risk Watch row should be amended/closed by Docs/QA only after stage smoke per the brief's Definition of Done.
