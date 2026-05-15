# Connection-pool & Cache Hardening

Structural follow-ups from the 2026-05-14 connection-pool exhaustion incident. This is a stub — when the feature is picked up, a Mastermind chat opens it for the full spec.

**Status:** `planned`
**Branch:** _(not yet created)_

---

## Why this is a feature, not a bug-fix

The connection-pool bug-fixer chat (2026-05-14) closed the **post-deploy cold-cache window** via warmup-gated readiness — traffic is now refused until the warmup pass has run, so the first wave of concurrent requests cannot hit a cold cache.

The mechanism it did **not** close is the **daily-TTL-expiry cold-cache window**: every 24 hours, `redisBaseSites` (and any other TTL'd reference-data cache) expires, and the next concurrent burst can trigger the same thundering-herd that drained the pool the first time. Closing that window — and the other structural items below — is a deliberate design effort, not a bug-fix. It requires per-key locking primitives that don't exist in the codebase today, or a different cache substrate for small reference-data, or both. That's feature work.

See [`decisions.md`](../decisions.md) 2026-05-14 entries (`Connection-pool exhaustion incident` and `Redis caching & filter control-flow fixes`) for the bug-fixer chat's recorded boundary.

---

## Must-address items

### 1. Per-key single-flight deduplication of concurrent cache misses

Spring's `CacheInterceptor` has no per-key locking. 20 concurrent misses on the same cache key produce 20 simultaneous DB reads — the core mechanism behind the pool exhaustion.

Candidate shapes discussed in the bug-fixer chat:

- An in-memory `Map<key, CompletableFuture<value>>` single-flight pattern inside `DefaultBaseSiteCacheService` and `DefaultLanguageService` — concurrent misses on the same key share one in-flight load.
- A long-lived in-memory cache (Caffeine, or a `ConcurrentHashMap` snapshot) of the small, rarely-changing reference-data lists — Redis is overkill for them. Bypass `redisBaseSites` for hot reads entirely.

Single-flight makes the TTL safe at any value, so it also closes item 3.

### 2. Remove `@Transactional` from `DefaultBaseSiteService.getAllBaseSites`

The method runs **1 + 3·N JPA queries** inside a single read-only transaction, which holds a JDBC connection for the entire duration. Splitting the read sequence so each repo call takes its own brief Spring-Data-JPA transaction reduces per-request connection-hold time.

This compounds with item 1 — even with single-flight, the surviving caller still holds a connection longer than it needs to.

### 3. The `redisBaseSites` 24h-TTL cold-cache trigger remains unaddressed

The warmup-gating fix closes the post-deploy window only. The daily TTL expiry still produces a cold-cache window during which the next concurrent burst can exhaust the pool.

Fix candidates: longer TTL, in-memory caching (item 1's second shape), or single-flight (item 1's first shape, which makes the TTL safe at any value).

### 4. `RateLimitFilter` categorisation of public reference-data endpoints

`/api/public/translations` and `/api/public/baseSite/overviews` have **no rate-limit category** — a single IP can fire arbitrary parallel requests. Adding a category cuts against the documented "rate-limiting narrow on purpose" stance, so it requires its own [`decisions.md`](../decisions.md) reasoning when the feature lands.

### 5. Filter `@Order` ambiguity

The four `@Component` filters (`BotFilter`, `BaseSiteFilter`, `CurrentLanguageFilter`, `InputSanitizationFilter`) have **no explicit `@Order`**. Relative ordering is currently undefined; it happens to work today, but a future filter that depends on ordering would silently break.

This is the cheapest item to address and should land alongside whichever item is picked up first.

---

## What was closed in the bug-fixer chat (do not re-do)

- Warmup-gated readiness (`CacheWarmupService` publishes `AvailabilityChangeEvent` only after warmup completes)
- Docker-compose healthcheck flipped from `/actuator/health/liveness` to `/actuator/health/readiness`, `start_period` 90s→120s
- `redisUserAuth` TTL raised 1min → 30min
- `DefaultBaseCurrencyService` self-invocation bypass fixed (`@Lazy` self-injection)
- `DefaultTranslationService.updateTranslation` made `@Transactional`
- `CurrentLanguageFilter` control-flow fix (`doFilter` outside `try`) + X-Lang 400 rule on non-allowlisted public routes

Full record in [`decisions.md`](../decisions.md) 2026-05-14 entries.

---

## Source material

The investigations in [`sessions/`](../sessions/) that did the original analysis. Whoever picks this feature up should read them first.

- [`investigation-connection-pool.md`](../sessions/investigation-connection-pool.md) — root-cause trace of the thundering-herd cold-cache miss
- [`investigation-redis-caching.md`](../sessions/investigation-redis-caching.md) — caller traces across the Redis caching layer
- [`investigation-translations-ttl.md`](../sessions/investigation-translations-ttl.md) — verdict that `/api/public/translations` is a victim, not a culprit

Adjacent session summaries with surrounding context:

- [`2026-05-14-oglasino-backend-connection-pool-1.md`](../sessions/2026-05-14-oglasino-backend-connection-pool-1.md) through [`-3.md`](../sessions/2026-05-14-oglasino-backend-connection-pool-3.md)
- [`2026-05-14-oglasino-backend-redis-caching-1.md`](../sessions/2026-05-14-oglasino-backend-redis-caching-1.md), [`-2.md`](../sessions/2026-05-14-oglasino-backend-redis-caching-2.md)
- [`2026-05-14-oglasino-backend-redis-cleanup-1.md`](../sessions/2026-05-14-oglasino-backend-redis-cleanup-1.md)

---

## Open questions for spec time

- Are items 1–5 one feature or two? Item 5 (filter `@Order`) is cheap and isolated; items 1–3 cluster around cache mechanics; item 4 is a separate `decisions.md`-shaped policy call. Splitting may be appropriate.
- Single-flight vs in-memory cache for item 1 — which shape, or both for different caches?
- Stage vs prod pool sizes (8 vs 18) — do any of the fixes interact with pool sizing?
