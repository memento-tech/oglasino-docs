# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** Read-only check — view-counter backend behavior (iamActive default + /seen vs /views)

## Implemented

- Read-only investigation. Three backend behaviors confirmed against source. No code changes, no git, no test runs. Output is the answers below.

## Files touched

- None (read-only).

## Tests

- None run (read-only check).

---

# Answers

## Q1 — `iamActive` default with no authenticated viewer

**Answer: `false`. The web gate fires for anonymous/no-identity callers.**

`iamActive` is populated by `DefaultUserFacade.applyCallerContext(UserInfoDTO)` at `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java:88-98`. The relevant branch:

```java
Long currentUserId = currentUserService.getCurrentUserId().orElse(null);
if (currentUserId != null && dto.getId() != null) {
  dto.setIamActive(currentUserId.equals(dto.getId()));
  dto.setFollowingCurrent(followService.isFollowing(currentUserId, dto.getId()));
} else {
  dto.setIamActive(false);          // ← line 94
  dto.setFollowingCurrent(false);
}
return dto;
```

- Both endpoints that return `UserInfoDTO` from this facade (`getUserData(Long)` at line 71, `getUserDataByFirebaseUid(String)` at line 76) pipe through `applyCallerContext`.
- The `UserInfoDTO` field initialiser at `src/main/java/com/memento/tech/oglasino/dto/UserInfoDTO.java:36` is `private boolean iamActive;` (no `= true` default), and the cache layer (`EntityUserInfoConverter`) deliberately does not set it — class javadoc at lines 9-19 and converter comment at line 64 both confirm this is intentional so cached representations don't leak one caller's view to another.
- For a `/public/user?id=<id>` request with no Bearer token, `FirebaseAuthFilter` populates no authentication (the filter suppresses 401s for `/api/public/**` paths — `FirebaseAuthFilter.java:118-123`), so `SecurityContextHolder.getContext().getAuthentication()` is null, `currentUserService.getCurrentUserId()` returns `Optional.empty()`, and the else branch (line 94) sets `iamActive = false`.

**Conclusion for Q1:** Anonymous viewer → `iamActive = false` → web gate `if (!owner.iamActive) markAsSeen(...)` **fires**. Q1 is not the bug.

---

## Q2 — Do `/seen` and `/views` touch the same counter?

**Answer: yes — same logical counter. Response shape is a bare number.**

### Write path (`GET /public/product/seen/<id>`)

- Controller: `src/main/java/com/memento/tech/oglasino/controller/PublicProductController.java:27-33`
  ```java
  @GetMapping("/seen/{productId}")
  public ResponseEntity<Void> setProductAsSeen(...) {
    productFacade.handleProductView(productId, request);
    return ResponseEntity.ok().build();
  }
  ```
