# Investigation — HikariCP connection-pool exhaustion under concurrent load

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Type:** Read-only investigation, no code changes.

---

## TL;DR

The pool exhaustion is real and explainable. The brief's working hypothesis — "even a cache HIT on `getAllBaseSites` acquires a JDBC connection" — is **wrong**. A cache HIT does NOT acquire a connection, because Spring's `CacheInterceptor` is the **outer** advisor and short-circuits before `TransactionInterceptor` is reached. The production stack trace shows `TransactionInterceptor` being entered because at the moment of those failed requests the cache was MISSED, not because of an unconditional per-request tx.

What actually drains the pool under burst load is a **thundering-herd cache miss**:

1. The `BaseSiteFilter` runs on every request that has an `X-Base-Site` header.
2. On a cold cache (post-restart between `RedisCacheEvictConfig.evictCachesOnStartup` and `CacheWarmupService.warmup` completion — or after a 24 h TTL expiry, or after an admin evict), `DefaultBaseSiteService.getAllBaseSites()` is `@Cacheable("redisBaseSites") + @Transactional(readOnly = true)`. The cached method body itself runs 1 + 3·N JPA queries (where N = number of active base sites) before commiting.
3. When ~20 requests arrive in the same second and all miss simultaneously, Spring's `CacheInterceptor` provides **no per-key locking** — every miss independently calls `proceed()`, which opens a transaction, which tries to acquire a connection.
4. With a pool of 8 (the value in the production log evidence — matches `application-stage.yaml` exactly), only 8 can run; the remaining 12 wait. After `connection-timeout=5000ms` they fail with `HikariPool-1 - Connection is not available, request timed out after 5001ms` and become HTTP 500s.

`/api/public/translations` is a **victim, not the culprit**. Its own controller path is Redis-only (no DB unless the Redis translation entry is itself missing). The connection acquisition that fails is upstream, in `BaseSiteFilter` (and `CurrentLanguageFilter` on its miss path), before the controller is even reached.

The most likely real-world trigger of the 14:35:49 incident is a bot burst arriving during a brief cold-cache window (post-deploy / post-warmup-failure / post-TTL-expiry) — multiplied by the fact that `/api/public/translations` is **not categorised by `RateLimitFilter`** so a single IP can hammer it without throttle.