- Facade: `DefaultProductFacade.handleProductView` (`facade/impl/DefaultProductFacade.java:54-56`) delegates to `DefaultProductSeenService.handleProductView`.
- Service: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductSeenService.java:29-63` — looks up `productOwnerId` (Redis-cached at key `product:owner:<id>`, falls back to DB on miss), excludes authenticated owners (line 49 — see Q3), then spawns a virtual thread that calls `handleProductViewInternally(productId, viewerKey)`.
- Internal: `handleProductViewInternally` (lines 72-81) consults a per-viewer token bucket via `RateLimiterService.tryConsume(viewerKey, capacity, refillPerSecond)`. If allowed, calls `redisViewCounterService.incrementViewDeltaBy(productId, 1)`.
- Redis write: `DefaultRedisViewCounterService.incrementViewDeltaBy` (`redis/service/impl/DefaultRedisViewCounterService.java:29-39`) — `INCRBY product:views:delta:<productId> 1`, `EXPIRE` per `redis.product.view.rate.limiter.delta.ttl`, and `SADD product:views:keys <productId>`.
- Scheduled flush to DB: `DefaultScheduledRedisFlushService.periodicFlush` (`redis/service/impl/DefaultScheduledRedisFlushService.java:20-54`) runs at fixed delay `${app.views.flush-delay-ms}`, `GETSET`s each tracked delta to 0, and persists via `productService.incrementNumberOfViewsBy(productId, delta)` → `ProductRepository.incrementNumberOfViewsBy` (`repository/ProductRepository.java:21-24`) → `UPDATE Product p SET p.numberOfViews = p.numberOfViews + :amount WHERE p.id = :productId`.

### Read path (`GET /public/product/views/<id>`)

- Controller: `PublicProductController.java:36-39`
  ```java
  @GetMapping("/views/{productId}")
  public ResponseEntity<Long> getNumberOfViews(@PathVariable @NotNull Long productId) {
    return ResponseEntity.ok(productFacade.getNumberOfViews(productId));
  }
  ```
- Facade: `DefaultProductFacade.getNumberOfViews` → `productSeenService.getNumberOfViewsForProduct(productId)`.
- Service: `DefaultProductSeenService.getNumberOfViewsForProduct` (`service/impl/DefaultProductSeenService.java:65-70`):
  ```java
  long base = productService.getNumberOfViews(productId);      // DB Product.numberOfViews
  long delta = redisViewCounterService.getDelta(productId);    // Redis delta (unflushed)
  return base + delta;
  ```
- `productService.getNumberOfViews` → `ProductRepository.getNumberOfViewsById` (`repository/ProductRepository.java:26-27`) → `SELECT p.numberOfViews FROM Product p WHERE p.id = :productId`.
- `redisViewCounterService.getDelta` reads `product:views:delta:<productId>` (same key the write path increments).

### Counter alignment

The write path increments `product:views:delta:<productId>` in Redis; the periodic flush merges that into `Product.numberOfViews` in DB. The read path returns `Product.numberOfViews + product:views:delta:<productId>`. **Increments are visible to `/views` immediately, even before the scheduled flush runs** — both stores are summed at read time.

### Response shape

`ResponseEntity<Long>` serialises to a **bare JSON number** (not `{ "count": N }`). Matches the web's `res.data as number` expectation. No shape mismatch.

**Conclusion for Q2:** Same logical counter (DB base + Redis delta). Bare-number response shape. Q2 is not the bug.

---

## Q3 — Does `/seen` persist, and is owner-exclusion duplicated server-side?

**Answer: yes, it persists; yes, server-side owner-exclusion is duplicated — but only for authenticated callers.**

### Persistence guards on `/seen`

1. **Security filter:** `/api/public/**` is `permitAll()` (`SecurityConfig.java:77-78`). `FirebaseAuthFilter` runs but does not require auth on this prefix (suppression at `FirebaseAuthFilter.java:118-123`). So no auth-required guard suppresses the increment.

2. **`RateLimitFilter` (filter-level):** `RateLimitFilter.java:132-134` maps `GET /api/public/product/seen*` to `RateLimitCategory.PRODUCT_SEEN`, which is `Bandwidth.simple(10, Duration.ofSeconds(10))` (`security/ratelimit/RateLimitCategory.java:22-23`) — burst 10 per 10 seconds per (`user:` | `device:` | `ip:`) key. Exceeding the budget returns 429 and the increment never runs. This is a per-caller cap, not a per-product cap; one user spamming refreshes on many different products still trips it.

3. **Per-(viewer + product) token bucket (service-level):** `handleProductViewInternally` (`DefaultProductSeenService.java:72-81`) builds `viewerKey = "view:<productId>:<deviceId-or-remoteAddr>"` (lines 51-57) and calls `RateLimiterService.tryConsume` with capacity/refill from config keys `redis.product.view.rate.limiter.capacity` / `.refill.per.second`. If the bucket is exhausted, the Redis delta does **not** increment. Default 1/sec sustained (configurable). This is the "don't count the same viewer twice in quick succession" guard — it's the main mechanism that suppresses repeat increments even when the filter-level burst is unspent.

4. **No auth-required guard, no rate-limit-by-user-required guard.** Anonymous callers are accepted by both layers.

### Server-side owner-exclusion (duplicated)

Yes, at `DefaultProductSeenService.java:49`:

```java
if (Objects.nonNull(currentUserId) && currentUserId.equals(productOwnerId)) return;
```

- **Authenticated owner → server suppresses the increment.** The web gate is redundant for this case.
- **Anonymous caller (no Bearer) → `currentUserId == null`, the branch does not fire.** The server cannot identify the caller as the owner, so the increment proceeds.

This is the trust-boundary-sound posture: the server only excludes a viewer when it can prove identity from the authenticated principal. It does **not** trust a client-supplied "I am the owner" signal.

### Persistence in steady state

`/seen` returns 200 immediately (`ResponseEntity.ok().build()`) regardless of whether the virtual thread eventually increments. The virtual thread:
- Always runs (no early-exit between line 58 spawn and line 80 increment except the per-viewer bucket guard inside `handleProductViewInternally`).
- Increments `product:views:delta:<id>` in Redis on success.
- The scheduled flush (`DefaultScheduledRedisFlushService`) drains to DB on `${app.views.flush-delay-ms}` cadence. On flush failure, the delta is re-incremented to preserve the count (`DefaultScheduledRedisFlushService.java:50-52`).
- `/views` reads sum the unflushed delta, so the increment is visible to the next `/views` call within milliseconds of the virtual-thread write — no flush-cadence latency between `/seen` and `/views`.

**Conclusion for Q3:** Increment persists. Server-side owner-exclusion is duplicated — but only for authenticated callers. If the web side's `/seen` request **does not carry a Bearer token**, the server cannot exclude the owner, and an owner viewing their own product page anonymously (or via an `/api/public` axios instance with `skipAuth: true`) would have their view counted. Whether the web's fire-and-forget call attaches a Bearer is outside this brief's scope; flagging for Mastermind.

---

## Verdict

**No backend bug in the view-counter chain.** All three legs work as designed:
- `iamActive = false` for anonymous viewers, so the web gate fires correctly.
- `/seen` writes and `/views` reads the same logical counter (DB base + Redis delta).
- `/seen` persists via virtual-thread Redis INCR with scheduled DB flush; `/views` sums both stores so increments are visible immediately.
- Server-side owner-exclusion is in place for authenticated callers and is the trust-boundary-correct implementation.

The proximate cause of the "view count never displays" defect is therefore not in the four backend behaviours this brief asked about. The web audit's render-guard hypothesis is the most likely surviving cause; if Mastermind wants to fold in the "drop the web gate, let the server decide" cleanup mentioned in Q3, that is a web-side simplification, but it depends on the web side guaranteeing that `/seen` is called through an authenticated axios instance (so the server's owner-exclusion branch can fire).

---

## Cleanup performed

- None needed (read-only).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A (read-only)
- Part 4a (simplicity) / Part 4b (adjacent observations): see "For Mastermind"
- Part 6 (translations): N/A this session
- Part 11 (trust boundaries): confirmed — the server's owner-exclusion at `DefaultProductSeenService.java:49` reads `currentUserId` from `SecurityContextHolder` via `CurrentUserService` and compares to the product's owner ID from the DB (with a Redis cache for the owner lookup keyed by `productId`). No client-supplied owner claim is trusted.
- Part 7 (error contract): N/A this session (no error paths in scope).

## Known gaps / TODOs

- None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only).
  - Considered and rejected: nothing (read-only).
  - Simplified or removed: nothing (read-only).

- **Trust-boundary observation tied to the brief's Q3.** The server-side owner-exclusion at `DefaultProductSeenService.java:49` only fires when the caller is authenticated. If the web's fire-and-forget `/api/public/product/seen/<id>` call goes through an axios instance that does **not** attach the Firebase Bearer token (e.g. the `skipAuth: true` instance noted in the brief for the `/public/user` call), then the server cannot identify the owner and will count an owner's view. The cleanest fix per the brief's Q3 ("drop the web gate entirely and let the server decide") is only safe if the web guarantees that `/seen` is called through the **authenticated** axios instance for logged-in users. Worth confirming with the Web agent before Mastermind issues a "drop the gate" brief. Medium severity (potentially under-reports owner-self-view exclusion, no user-facing breakage).

- **Adjacent observation (Part 4b — out of scope for this brief).** The Redis owner cache key `product:owner:<productId>` in `DefaultProductSeenService` (line 33) is populated synchronously on cache miss with a TTL from config key `redis.product.view.rate.limiter.product.owner.ttl`. The key never expires on owner change (transfer of ownership is unlikely in this product, but if a Product's owner ever changes within the TTL window, this cache would serve a stale owner ID and the wrong principal would be excluded). Very low severity; flagging for completeness. File: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductSeenService.java:32-47`.

- **Adjacent observation (Part 4b — out of scope).** `DefaultScheduledRedisFlushService.periodicFlush` (`redis/service/impl/DefaultScheduledRedisFlushService.java:20-54`) is annotated `@Transactional` at the method level. The body iterates over per-product Redis `getAndResetDelta` calls (Redis ops, not transactional) **before** the per-product `productService.incrementNumberOfViewsBy` DB calls (transactional). If the DB write throws for one product, the catch block re-increments the Redis delta (lines 50-52) — so the recovery is correct — but the surrounding `@Transactional` at the flush-all method is misleading: each `incrementNumberOfViewsBy` is its own `@Modifying @Transactional` repository call (`ProductRepository.java:21-23`), so the outer `@Transactional` here only matters if the per-product calls are intended to commit as a batch. Currently it doesn't change behaviour (Spring's default propagation `REQUIRED` will reuse the outer tx, but per-product failures still trigger the catch). Low severity; might be worth a comment or removal, but not in scope here.

- (or: nothing else flagged.)