The fix shape is bug-sized for the request-path symptom but has a feature-sized footprint if Mastermind wants to harden the broader pattern (per-key in-flight deduplication, cache prewarm before traffic, `connection-timeout` rebalancing, `RateLimitFilter` categorisation for public reference-data endpoints, removal of `@Transactional` from `getAllBaseSites` so cache misses don't hold a connection across 1+3·N queries). Discussed in "For Mastermind" below.

---

## Brief vs reality

One mismatch worth surfacing before the questions. **Surfacing, not blocking** — the investigation proceeds as scoped.

1. **The "8" in the logs is not a prod default — it's the stage profile.**
   - Brief says: "the logs show `total=8`. Confirm that's the configured value and not a default, and report where it comes from."
   - Code says:
     - `application-prod.yaml:15` — `maximum-pool-size: 18  # tuned for DO Basic 1GB Postgres (25 conn limit)`
     - `application-stage.yaml:18` — `maximum-pool-size: 8`
     - `application-dev.yaml:18` — `maximum-pool-size: 20`
   - The only place a max-pool-size of 8 is configured is `application-stage.yaml`. HikariCP's library default is `10`, not `8`, so the observed `total=8` cannot be a missing-config default — it has to be an active stage profile.
   - Why this matters: either the "production" logs are actually from staging, or the prod deployment is running with `SPRING_PROFILES_ACTIVE=stage` (or with an env-var override that resolves to 8). The latter is plausible if the rollout used the wrong env file. **Worth confirming with Igor before any fix is sized to "production with 18 connections" or "production with 8 connections."** The two have very different headroom: at 18, 20 concurrent misses are tight but might fit; at 8, they cannot.
   - I have not opened the deploy config files in `/opt/oglasino/.env` or `docker-compose.yml` because the brief is read-only and I don't have those outside the repo. Surfacing only.

I have not stopped on this. The questions are answered below using `application-prod.yaml`'s declared `maximum-pool-size: 18`, with the caveat that the actual prod value at the time of the incident may have been 8.

---

## Question 1 — The HikariCP pool configuration

### Where it's set

The pool config is declared entirely in YAML — there is **no `@Configuration` `DataSource` bean** in the codebase. Verified by `grep -rn "DataSource" src/main/java/` returning zero matches and `grep -rn "HikariConfig\|HikariDataSource" src/main/java/` returning zero matches. Spring Boot's `DataSourceAutoConfiguration` reads `spring.datasource.*` and constructs the `HikariDataSource` from the YAML.

### Prod values (`application-prod.yaml:13-20`)

```yaml
spring:
  datasource:
    hikari:
      minimum-idle: 10
      maximum-pool-size: 18      # tuned for DO Basic 1GB Postgres (25 conn limit)
      auto-commit: true
      idle-timeout: 300000       # 5 min
      max-lifetime: 1200000      # 20 min
      connection-timeout: 5000   # 5 s — matches the prod log evidence
      initialization-fail-timeout: 10000
```

### Stage values (`application-stage.yaml:13-23`) — exactly matches the log evidence

```yaml
spring:
  datasource:
    hikari:
      minimum-idle: 2
      maximum-pool-size: 8       # <-- the 8 in the log evidence
      auto-commit: true
      idle-timeout: 300000
      max-lifetime: 1200000
      connection-timeout: 5000   # <-- the 5001ms timeout in the log evidence
      initialization-fail-timeout: 10000
```

### Dev values (`application-dev.yaml:16-18`, partial)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
```

### Answering the brief's sub-questions

- **`maximum-pool-size` in production** as declared: `18` (`application-prod.yaml:15`). The log evidence (`total=8`) does **not** match the prod declaration; it does match `application-stage.yaml:18` exactly. See "Brief vs reality" above.
- **`connection-timeout` mapping to the 5-second wait**: yes. `application-prod.yaml:19` (and `application-stage.yaml:22`) both set `connection-timeout: 5000`, which matches the logged `request timed out after 5001ms` (HikariCP adds ~1 ms tolerance to the timeout-message string).

### Other relevant `spring.datasource` settings

- `spring.jpa.hibernate.ddl-auto: ${JPA_DDL_AUTO}` (validate in prod). No DDL at runtime.
- `spring.jpa.open-in-view` is **not set** anywhere in the three yaml files; Spring Boot defaults to `open-in-view: true`. This means `OpenEntityManagerInViewInterceptor` is registered. **It does not change the connection-holding analysis** for the failure path under investigation — connections are acquired and released around each `@Transactional` boundary regardless of OSIV (Hibernate's default connection-handling-mode is `DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION`). OSIV does, however, leave a Session bound to the request for lazy-loading in view-rendering, which can re-acquire a connection if any lazy-init fires post-tx; none of the affected controllers do that on the hot path, so it's noise here.

---

## Question 2 — `BaseSiteFilter` and the per-request transaction (load-bearing finding)

### The chain — file:line accurate

```
HTTP request
  → BaseSiteFilter.doFilterInternal (filter/BaseSiteFilter.java:21-36)
      header X-Base-Site or query param baseSite is present
      → baseSiteCacheService.getBaseSiteForCode(code) (BaseSiteFilter.java:31)
          (proxy: BaseSiteCacheService is a Spring bean, but the called method
           getBaseSiteForCode is NOT annotated, so no interceptor fires here)
          → DefaultBaseSiteCacheService.getBaseSiteForCode (DefaultBaseSiteCacheService.java:35)
              → this.getAllBaseSites() (DefaultBaseSiteCacheService.java:36 — SELF call, no proxy)
                  → DefaultBaseSiteCacheService.getAllBaseSites (DefaultBaseSiteCacheService.java:30)
                      → baseSiteService.getAllBaseSites() (DefaultBaseSiteCacheService.java:31
                         — CROSS-BEAN call to a different bean, proxy fires)
                          → DefaultBaseSiteService$$SpringCGLIB.getAllBaseSites
                              → CacheInterceptor    (outer — see "interceptor order" below)
                              → TransactionInterceptor (inner)
                                  → JpaTransactionManager.doBegin
                                      → HikariDataSource.getConnection()  ← pool acquisition
                                      → run method body
                                          → baseSiteRepository.findActiveBaseSiteCodes()
                                          → for each code:
                                              baseSiteRepository.findCoreByCode
                                              baseSiteRepository.findAllowedLanguages
                                              baseSiteRepository.findAllowedCurrencies
                                      → commit
                                      → return connection to pool
                              → CacheInterceptor stores result in redisBaseSites
                              → return list
```

### Is `DefaultBaseSiteService.getAllBaseSites()` `@Transactional`?

**Yes.** From `DefaultBaseSiteService.java:57-64`:

```java
@Override
@Cacheable(value = "redisBaseSites", unless = "#result == null || #result.isEmpty()")
@Transactional(readOnly = true)
public List<BaseSiteDTO> getAllBaseSites() { ... }
```

Both `@Cacheable` and `@Transactional(readOnly = true)` apply.

Compare `DefaultBaseSiteService.getAllBaseSiteOverviews()` (lines 29-55), which is `@Cacheable` but **not** `@Transactional` — so its cache-miss path also opens a tx, but via Spring Data JPA's per-repository-method tx (a separate, narrower mechanism), not via a class-method `@Transactional`.

### Interceptor ordering — `@Cacheable` vs `@Transactional` on the same method

`@EnableCaching` is on `RedisConfig.java:24`; `@EnableTransactionManagement` is not present in user code (Spring Boot auto-config registers transaction management via `TransactionAutoConfiguration`). Neither configures a non-default `order` attribute. Both advisors are registered at `Ordered.LOWEST_PRECEDENCE`, which means ordering is determined at runtime by `AnnotationAwareOrderComparator`'s fallback to bean-registration order.

**The production stack trace tells us the runtime ordering directly.** The trace order (caller → callee) is:

```
DefaultBaseSiteService$$SpringCGLIB.getAllBaseSites
  → CacheInterceptor
  → TransactionInterceptor
  → JpaTransactionManager.doBegin
```

`CacheInterceptor` invoked `TransactionInterceptor.invoke()` via `chain.proceed()`. That means **`CacheInterceptor` is the OUTER advisor and `TransactionInterceptor` is the INNER**.

This is the standard Spring auto-config outcome when both advisors share `LOWEST_PRECEDENCE`: with Boot's auto-configuration order (transaction registered first, caching registered second), the caching advisor ends up applied *after* the transaction advisor on the proxy chain — which means it is **closer to the caller**, i.e. outer. The chain is `caller → CacheInterceptor → TransactionInterceptor → method`.

### **THE LOAD-BEARING ANSWER: does a cache HIT on `getAllBaseSites()` acquire a JDBC connection?**

**No.** With `CacheInterceptor` as the outer advisor, on a cache hit `CacheInterceptor.invoke()` retrieves the value from Redis and returns it directly **without calling `chain.proceed()`**. `TransactionInterceptor` is never entered. `JpaTransactionManager.doBegin` is never called. **No JDBC connection is acquired.**

The Redis read itself is the only resource cost on a hit — a single GET against `redisBaseSites`, deserialise the cached `List<BaseSiteDTO>`, return. No pool contact.

(That this is what's happening on a HIT is also corroborated by the previous Redis investigation `.agent/investigation-redis-caching.md` Step 2 — the chain is shown crossing the proxy, but neither cache-hit benchmark nor key alignment indicated any DB traffic on warm-cache reads.)

### What the failing requests in the production logs actually show

The stack trace ends in `JpaTransactionManager.doBegin` → connection-acquisition timeout. With Cache outer, the only way that path executes is **a cache miss**. At the moment those 20+ requests fired:

- The cache was empty for the `redisBaseSites` bucket (or `redisBaseSiteOverviews` for the `/baseSite/overviews` path), AND
- All 20+ requests missed simultaneously, AND
- Spring's `CacheInterceptor` has **no per-key locking** for thundering-herd cache misses, so each request independently calls through to `TransactionInterceptor`.

So 20 simultaneous misses → 20 simultaneous tx-begin attempts → 8 succeed, 13 wait → 5 s timeout → HTTP 500 with the trace shown.

This is **the correct mechanism** by which a bursty client can drain the pool. It does not require a per-request transaction on warm-cache reads; it requires a brief cold-cache window plus a concurrency burst.

When the cache is cold:
- **Post-deploy window**: `RedisCacheEvictConfig.evictCachesOnStartup` (an `ApplicationRunner`) wipes all caches BEFORE `ApplicationReadyEvent` fires. `CacheWarmupService.warmup` runs as an `ApplicationReadyEvent` listener with `@Order(10)`, AFTER application ready event. Between the two, all caches are empty. If traffic reaches the server in that window (Caddy/CF flips ready → traffic flows), the very first burst of `/api/public/translations` requests with `X-Base-Site` will all miss `redisBaseSites` simultaneously.
- **24 h TTL expiry** (`RedisConfig.java:49` — `1440` min for `redisBaseSites`): once per day per cache entry, the entry expires. The next access is a miss. If the next access happens to be a burst, same thundering-herd.
- **Admin manual evict** (`CacheAdminController` exposes `/api/secure/admin/cache/evict[/{name}]`): if used in production, creates the same window.
- **Warmup itself failed** for `redisBaseSites`: `CacheWarmupService.warmup` catches per-cache exceptions and logs them as warnings (`warmOne` at `CacheWarmupService.java:97-106`). If the warmup query failed (e.g. a transient DB blip), the cache stays cold and the first user-request burst is the cold-fill.

### Same question for `CurrentLanguageFilter` → `getLanguageByCode`

The chain:

```
HTTP request
  → CurrentLanguageFilter.doFilterInternal (filter/CurrentLanguageFilter.java:31)
      header X-Lang is present
      → languageService.getLanguageByCode(langCode) (CurrentLanguageFilter.java:55)
          → DefaultLanguageService$$proxy.getLanguageByCode
              → CacheInterceptor (outer)
                  → cache hit: deserialise LanguageDTO, return — NO method body executed
                  → cache miss: chain.proceed()
                      → DefaultLanguageService.getLanguageByCode body
                          → languageRepository.findLangByCode(...)
                              → Spring Data JPA-internal @Transactional(readOnly = true)
                                  on SimpleJpaRepository — opens tx, acquires connection
```

From `DefaultLanguageService.java:25-29`:

```java
@Override
@Cacheable(value = "redisLanguage", key = "#langCode", unless = "#result == null")
public LanguageDTO getLanguageByCode(String langCode) {
  return languageRepository.findLangByCode(langCode).orElse(null);
}
```

**There is no `@Transactional` on `DefaultLanguageService.getLanguageByCode`.** On a hit, CacheInterceptor returns and no method body runs. **No connection is acquired on a HIT.** On a miss, `languageRepository.findLangByCode` (`LanguageRepository.java:24-35`) opens its own tx via Spring Data JPA's `SimpleJpaRepository` (`@Transactional(readOnly = true)` at the framework level) — so a miss does acquire a connection, but only for the duration of one SELECT.

`CurrentLanguageFilter` also has a try/catch (`CurrentLanguageFilter.java:43-45`) that **silently swallows any exception** from `getLanguageByCode`, including HikariCP timeout exceptions. So a cold-cache language burst would not surface as 500s on the language path specifically — the request would just continue with `languageContext` unset. **This means `CurrentLanguageFilter` contributes to pool drain but not to 500-count.** Cause-without-symptom.

### Composite per-request worst-case (cold caches, X-Base-Site + X-Lang both set)

A single cold-cache request to `/api/public/translations?namespace=X&lang=Y` with `X-Base-Site: oglasino` and `X-Lang: en` would, on the worst path:

1. `BaseSiteFilter` → `getAllBaseSites` miss → tx → 1 + 3·N queries → connection released
2. `CurrentLanguageFilter` → `getLanguageByCode("en")` miss → 1 query → connection released
3. `TranslationsController` → `languageService.getLanguageByCode("en")` miss again? No — same cache, would now be warm from step 2 → hit → no DB
4. Redis check for the translations namespace+lang → miss? → `indexNamespaceLang` → `translationRepository.findByNamespaceAndLanguage` → 1 query → connection released

So one cold-cache request can acquire and release the connection up to **three times** sequentially. The connection holding is brief each time (a few SELECTs), but each acquisition counts against the 5 s `connection-timeout` if the pool is full at that instant.

---

## Question 3 — `/api/public/translations` — culprit or victim?

### **Victim.**

The `/api/public/translations` endpoint, in isolation, does very little DB work — and on the warm-Redis path, does **zero** DB work. The pool-drain happens upstream, in the filter chain. Trace below.

### Full request-path trace for `GET /api/public/translations?namespace=ERRORS&lang=en`

1. **Filter chain** (same as in `.agent/investigation-redis-caching.md` Step 5b, but with this endpoint):
   - `RequestLoggingFilter` — no DB.
   - Spring Security chain:
     - `InternalTokenFilter` (`security/filter/InternalTokenFilter.java`) — `shouldNotFilter` returns true for non-`/internal/` paths. No DB.
     - `FirebaseAuthFilter` (`security/filter/FirebaseAuthFilter.java:44-92`) — if no `Authorization` header (typical for `/api/public/translations` from a browser), the `isNotBlank(idToken)` check at line 60 short-circuits. No `verifyToken`, no `getCachedAuthData`. **No DB.** If `Authorization` IS present, runs Firebase token verify (network call to Google, ~50-200ms), then `getCachedAuthData(uid)` — `@Cacheable("redisUserAuth", TTL = 1 min)`. Cache hit: no DB. Cache miss: 1 query via `userRepository.findAuthDataByFirebaseUid`.
     - `RateLimitFilter` (`security/filter/RateLimitFilter.java:43-78`) — `categorize` (line 88-137) returns **null** for `/api/public/translations`. The `if (category == null)` short-circuit at line 54 skips the Bucket4j Redis call. **No DB, no Redis. `/api/public/translations` is NOT rate-limited.**
   - `BotFilter` — in-memory User-Agent check. No DB.
   - **`BaseSiteFilter`** (`filter/BaseSiteFilter.java:21-36`) — if `X-Base-Site` header present, calls `baseSiteCacheService.getBaseSiteForCode(code)`. **This is the connection-acquisition site that fails on cold cache** (Question 2 above).
   - **`CurrentLanguageFilter`** (`filter/CurrentLanguageFilter.java:31-46`) — if `X-Lang` header present, calls `languageService.getLanguageByCode(...)`. Same shape; on warm cache no DB; on miss, 1 query (and exceptions are swallowed).
   - `InputSanitizationFilter` (`security/filter/InputSanitizationFilter.java`) — wraps the request. No DB.
2. **Controller** `TranslationsController.getTranslations` (`controller/TranslationsController.java:30-38`):
   - Calls `translationService.getTranslationsForNamespace(TranslationNamespace.valueOf(namespace), lang)`.
3. **Service** `DefaultTranslationService.getTranslationsForNamespace` (`service/impl/DefaultTranslationService.java:106-141`):
   - `languageService.getLanguageByCode(langCode)` (`:114`) — `@Cacheable`, warm cache hit by now (filter would have populated it). No DB.
   - `configurationService.getBooleanConfig("cache.translations.redis")` (`:118`) — **in-memory cache** populated at startup by `DefaultConfigurationService.onAppReady` (`service/impl/DefaultConfigurationService.java:30-40`). No DB.
   - `redisTemplate.opsForValue().get(redisKey)` (`:122`, inside `getOrRebuild` at `:233`) — **direct Redis read**, not via `@Cacheable`. No DB.
   - On Redis hit: gunzip + JSON parse + return. **NO DB CALL.**
   - On Redis miss (rare; the entry is pre-populated by `@PostConstruct initTranslations` at `:54-57` for every namespace × language at startup): `getOrRebuild` (`:231-249`) enters a synchronized block keyed on `(namespace + "_" + langCode).intern()`, then calls `indexNamespaceLang(namespace, language)` (`:243`), which runs `translationRepository.findByNamespaceAndLanguage(namespace, language.id())` (`:78`) — **one Spring Data JPA query** opening a per-method tx via `SimpleJpaRepository`. Connection held briefly for that one SELECT.

### Verdict on `/api/public/translations`

- **Steady state (warm caches everywhere)**: zero DB queries, zero JDBC connections. Pure Redis + in-memory.
- **`BaseSiteFilter`-cold path**: 1 + 3·N queries in the filter, one tx, before the controller is even reached. **This is the failure mode.**
- **`CurrentLanguageFilter`-cold path**: 1 query in the filter, one tx, before the controller.
- **Redis-cold for the specific translations entry**: 1 query inside the controller (a `synchronized(intern())` lock serializes refills per `(namespace, lang)` pair — so it's not a thundering herd at the DB even if many requests hit the same cold key at once).

The endpoint itself is **innocent** under all of these paths — the connection acquisition is either in `BaseSiteFilter` ahead of it, or in `CurrentLanguageFilter` ahead of it, or in a brief refill of a translation entry (which is per-key serialized and not what the burst pattern implies).

The bots are hammering `/api/public/translations` because it's a low-friction public endpoint with no rate limit (`RateLimitFilter.categorize` line 88-137 returns null for it). That's why translations shows up in the failure window. The actual victim/culprit split is upstream.

### Quick check for `/api/public/baseSite/overviews`

From `controller/BaseSiteController.java:54-59`:

```java
@GetMapping("/overviews")
public ResponseEntity<List<BaseSiteOverviewDTO>> getAllBaseSiteOverviews() {
  return ResponseEntity.ok()
      .cacheControl(REFERENCE_DATA_CACHE)
      .body(baseSiteFacade.getAllBaseSiteOverviews());
}
```

`baseSiteFacade.getAllBaseSiteOverviews()` (`facade/impl/DefaultBaseSiteFacade.java:32-34`) → `baseSiteCacheService.getAllBaseSiteOverviews()` (`DefaultBaseSiteCacheService.java:17-19`) → `baseSiteService.getAllBaseSiteOverviews()` (cross-bean, proxy fires) → `@Cacheable("redisBaseSiteOverviews")` on `DefaultBaseSiteService.java:30-55`.

Same shape as `getAllBaseSites`, EXCEPT this `@Cacheable` method is **not** `@Transactional`. On a miss, the JPA repo calls inside the method body (`baseSiteRepository.findActiveBaseSiteOverviews()`, then per-overview `findAllowedLanguages`) each open their own Spring Data JPA tx. So a cold-cache miss for `/overviews` opens **several** short transactions back-to-back (one per repo call), which actually trips the pool *more* than `getAllBaseSites` would per request — though each tx is briefer.

Also: this endpoint runs `BaseSiteFilter` ahead of it just like translations does. If the burst hits `/baseSite/overviews` with `X-Base-Site` set, the request acquires connections in BOTH `BaseSiteFilter`'s `getBaseSiteForCode` chain (cold-miss → tx) AND in the controller's `getAllBaseSiteOverviews` (cold-miss → repo calls).

`/baseSite/overviews` is **both a victim** (filter-chain ahead of it) **and a partial culprit** (its own controller path does DB work on miss). Compared to `/translations`, it's more directly exposed.

---

## Question 4 — What holds connections, and for how long

### Hot path — request-entry filters (cold-cache miss scenarios)

| Site | Tx scope | Queries held | Notes |
|---|---|---|---|
| `BaseSiteFilter.doFilterInternal:31` → `DefaultBaseSiteService.getAllBaseSites` (`@Transactional(readOnly = true)`, `DefaultBaseSiteService.java:59`) | One outer tx wrapping the whole `@Cacheable` method body | 1 + 3·N (N = active base sites) — `findActiveBaseSiteCodes`, then for each: `findCoreByCode`, `findAllowedLanguages`, `findAllowedCurrencies` (`DefaultBaseSiteService.java:60-76`) | The single largest connection-holding hot spot on the hot path. With N=5 base sites, 16 SELECTs per cold-miss while a connection is held. |
| `BaseSiteFilter` (same chain) for `getAllBaseSiteOverviews` (called via `BaseSiteCacheService.getBaseSiteOverviewForCode` from converters — see Question 3's `/overviews` analysis) | No outer `@Transactional`; each repo call opens its own | 1 + N queries (`findActiveBaseSiteOverviews`, then per-row `findAllowedLanguages`) | Multiple back-to-back short transactions on miss. Each opens, runs one SELECT, closes. |
| `CurrentLanguageFilter.doFilterInternal:55` → `DefaultLanguageService.getLanguageByCode` (no `@Transactional`; Spring Data JPA does it) | Per-repo-method tx | 1 SELECT per miss | Exceptions silently swallowed (filter try/catch lines 43-45). |
| `FirebaseAuthFilter.doFilterInternal:67` → `DefaultFirebaseAuthService.getCachedAuthData` (`@Cacheable`, **no `@Transactional`**, `DefaultFirebaseAuthService.java:62-69`) | Per-repo-method tx on miss | 1 SELECT (`findAuthDataByFirebaseUid`) — a JOIN-FETCH constructor projection per the comment at `:65` | `redisUserAuth` TTL is **1 minute** (`RedisConfig.java:46`). On the per-user hot path, every authenticated request fetches once per minute. Not in the unauth-translation burst path, but contributes to steady-state DB load for any logged-in user. (Same finding as `.agent/investigation-redis-caching.md` Step 5; reaffirmed here as a connection-holding contributor.) |

### Write-path — connections held across slow external work

| Site | Tx scope | What's held during | Severity |
|---|---|---|---|
| `DefaultProductService.createProduct` (`:93-182`) — class-level `@Transactional(rollbackFor = Exception.class)` at `:67` | Whole-method tx | **OpenAI HTTP round-trip** via `openAIFacade.translateTextWithPrompt` (called from `updateTranslations` at `:514`, which is invoked from `createProduct:113` and `createProduct:119`) — the OpenAI translation call is synchronous and can take seconds. The JDBC connection is held for the entire create-product duration. | High at write-path — but write-path traffic is low (rate-limited by `RateLimitFilter.EXPENSIVE_WRITE`). Real, but not what's exhausting the pool right now. Worth flagging for a future fix brief. |
| `DefaultProductService.updateProduct` (`:212-...`) — same class-level `@Transactional` | Whole-method tx | Same OpenAI call (lines 249-254, 257-262 → `updateTranslations` → `openAIFacade.translateTextWithPrompt`) | Same as createProduct. |
| `CacheWarmupService.warmup` (`config/CacheWarmupService.java:44-95`) — method-level `@Transactional(readOnly = true)` at `:46` | Whole-method tx | All sequential warmup work — `getAllBaseSites`, `getAllBaseSiteOverviews`, `getBaseCurrency`, `getAllLanguages`, per-lang `getLanguageByCode`, AND the Redis writes inside each cached method | One connection held for the duration of warmup at startup. If warmup is slow (cold DB, many languages, many base sites), this is a single connection out of 8/18 held for several seconds at startup. **More importantly**, if traffic arrives during warmup, the 7/17 remaining are fielding requests while warmup also competes for them via internal queries. |
| `ProductIndexer` (`elasticsearch/service/impl/ProductIndexer.java:102, 113, 342`) — `@Transactional(readOnly = true)` on read methods; `@Transactional` on a write method | Per-method | ES indexing happens inside (lines vary). The ES calls are HTTP. Connection held across ES round-trips. | Off the hot path — runs on event-listener triggers or reindex admin endpoint. Worth noting; not blocking. |

### Transactions without a sane timeout

`grep`ed for `@Transactional(.*timeout` and `@Transactional(.*Timeout` across `src/main/java/`: **zero matches**. No `@Transactional` method in the codebase declares a timeout. They rely entirely on `connection-timeout=5000ms` to bound pool acquisition, but once the connection is in hand, the tx can run as long as the queries take. For read-mostly Postgres on DO this is rarely an issue, but `DefaultProductService.createProduct`/`updateProduct` hold the connection across OpenAI calls with no upper bound except the OpenAI client's own timeout (not investigated). If OpenAI hangs for 60 s, the connection is held for 60 s. **Worth flagging.**

### Connection-handling-mode

Not set explicitly (`grep -n "connection.handling_mode" application*.yaml`: no matches; `grep -rn` in `src/`: no matches). Hibernate 6+ default is `DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION` — connection is acquired lazily on first JDBC operation inside the tx and released immediately after commit/rollback. This is the safer default (the alternative, `IMMEDIATE_ACQUISITION_AND_RELEASE_AFTER_STATEMENT`, would be even more pool-friendly but breaks anything that touches multiple statements in one tx). No action item here; just noting.

---

## Question 5 — The concurrent-burst trigger

### Backend-side: nothing forces a fan-out

I searched for:
- WebSocket / server-sent-events fan-out: no SSE/WebSocket endpoints serve translation events. The repo has Firestore-realtime usage in the chat path, not translations.
- Retry middleware: no retry filter, no `RetryTemplate` on the translations service, no `@Retryable` on the service. `RateLimitFilter` rejects with 429 + `Retry-After` (a hint, not an action), but as established `/api/public/translations` is not categorised — so no 429 ever fires on it.
- Cache-control responses that could trigger client re-fetches: `TranslationsController` (`:23-26`) sets `CacheControl.maxAge(5min).cachePublic().staleWhileRevalidate(1d)` — this **reduces** client refetches, not increases them. With it, a browser/CDN can serve a cached translation response for 5 minutes fresh + 1 day stale-while-revalidate. Without it, every page navigation refetches.
- A 401/redirect loop that could trigger a client to fire many translation reqs: not present. The endpoint is in `/api/public/...`, no auth required.

**Verdict on backend cause: there is nothing in the backend that fans out, retries, or amplifies translation requests.** A `~20 concurrent same-IP-same-second` burst originates outside the backend.

### Plausible non-backend causes (NOT determinable from backend code — flagged for completeness, not a conclusion)

- **Web client startup behaviour**: a Next.js / React `useEffect` that fetches multiple translation namespaces in parallel on page load. With 22 namespaces in `TranslationNamespace` and the project using `next-intl`, it's plausible the bootstrap fetches several (or all) namespaces concurrently per locale. Multiple tabs opening, or a bot mimicking a browser scraping multiple paths, would multiply this.
- **Bot / scraper**: same-IP-same-second is consistent with an automated scanner that pulls a page, extracts XHR URLs, and fires them in parallel. The 14:35:49 and 00:33:30 timestamps (the latter is overnight) point at scanners or scheduled crawlers more than at human users.

Neither of these is checkable from this repo's source code. **Not determinable from backend code alone.** The bot mitigation question is out of scope per the brief ("bot-blocking is a separate concern"), but the absence of `RateLimitFilter` coverage on `/api/public/translations` and `/api/public/baseSite/overviews` is the missing guardrail that lets a single IP do this damage. Worth a separate decision on whether public reference-data endpoints should be in `RateLimitFilter.categorize`.

---

## Root cause statement

A burst of concurrent requests exhausts the pool through the following exact mechanism:

1. Each affected request enters `BaseSiteFilter.doFilterInternal` (`filter/BaseSiteFilter.java:21`) carrying an `X-Base-Site` header (set by the web client, by a bot mimicking it, or by the request's `baseSite` query parameter).
2. `BaseSiteFilter` calls `baseSiteCacheService.getBaseSiteForCode(...)` (line 31), which transits `DefaultBaseSiteCacheService` (no proxy interception on its own methods) into `DefaultBaseSiteService.getAllBaseSites()` (cross-bean, proxy fires).
3. `DefaultBaseSiteService.getAllBaseSites()` is annotated `@Cacheable("redisBaseSites") + @Transactional(readOnly = true)` (`DefaultBaseSiteService.java:57-60`). Spring's interceptor chain on this method is `CacheInterceptor` (outer) → `TransactionInterceptor` (inner) → method body — confirmed by the production stack trace.
4. **On a cache HIT, `CacheInterceptor` returns from Redis without entering `TransactionInterceptor`. No connection is acquired.** This refutes the brief's working hypothesis.
5. **On a cache MISS**, `CacheInterceptor` calls `proceed()`, `TransactionInterceptor` opens a JPA transaction, `JpaTransactionManager.doBegin` acquires a JDBC connection from the Hikari pool, the method body runs 1 + 3·N JPA queries, the tx commits, and the connection is released.
6. The cold-cache window happens at any of: (a) post-deploy between `RedisCacheEvictConfig.evictCachesOnStartup` (`config/RedisCacheEvictConfig.java:39-53`, runs as `ApplicationRunner`) and `CacheWarmupService.warmup` (`config/CacheWarmupService.java:44`, `@Order(10)` on `ApplicationReadyEvent`); (b) on 24 h TTL expiry of the `redisBaseSites` bucket (`RedisConfig.java:49`); (c) after a manual admin evict via `CacheAdminController`; (d) if warmup itself silently failed for `redisBaseSites` (it logs a warning and continues — `CacheWarmupService.java:102-104`).
7. During a cold-cache window, **Spring's `CacheInterceptor` has no per-key in-flight deduplication / leader-election**. Every concurrent miss independently calls `proceed()`. 20 concurrent misses → 20 concurrent `doBegin` → 20 concurrent pool acquisitions.
8. The pool at the time of the incident was 8 (`active=8, idle=0`, matching `application-stage.yaml:18` exactly — see "Brief vs reality"). 8 succeed; 13 wait for up to `connection-timeout=5000ms` and fail with `HikariPool-1 - Connection is not available, request timed out after 5001ms` — exactly the log line.
9. The exhaustion is then visible across any endpoint whose filter chain passes through `BaseSiteFilter` (every request hitting `/api/*`) — including `/api/public/translations` (which is otherwise innocent) and `/api/public/baseSite/overviews` (which is itself a culprit when its own `redisBaseSiteOverviews` cache is also cold).
10. **`/api/public/translations` shows up as the failing endpoint because the bot is hammering it** — it has no rate-limit category in `RateLimitFilter.categorize` (`security/filter/RateLimitFilter.java:88-137`) — but the endpoint itself does not touch the DB on warm caches. Its appearance in the failure list is downstream of `BaseSiteFilter`'s cold-cache contention.

The exact code involved:

- `filter/BaseSiteFilter.java:21-36`
- `service/impl/DefaultBaseSiteCacheService.java:35-40` (and `:30-32`)
- `service/impl/DefaultBaseSiteService.java:57-64` (`@Cacheable + @Transactional`)
- `config/RedisConfig.java:24` (`@EnableCaching`, default `LOWEST_PRECEDENCE` order)
- `config/RedisCacheEvictConfig.java:39-53` (startup evict, runs before warmup)
- `config/CacheWarmupService.java:44-95` (warmup, `@Order(10)` on `ApplicationReadyEvent`)
- `application-prod.yaml:13-20` (or `application-stage.yaml:13-23` if the running profile was stage at the time of the incident — see "Brief vs reality")
- `security/filter/RateLimitFilter.java:88-137` (no category for `/api/public/translations`)

---

## Candidate fix directions (brief)

Per the brief, "Do not propose the fix in detail. The report should make the root cause unambiguous and *can* list candidate fix directions briefly." Per-direction trade-offs left for the next brief.

1. **Cache-warmup must complete before the readiness probe flips to ready.** Today, warmup runs on `ApplicationReadyEvent` — which is the same signal Spring uses to flip the readiness probe. Race-free fix: gate readiness on warmup completion, or move warmup into the boot lifecycle phase before `ApplicationReadyEvent`. Either way, the post-deploy cold-cache window stops existing.
2. **Per-key in-flight deduplication on cache miss.** Spring's `CacheInterceptor` doesn't lock. Either (a) add a single-flight pattern (e.g. a `Map<key, CompletableFuture<value>>`) inside `DefaultBaseSiteCacheService` and `DefaultLanguageService` for the leader-election on miss, or (b) keep a long-lived in-memory copy of these reference-data lists with periodic refresh — the data is small and rarely changes, so Redis is overkill on top of in-memory.
3. **Decouple the `@Transactional` from `@Cacheable` on `getAllBaseSites`.** The method runs 1+3·N queries inside one read-only tx; the `@Transactional` is what makes the connection-held duration linear in N. If the read sequence is split, each repo call's own per-method tx is shorter and pool pressure drops correspondingly. (`getAllBaseSiteOverviews` is already structured this way and is comparatively easier on the pool per-call, although each request still does N+1 queries — but not under one outer tx.)
4. **Categorise `/api/public/translations` and `/api/public/baseSite/overviews` in `RateLimitFilter`.** A `PUBLIC_REFERENCE_DATA` bucket capped at, say, 60 req/min per IP would stop a single-IP burst from sequencing 20 in one second — the trigger that exposes the underlying race. Note the project decision documented in memory ("Rate-limiting scope: narrow on purpose; no default catch-all; protect only endpoints with concrete abuse cost") — adding a category here would need Mastermind agreement, since this endpoint has a *concrete abuse cost* (pool exhaustion) but isn't a write or a search.
5. **Bump `connection-timeout` selectively, or `maximum-pool-size`.** If running on prod-config-18, the headroom is much better than on stage-config-8 — but the failure mode still exists at any pool size during a thundering-herd cold miss. Pool sizing is a palliative; the structural fixes above are the lever.
6. **Reconsider class-level `@Transactional` on `DefaultProductService`.** Holding a JDBC connection across `openAIFacade.translateTextWithPrompt` is a real latent bug that will surface in production if OpenAI ever slows down. Separate brief.

---

## What this investigation rules out / confirms

- **A cache HIT on `getAllBaseSites` does NOT acquire a JDBC connection.** Confirmed from interceptor order + the production stack trace's call direction.
- The CacheInterceptor → TransactionInterceptor chain ordering is **CacheInterceptor outer**, per the prod trace.
- `/api/public/translations`'s own controller path is **DB-free on warm caches**. It is a victim of upstream filter-chain pool contention.
- `/api/public/baseSite/overviews` is a partial culprit on cold cache because its own `@Cacheable` method body runs JPA repo calls (without an outer `@Transactional`, but each repo call opens its own tx).
- **Spring's `CacheInterceptor` does not lock per key.** Concurrent misses are all admitted to the chain — the thundering-herd is real.
- **`/api/public/translations` and `/api/public/baseSite/overviews` are not categorised by `RateLimitFilter`** — a single IP can fire arbitrary parallel requests with no throttle.
- **No `@Transactional` in the codebase declares a timeout.** Tx duration is bounded only by the queries it runs.
- **No backend mechanism fans out, retries, or amplifies translation requests.** The burst origin is client-side or external — not determinable from backend code.

---

## For Mastermind

- **Is this bug-sized or feature-sized?** It's bug-sized for the request-path symptom (the load-bearing fix is "make warmup actually complete before traffic arrives" or "deduplicate concurrent misses"), which is a single-focused session. It becomes feature-sized if Mastermind wants to harden the broader pattern at the same time (rate-limit categorisation of public reference-data endpoints, removal of `@Transactional` on `getAllBaseSites`, separation of `DefaultProductService`'s write-tx from OpenAI calls, in-memory long-lived reference-data cache as a structural answer to "Redis-as-the-only-defence"). My recommendation: **scope the bug fix tight (warmup-gating + miss-dedup on `getAllBaseSites`/`getLanguageByCode`)** and queue the structural items as separate briefs. Prevents this incident class from recurring without rewriting the cache layer.
- **The stage-vs-prod profile discrepancy is the first thing to confirm.** `application-prod.yaml:15` declares `maximum-pool-size: 18` but the production log evidence shows `total=8`, which matches `application-stage.yaml:18` exactly. Two paths forward:
  - The "prod" deployment is actually running with the stage profile (env var or compose mistake). Fix the deploy.
  - Or the log lines were genuinely from staging and someone called them prod. Then the actual prod incident may have different numbers, and the size-of-fix question depends on what they are.
  - Either way, no fix should be tuned to a pool size without confirming which one is live. Igor can confirm this in seconds by reading `/opt/oglasino/.env` on the box.
- **The connection-timeout=5000ms is doing exactly what it's supposed to.** Don't bump it without first deciding whether 5 s is the right user-facing failure latency. The incident says "20 reqs fail at 5 s" — bumping to 10 s would say "20 reqs fail at 10 s." Doesn't help the user; just changes the failure shape.
- **Adjacent observations (low-severity, off-scope, not fixed):**
  - `CurrentLanguageFilter.doFilterInternal:43-45` silently swallows ALL exceptions, including `HikariPool` timeouts and any `IllegalArgumentException` from invalid lang codes. This is the same asymmetric-handling note from `.agent/investigation-redis-caching.md` Step 5b. Connection-pool failures here would be invisible in logs (no stack, no level=ERROR). Worth a logged warning at minimum.
  - `RedisCacheEvictConfig.evictCachesOnStartup` clears the caches every restart, including caches that warmup will then refill. Warmup's existence acknowledges this is intentional ("avoid stale entries from previous deploys"). But the gap between evict-completes and warmup-completes is the very window the incident exploits. The "right" sequence is: evict, then warm, then flip readiness. Today readiness flips concurrent with warmup starting.
  - No `@Transactional` method in the codebase declares a timeout. `DefaultProductService.createProduct/updateProduct` could hold a JDBC connection through an arbitrarily-long OpenAI call. Not the current incident, but a latent connection-leak vector.
  - `application-stage.yaml`'s pool of 8 with `minimum-idle: 2` is unusually small for a stack with `tomcat.max-threads: 100`. The ratio (max-threads : pool-size) is 12:1, which is very aggressive. Even modest concurrency on stage would saturate the pool. If the running prod is actually using stage's pool, the math is unforgiving: ~12 simultaneous DB-touching requests fully saturate.

- **One thing I deliberately did not do:** I did not look at the production deploy config files (`/opt/oglasino/.env`, `docker-compose.yml` on the box) because they're outside the repo and outside the read-only brief. The stage-vs-prod question above can only be settled by reading those.
