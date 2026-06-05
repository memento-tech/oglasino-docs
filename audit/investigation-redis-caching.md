# Investigation — Redis caching bypass

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Type:** Read-only investigation, no code changes.

---

## TL;DR

**Update (session 2):** the original Step 2 caller tables for `redisBaseSites`, `redisBaseSiteOverviews`, and `redisLanguage` only listed *direct* callers of the `@Cacheable` method and skipped the request-entry filter chain (`BaseSiteFilter`, `CurrentLanguageFilter`) and the `*Context` consumers. Those tables have now been replaced with full request-path chains. **The original conclusion survives unchanged**: every entry into these caches on the request hot path crosses the proxy, hits Redis (warm), and does not hit the DB. There is no proxy bypass on `BaseSite` or `Language` — it's *shown* below now, not just asserted. The 404 path is included explicitly: bots hitting unknown routes without `X-Base-Site` / `X-Lang` / Bearer headers do **zero** BaseSite or Language DB / cache work.

The brief's prime suspect — **Spring proxy-based bypass on the `BaseSite` caches** — is **not what's happening**. The two `BaseSite` caches (`redisBaseSites`, `redisBaseSiteOverviews`) go through the proxy correctly on every request hot path. Their cross-bean delegation (`BaseSiteCacheService → BaseSiteService`) was structured to avoid the self-invocation problem, and it does.

What I *did* find:

1. **`redisUserAuth` has TTL = 1 minute.** Configured in `RedisConfig` line 46. This cache is hit by `FirebaseAuthFilter` on **every authenticated request**. With a 60-second TTL, every active user's row is re-fetched from DB at least once per minute regardless of how many requests they make. **This is the most plausible single explanation for ordinary request traffic translating into ordinary DB load on the auth path.** It's a design choice, not a bypass — the comment at `RedisConfig:43-46` calls it "a safety net" — but the trade-off has real cost. Worth a follow-up conversation about whether 1 minute is the right value.

2. **`DefaultBaseCurrencyService` self-invokes its own `@Cacheable` method.** `convertToBaseCurrency()` calls `this.getBaseCurrency()` (`DefaultBaseCurrencyService.java:91`) — a textbook Spring proxy bypass. Same pattern in `updateBaseCurrency()` (`:55`). The bypassed call hits `currencyRepository.findByBaseCurrencyTrue()` every time. The hot consumer is `PriceQueryGenerator.convertToBaseCurrency()` in price-range-filtered product searches: each such search makes **two** redundant `findByBaseCurrencyTrue()` queries. This is a real bypass with a measurable cost on a real request path, but it's narrower than the brief implied (only fires on price-filtered searches, not all searches).

3. **All caches are cleared on every app restart** (`RedisCacheEvictConfig.evictCachesOnStartup`). Warmup (`CacheWarmupService`) repopulates the global caches afterward, but `redisUserInfo` and `redisUserAuth` are per-user and stay cold until each user requests. After a deploy, the first request from each authenticated user costs a DB hit on both caches. Not a steady-state issue, but a "spike after every deploy" issue.

The `BaseSite` caches themselves work. If Igor's observation was specifically that `BaseSite`-related DB queries are showing up in heavy load, the most likely explanation is that the underlying queries fire from the `redisUserInfo` repopulation path (`mapProjectionToUserInfo` → `baseSiteCacheService.getBaseSiteOverviewForCode(...)` → cached, no DB unless `redisBaseSiteOverviews` is also cold) — **not** from the BaseSite cache itself being bypassed.

---

## Brief vs reality

Three discrepancies between the brief and the code that are worth surfacing before drawing conclusions:

1. **Cache count.** Brief says "5–6 caches total." Actual count is **7**: `redisUserInfo`, `redisUserAuth`, `redisBaseSites`, `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. Listed in `RedisConfig.java:38-64` and `RedisCacheEvictConfig.java:25-31`. Translations are not counted (read direct from Redis, per the brief's note).
2. **`BaseSite` cache name.** Brief's tentative names were `BaseSiteData` / `BaseSiteOverview`. Actual cache names are `redisBaseSites` and `redisBaseSiteOverviews` (and the service is `BaseSiteCacheService`).
3. **Prime-suspect framing.** The brief frames the `BaseSite` caches as the most likely site of self-invocation bypass. They're not. The class structure (a `DefaultBaseSiteCacheService` that delegates cross-bean to `DefaultBaseSiteService`) was deliberately built to put the `@Cacheable` annotation on the side of the call boundary that the proxy can intercept. It works.

---

## Step 1 — Cache inventory

### Caching enabled

- `@EnableCaching` is on `RedisConfig` (`RedisConfig.java:24`). ✅
- Cache manager is `RedisCacheManager` from `RedisConfig.cacheManager(...)` (`RedisConfig.java:33-70`). ✅

### Cache catalogue

| # | Cache name | TTL | Cached method(s) | Key strategy |
|---|---|---|---|---|
| 1 | `redisUserInfo` | 30 min | `DefaultUserService.getUserInfoForFirebaseUid(String)` (`DefaultUserService.java:34`); `DefaultUserService.getUserInfoForId(Long)` (`:44`) | `key = "#firebaseUid"` and `key = "#userId"` respectively. Two entries per user. |
| 2 | `redisUserAuth` | **1 min** | `DefaultFirebaseAuthService.getCachedAuthData(String)` (`DefaultFirebaseAuthService.java:64`) | `key = "#firebaseUid"` |
| 3 | `redisBaseSites` | 24 h (1440 min) | `DefaultBaseSiteService.getAllBaseSites()` (`DefaultBaseSiteService.java:60`) | default (no args, single bucket) |
| 4 | `redisBaseSiteOverviews` | 24 h | `DefaultBaseSiteService.getAllBaseSiteOverviews()` (`DefaultBaseSiteService.java:31`) | default (single bucket) |
| 5 | `redisLanguage` | 24 h | `DefaultLanguageService.getLanguageByCode(String)` (`DefaultLanguageService.java:27`) | `key = "#langCode"` |
| 6 | `redisLanguages` | 24 h | `DefaultLanguageService.getAllLanguages()` (`DefaultLanguageService.java:21`) | default (single bucket) |
| 7 | `redisBaseCurrency` | 24 h | `DefaultBaseCurrencyService.getBaseCurrency()` (`DefaultBaseCurrencyService.java:48`) | default (single bucket) |

### Eviction sites

- `redisUserInfo`: `@Caching(evict = ...)` on `DefaultUserService.saveUser(User)` (`:63-74`) evicts both id-keyed and firebaseUid-keyed entries. Same on `evictUserInfoCache(User)` (`:102-106`). Plus three explicit programmatic invocations in `DefaultProductService` (`:154`, `:380`, `:422`) on product create / delete / state-change.
- `redisUserAuth`: `@CacheEvict` in the same `@Caching` block on `saveUser` (`:73`). Plus the 1-minute TTL.
- `redisBaseSites` / `redisBaseSiteOverviews` / `redisLanguage` / `redisLanguages` / `redisBaseCurrency`: **no programmatic eviction.** TTL handles staleness for all of them. Admin endpoints (`/api/secure/admin/cache/evict[/{name}]` in `CacheAdminController`) can clear any of them on demand.
- All seven caches are cleared on app start by `RedisCacheEvictConfig.evictCachesOnStartup` (`RedisCacheEvictConfig.java:40-53`).

---

## Step 2 — Per cache, read path & proxy verdict

For each `@Cacheable` method, every caller traced. Where the original (session 1) tables stopped at the *immediate* `@Cacheable` caller, the tables below trace the chain back to the request entry point. `*Context` consumers are listed too — they don't fire DB/cache work themselves (the `@RequestScope` beans are dumb DTO holders), but readers should see them in the chain rather than have to take that on faith.

### Filter chain context (read this once before the per-cache tables)

The `oglasino-backend` request entry has these filters, ordered by `Ordered`/registration:

| Order | Filter | Wired by | Touches BaseSite / Language? |
|---|---|---|---|
| `Integer.MIN_VALUE` (HIGHEST_PRECEDENCE) | `RequestLoggingFilter` | `LoggingConfig.java:21-25` (manual `FilterRegistrationBean`) | No. MDC + access log only. |
| `-100` (Spring Security default) | Spring Security chain — contains `InternalTokenFilter` (`/internal/*` only), `FirebaseAuthFilter` (Bearer-token only), `RateLimitFilter` (categorised paths only) | `SecurityConfig.java:54-86` | `FirebaseAuthFilter` reads `redisUserAuth` (covered separately below). The other two don't touch BaseSite/Language. |
| `Integer.MAX_VALUE` (LOWEST_PRECEDENCE — Spring Boot default for `@Component` filters with no `@Order`) | `BotFilter`, `BaseSiteFilter`, `CurrentLanguageFilter`, `InputSanitizationFilter` (relative order between these is undefined — none has `@Order`) | All four are `@Component`, auto-registered as servlet filters | `BaseSiteFilter` reads `redisBaseSites` (via cache service), `CurrentLanguageFilter` reads `redisLanguage`. The other two don't. |
| n/a | `DispatcherServlet` → controller (or 404 / 401) | Spring MVC | n/a |

There are **no `HandlerInterceptor` implementations** in the codebase (verified — grep returned only the `WebMvcConfigurer` import nothing). All request-entry work is in the filters above.

`@RequestScope` beans `BaseSiteContext` (`context/BaseSiteContext.java`) and `LanguageContext` (`context/LanguageContext.java`) are plain holders. Setters are called once by the filter; getters return the held DTO. Neither triggers a service call when read.

### `redisBaseSites` — `DefaultBaseSiteService.getAllBaseSites()`

Full request-path chain (warm cache):

```
HTTP request → BaseSiteFilter.doFilterInternal (filter/BaseSiteFilter.java:21)
  → if X-Base-Site header (or `baseSite` query param) is set:
     baseSiteCacheService.getBaseSiteForCode(code) (BaseSiteFilter.java:30)
       → DefaultBaseSiteCacheService.getBaseSiteForCode(code) (DefaultBaseSiteCacheService.java:35)
         → this.getAllBaseSites() (DefaultBaseSiteCacheService.java:36 — self-call, but the
           callee is NOT @Cacheable, so self-invocation is benign)
           → DefaultBaseSiteCacheService.getAllBaseSites() (line 30)
             → baseSiteService.getAllBaseSites() (line 31 — CROSS-BEAN, hits proxy)
               → @Cacheable("redisBaseSites") on DefaultBaseSiteService.getAllBaseSites
                 → cache hit: deserialise list from Redis, return.
                 → cache miss: 1 DB query (findActiveBaseSiteCodes) + N×3 (per code) → cache store → return.
  → baseSiteContext.setCurrentBaseSite(matchedDto)
```

| Caller (full chain) | Entry hop file:line | @Cacheable hop file:line | Crosses proxy? |
|---|---|---|---|
| HTTP request → `BaseSiteFilter.doFilterInternal` (only when `X-Base-Site` header or `baseSite` param present) | `filter/BaseSiteFilter.java:30` | `DefaultBaseSiteService.java:60` (via cross-bean from `DefaultBaseSiteCacheService.java:31`) | **YES** — chain crosses the proxy at the `BaseSiteCacheService → BaseSiteService` boundary. |
| HTTP request → controller / facade reading `BaseSiteContext` (`DefaultProductsSearchFacade.java:31`, `BaseSiteQueryGenerator.java:18`, `DefaultProductsFilterQueryBuilder.java:26`, `DefaultProductService.java:202`) | reads in-memory `@RequestScope` DTO; **no service call** | n/a — context reader only | n/a (no DB / cache work) |
| `DefaultProductService.createProduct` / `updateProduct` (write path) | `DefaultProductService.java:178`, `:226` (via `baseSiteCacheService.getBaseSiteForCode(...)`) | `DefaultBaseSiteService.java:60` | **YES** — same cross-bean delegation. |
| `EntityUserInfoConverter` (`:66`), `AuthUserConverter` (`:48`), `UserOverviewConverter` (`:35`), `UserDetailsConverter` (`:55`), `ProductDetailsConverter` (`:64`), `DefaultUsersService` (autowired), `CatalogToJsonService` (autowired) — all `baseSiteCacheService.*` calls | various | `DefaultBaseSiteService.java:60` | **YES** — all field-injected `BaseSiteCacheService`, all cross-bean to the proxy. |
| `DefaultBaseSiteFacade.getAllBaseSites()` / `getBaseSiteForCode(...)` | `DefaultBaseSiteFacade.java:23,28` | same | **YES** |
| `CacheWarmupService.warmup()` | `CacheWarmupService.java:53` | `DefaultBaseSiteService.java:60` (direct field-injected `baseSiteService` — bypasses the cache service entirely, but still cross-bean to the proxied `BaseSiteService`) | **YES** |

No caller bypasses the proxy. The `this.getAllBaseSites()` self-call inside `DefaultBaseSiteCacheService.getBaseSiteForCode(...)` (line 36) is the only self-invocation in the chain, and **it does not bypass any cache** because `DefaultBaseSiteCacheService.getAllBaseSites()` is not `@Cacheable` — it just delegates one more hop to `baseSiteService.getAllBaseSites()`, where the proxy actually sits.

**Verdict: every caller goes through the proxy.** Confirmed across 11 distinct entry sites including the per-request `BaseSiteFilter`.

### `redisBaseSiteOverviews` — `DefaultBaseSiteService.getAllBaseSiteOverviews()`

Same call-chain shape as `redisBaseSites`, but no per-request filter consumer — `BaseSiteFilter` only uses `getBaseSiteForCode` (which goes through `redisBaseSites`, not overviews).

| Caller (full chain) | Entry hop file:line | @Cacheable hop file:line | Crosses proxy? |
|---|---|---|---|
| HTTP request → controller path that returns a `UserInfoDTO` (e.g. `GET /api/secure/users/{id}`) → `DefaultUserFacade.getUserData(Long)` (`:72`) → `userService.getUserInfoForId(...)` (cached itself; on miss falls through to) `mapProjectionToUserInfo` (`DefaultUserService.java:113`) → `baseSiteCacheService.getBaseSiteOverviewForCode(...)` (`:133`) | `DefaultUserService.java:133` | `DefaultBaseSiteService.java:31` (via cross-bean from `DefaultBaseSiteCacheService.java:18`) | **YES** |
| HTTP request → ES product search → `ProductDetailsConverter.convert` (`:64`) → `baseSiteCacheService.getBaseSiteOverviewForCode(...)` | `ProductDetailsConverter.java:64` | `DefaultBaseSiteService.java:31` | **YES** |
| `EntityUserInfoConverter.convert` (`:66`) — used during user-info construction whenever `redisUserInfo` is repopulated | `EntityUserInfoConverter.java:66` | same | **YES** |
| Admin user converters: `UserOverviewConverter` (`:35`), `UserDetailsConverter` (`:55`) | various | same | **YES** |
| `DefaultBaseSiteFacade.getAllBaseSiteOverviews()` (`:33`) | `DefaultBaseSiteFacade.java:33` | same | **YES** |
| `CacheWarmupService.warmup()` | `CacheWarmupService.java:55` | `DefaultBaseSiteService.java:31` (direct, cross-bean) | **YES** |

`getBaseSiteOverviewForCode(String)` in `DefaultBaseSiteCacheService` (`:22-27`) self-invokes `this.getAllBaseSiteOverviews()` (`:18`) — same pattern as `getBaseSiteForCode` above. The self-call target is also not `@Cacheable`; it crosses to the cached method on the next hop. No bypass.

**Verdict: every caller goes through the proxy.**

### `redisUserInfo` — `DefaultUserService.getUserInfoFor{Id,FirebaseUid}`

Not on the per-request filter path — only invoked from controllers that return user data.

| Caller (full chain) | Entry hop file:line | Crosses proxy? |
|---|---|---|
| HTTP request → controller → `DefaultUserFacade.getUserData(Long)` → `userService.getUserInfoForId(...)` | `DefaultUserFacade.java:72` | **YES** — `userService` field-injected. |
| HTTP request → controller → `DefaultUserFacade.getUserDataByFirebaseUid(String)` → `userService.getUserInfoForFirebaseUid(...)` | `DefaultUserFacade.java:77` | **YES** |

No internal callers in `DefaultUserService`. **Verdict: every caller goes through the proxy.**

### `redisUserAuth` — `DefaultFirebaseAuthService.getCachedAuthData(String)`

| Caller (full chain) | Entry hop file:line | Crosses proxy? |
|---|---|---|
| HTTP request (with `Authorization: Bearer ...`) → Spring Security chain → `FirebaseAuthFilter.doFilterInternal(...)` → `firebaseAuthService.getCachedAuthData(uid)` | `FirebaseAuthFilter.java:67` | **YES** — `firebaseAuthService` is constructor-injected. The filter bean itself is built by `ApplicationConfig.firebaseAuthFilter(FirebaseAuthService firebaseAuthService, ...)` (`ApplicationConfig.java:23-30`) — Spring resolves the parameter to the proxy bean and passes it in. |

`FirebaseAuthFilter` is **not** `@Component`; it's wired explicitly via the `@Bean` factory in `ApplicationConfig`, then plugged into the Spring Security chain by `SecurityConfig.java:83`. The Bean factory's parameter is the proxy. **Verdict: every caller goes through the proxy.** The author's defensive comment in `DefaultFirebaseAuthService.java:54-57` correctly notes the bypass risk and warns against self-invoking from `getOrCreateUser`. That self-invocation has not been introduced.

### `redisLanguage` — `DefaultLanguageService.getLanguageByCode(String)`

Full request-path chain (warm cache):

```
HTTP request → CurrentLanguageFilter.doFilterInternal (filter/CurrentLanguageFilter.java:31)
  → resolveRequestLanguage(request) (line 36, 48)
    → if X-Lang header set:
       languageService.getLanguageByCode(langCode) (CurrentLanguageFilter.java:55)
         → @Cacheable("redisLanguage") on DefaultLanguageService.getLanguageByCode
           → cache hit: deserialise LanguageDTO from Redis, return.
           → cache miss: 1 DB query (findLangByCode) → cache store → return.
       languageContext.setCurrentLanguage(dto)
  → resolveAuthenticatedUserPreferredLanguage (line 38, 64)
    → reads OglasinoAuthentication.getPreferredLanguage() from SecurityContextHolder
       — already populated by FirebaseAuthFilter from the redisUserAuth-cached
         AuthenticatedUserDTO. NO extra service call here.
       languageContext.setCurrentUserPreferredLanguage(dto)
```

| Caller (full chain) | Entry hop file:line | Crosses proxy? |
|---|---|---|
| HTTP request → `CurrentLanguageFilter.resolveRequestLanguage(...)` → `languageService.getLanguageByCode(langCode)` (only when `X-Lang` header present) | `CurrentLanguageFilter.java:55` | **YES** — `languageService` constructor-injected on the filter (`CurrentLanguageFilter.java:23-28`). The filter is `@Component`, so Spring builds and injects the proxy. |
| `CacheWarmupService.warmup()` (per-language warmup loop) | `CacheWarmupService.java:81` | **YES** — field-injected. |

No other callers. The "preferred language" path inside `CurrentLanguageFilter` (`resolveAuthenticatedUserPreferredLanguage`, line 64) reads from the already-populated `SecurityContextHolder` — it does not call the language service at all. The preferred language was put there by `FirebaseAuthFilter` from the `AuthenticatedUserDTO` that came out of `redisUserAuth`. So one cache (`redisUserAuth`) effectively also feeds the preferred-language slot of `LanguageContext`.

**Verdict: every caller goes through the proxy.** The class name in the brief was `LanguageFilter`; the actual class is `CurrentLanguageFilter` (`filter/CurrentLanguageFilter.java`). There is no separate `LanguageFilter`.

### `redisLanguages` — `DefaultLanguageService.getAllLanguages()`

| Caller (full chain) | Entry hop file:line | Crosses proxy? |
|---|---|---|
| `CacheWarmupService.warmup()` | `CacheWarmupService.java:67` | **YES** — field-injected. |

That's the only call site for this `@Cacheable` method. The cache is effectively a one-shot warm-up store. No request-path consumer reads it. **Verdict: every caller goes through the proxy.**

(Two separate code paths read `LanguageRepository.findAllLanguages()` directly — `DefaultTranslationService.indexTranslations()` at `:63` and `MissingExtraTranslationsService.printMissingTranslations()` at `:31`. Neither uses the cached `LanguageService.getAllLanguages()`. Both bypass `redisLanguages` *as a cache surface* — but they're not on the request hot path: `indexTranslations` runs once at `@PostConstruct`, and `MissingExtraTranslationsService` is `@Profile("dev")` only. Flagged in "For Mastermind" of the session summary as low-severity adjacent observations.)

### `redisBaseCurrency` — `DefaultBaseCurrencyService.getBaseCurrency()`

| Caller | File:line | Crosses proxy? |
|---|---|---|
| `CacheWarmupService.warmup()` | `CacheWarmupService.java:60` | YES — field-injected. |
| `ProductBaseCurrencyUpdater.updateBasePrices()` | `ProductBaseCurrencyUpdater.java:46` | YES — field-injected. |
| `PriceQueryGenerator.getPriceRangeQuery(...)` | `PriceQueryGenerator.java:69` | YES — field-injected. |
| **`DefaultBaseCurrencyService.updateBaseCurrency()`** | `DefaultBaseCurrencyService.java:55` | **NO — self-invocation, bypasses proxy.** |
| **`DefaultBaseCurrencyService.convertToBaseCurrency(...)`** | `DefaultBaseCurrencyService.java:91` | **NO — self-invocation, bypasses proxy.** |

**Verdict: bypassed via self-invocation in two methods on the owning class.**

`updateBaseCurrency()` runs once at startup (`@EventListener(ApplicationReadyEvent.class)` `:42`) and once daily (`@Scheduled(fixedRate = 86400000)` `:53`). Two extra DB hits per day — negligible.

`convertToBaseCurrency(String, BigDecimal)` (`:90-105`) is the load-bearing one. Its external callers are:
- `PriceQueryGenerator.getPriceRangeQuery(...)` (`PriceQueryGenerator.java:71` and `:74`) — **2 calls per price-range-filtered product search.**
- `DocumentProductConverter` (`:109`) — per-document during ES indexing (one-shot during reindex, not per request).
- `ProductBaseCurrencyUpdater.updateBasePrices()` (`:84`) — per-document during the daily 03:00 update job.

The `PriceQueryGenerator` path is the one that touches a real request hot path. Each price-range-filtered product search emits **2 redundant `currencyRepository.findByBaseCurrencyTrue()` queries** that should have been served from `redisBaseCurrency`. They're cheap individually, but they multiply by search volume.

The first call in `getPriceRangeQuery` — `baseCurrencyService.getBaseCurrency()` at `PriceQueryGenerator.java:69` — *does* go through the proxy and hits the cache. So the `redisBaseCurrency` cache is populated normally. The bypass only fires when `convertToBaseCurrency` is reached, because the self-invocation inside that method skips the proxy and goes straight to `currencyRepository.findByBaseCurrencyTrue().orElseThrow()`.

---

## Step 3 — Per cache, key alignment

| Cache | Read keys | Write keys | Evict keys | Aligned? |
|---|---|---|---|---|
| `redisUserInfo` | `#firebaseUid` (String) and `#userId` (Long) | same as read | `#user.id` and `#user.firebaseUid` (in `evictUserInfoCache`); `#currentUser.id` and `#currentUser.firebaseUid` (in `saveUser`) | ✅ Read & evict resolve to the same string keys. |
| `redisUserAuth` | `#firebaseUid` | same | `#currentUser.firebaseUid` | ✅ |
| `redisBaseSites` | none (default — single bucket) | same | none | ✅ |
| `redisBaseSiteOverviews` | none | same | none | ✅ |
| `redisLanguage` | `#langCode` | same | none | ✅ |
| `redisLanguages` | none | same | none | ✅ |
| `redisBaseCurrency` | none | same | none | ✅ |

**Verdict: write & read keys align for all seven caches.**

One observation, not a bug: `redisUserInfo` mixes two key namespaces (Long ids and String firebase UIDs) in the same cache. Spring's default key generator stringifies both, and the two domains don't overlap in practice (firebase UIDs are 28-character random strings, ids are integers), so collision is essentially impossible. But it does mean a single user has two distinct cache entries that must both be evicted together — which the existing `@Caching(evict = {...})` block correctly does.

---

## Step 4 — Per cache, eviction audit

| Cache | Triggers | Verdict |
|---|---|---|
| `redisUserInfo` | `saveUser` (annotation), explicit `evictUserInfoCache(user)` from `DefaultProductService` on product create / delete / state-change. | **Appropriate.** Every state change that affects the cached `activeProducts` count or shortBio dual-evicts both keys. The deferred-fix note in `DefaultUserFacade.java:108-122` about `userTranslationsService.generateUserTranslations` running after `saveUser`'s evict is correctly self-aware and a noted follow-up. |
| `redisUserAuth` | `saveUser` (annotation) + 1-minute TTL. | **Appropriate triggers, but see Step 5 commentary on the 1-minute TTL.** |
| `redisBaseSites`, `redisBaseSiteOverviews` | None programmatic. 24-h TTL. Admin can evict via `/api/secure/admin/cache/evict[/{name}]`. | **Appropriate** if admin edits to base sites are rare and admins know to call the evict endpoint. No code path mutates BaseSite rows from a user request, so the absence of `@CacheEvict` is correct. |
| `redisLanguage`, `redisLanguages` | None programmatic. 24-h TTL. Admin-evictable. | **Appropriate.** Languages are reference data. |
| `redisBaseCurrency` | None programmatic. 24-h TTL. Admin-evictable. | **Appropriate.** Currency rows change rarely. |
| **All seven** | `RedisCacheEvictConfig.evictCachesOnStartup` clears every cache on app boot. | **Appropriate but with a side effect** — see Step 5. |

**No over-eviction observed.** No cache is wiped on a hot-path write, no `allEntries=true` on a frequently-called method, no eviction on every request. The eviction architecture is conservative.

---

## Step 5 — Caching wiring & Redis fallback

- `@EnableCaching` ✅ (`RedisConfig.java:24`).
- Cache manager is `RedisCacheManager` ✅ — bean defined explicitly in `RedisConfig.cacheManager(...)`. Because the bean is defined manually, Spring Boot's auto-config does **not** layer in a competing manager.
- `spring.cache.type: redis` is set in both `application-dev.yaml` (line 48) and `application-prod.yaml` (line 61). With this set, Spring won't silently substitute a `ConcurrentMapCacheManager` if Redis fails — the only manager wired is the Redis one.
- Redis connection config: `application-prod.yaml` lines 53-58 reads `${DATA_REDIS_HOST}`, `${DATA_REDIS_PORT}`, `${DATA_REDIS_PASSWORD}`, `${DATA_REDIS_TIMEOUT}` from env. The comment in `RedisConfig.java:27-30` explicitly notes that defining `LettuceConnectionFactory` manually with no-args would silently pin to `localhost:6379` — and that the manual bean has been removed for exactly that reason. Currently Spring Boot's `RedisAutoConfiguration` builds the connection factory from the YAML properties. ✅
- `management.health.redis: { enabled: true }` (`application-prod.yaml:110`) — Redis is part of the readiness check. If Redis is unreachable at boot, the health probe fails. So the deploy would not flip to ready and traffic would not route. ✅
- `RedisCacheConfiguration.disableCachingNullValues()` (`RedisConfig.java:35`) is applied to the default config used for every per-cache config. Empty `Optional` results from `getCachedAuthData` and `getUserInfoForId` are not cached, which is correct.

**No silent Redis fallback path exists.** If Redis is unreachable at runtime, cache reads/writes throw a `RedisCommandTimeoutException` or similar, which would surface as 500s — visible. They would not silently degrade to "always hits DB."

### The two TTL/eviction observations worth Mastermind-level discussion

These are not bypasses, but they shape the cache's effective hit rate:

- **`redisUserAuth` TTL = 1 minute** (`RedisConfig.java:46`). On the per-request hot path. Even when "working perfectly," every active user re-fetches `findAuthDataByFirebaseUid` from Postgres at least once per minute. The author's comment frames this as defense-in-depth, accepting that the explicit `@CacheEvict` on `saveUser` is the real freshness mechanism. If steady-state DB load is dominated by `findAuthDataByFirebaseUid`, this is the explanation, and the lever is the TTL. The current evictions look complete to me — bumping TTL substantially (e.g., 30 min like `redisUserInfo`, or longer) should be safe given the explicit invalidations.
- **All caches cleared on every restart** (`RedisCacheEvictConfig.evictCachesOnStartup`). `CacheWarmupService` immediately repopulates the global ones (`redisBaseSites`, `redisBaseSiteOverviews`, `redisBaseCurrency`, `redisLanguages`, `redisLanguage`), but `redisUserInfo` and `redisUserAuth` go cold and stay cold until each user's first request. After every deploy, every active user pays one DB hit on each. If the deploy cadence is high or the pod cycles often, that produces visible DB load. This is intentional design (avoid stale entries from previous deploys) but worth being explicit about.

---

## Step 5b — The 404 / bot path, hop by hop (added in session 2)

The session-1 report didn't walk this through explicitly. Doing it now.

A request to a non-existent route (`GET /this-does-not-exist`) from a bot or scanner. Assume warm caches, no `Authorization` header, no `X-Base-Site`, no `X-Lang`, an unknown but non-empty `User-Agent`.

| Hop | File:line | DB hit? | Cache fetch? |
|---|---|---|---|
| `RequestLoggingFilter.doFilterInternal` | `logging/RequestLoggingFilter.java:44` | No | No — pure MDC seeding. |
| Spring Security chain enters | `security/config/SecurityConfig.java:54-86` | No | No |
| `InternalTokenFilter` | `security/filter/InternalTokenFilter.java:17` | No — `shouldNotFilter` returns true (path doesn't start with `/internal/`). | No |
| `FirebaseAuthFilter.doFilterInternal` | `security/filter/FirebaseAuthFilter.java:45` | **No** — `idToken` is null (no `Authorization` header), the `if (... isNotBlank(idToken))` block at line 60 is skipped entirely. `getCachedAuthData` is **never called**. SecurityContext stays empty. | **No** |
| `RateLimitFilter` | `security/filter/RateLimitFilter.java:44` | No | No — `categorize(path, method)` returns null for unknown paths (line 88-137 falls through), the `if (category == null)` short-circuit at line 54 skips the bucket lookup entirely. **No Redis call, no DB call.** |
| Spring Security `AuthorizationFilter` | n/a (Spring framework) | No | No — `anyRequest().permitAll()` at `SecurityConfig.java:80-81` allows the request through. Unknown paths are not in the `/api/secure/**` matcher, so no 401. |
| `BotFilter.doFilterInternal` | `security/filter/BotFilter.java:43` | No | No — User-Agent check is in-memory string compare against a hardcoded list (line 17-32). If the UA matches a *known* bot string (e.g. `Googlebot`), responds 204 and stops the chain. If unknown bot UA: passes through. |
| `BaseSiteFilter.doFilterInternal` | `filter/BaseSiteFilter.java:21` | **No** | **No** — header check at line 24-28: `X-Base-Site` header is null, `baseSite` query param is null, `if (baseSiteCode != null)` at line 29 is false. **`baseSiteCacheService.getBaseSiteForCode(...)` is never called.** `baseSiteContext` remains empty. |
| `CurrentLanguageFilter.doFilterInternal` | `filter/CurrentLanguageFilter.java:31` | **No** | **No** — `resolveRequestLanguage` returns `Optional.empty()` (X-Lang header is null, line 51-53 short-circuits). `resolveAuthenticatedUserPreferredLanguage` returns `Optional.empty()` (no auth in SecurityContext, the `Authentication::isAuthenticated` filter at line 67 fails). **`languageService.getLanguageByCode(...)` is never called.** `languageContext` remains empty. |
| `InputSanitizationFilter` | `security/filter/InputSanitizationFilter.java:24` | No | No — wraps the request only. |
| `DispatcherServlet` resolves the route | n/a | No | No — no handler match, returns 404. |

**404 verdict: zero BaseSite / Language DB hits, zero BaseSite / Language cache fetches.** A bot hammering 404s does not touch the BaseSite or Language caches at all (and doesn't touch the auth cache either, because no Bearer token is provided). It does touch Redis once per request indirectly via `RateLimitFilter` only when the path matches a categorised pattern (`/api/auth`, `/api/secure/products/...`, `/api/public/product/search`, `/api/public/product/seen`, `/api/public/verify-recaptcha`, `/api/public/suggestion`, `/api/secure/openai`) — unknown 404 paths fall outside this list and skip the bucket entirely.

If the bot started setting `X-Base-Site: <something>`, that *would* trigger one Redis fetch per request via `getAllBaseSites` (cache hit after warmup). If the supplied code doesn't match a known base site, `findFirst().orElseThrow()` throws `NoSuchElementException`, and because `BaseSiteFilter.doFilterInternal` has no try/catch, that propagates up the chain and produces a 500. Same shape for `X-Lang` — `CurrentLanguageFilter` line 57-59 throws `IllegalArgumentException` on an unknown lang code, but that one is wrapped in a try/catch (line 35-45) that swallows it silently. Asymmetric handling, not a cache concern; flagged in the session-2 summary as an adjacent observation.

If the bot supplies a valid `Authorization: Bearer <something>`, then `FirebaseAuthFilter.verifyToken` fires Firebase admin SDK token verification (network round-trip to Google, not a DB hit), and on success calls `getCachedAuthData` → cache hit / 1-min-TTL miss to DB. Bots rarely have valid Firebase tokens, so this isn't a typical scenario.

---

## Step 6 — Concrete signal (skipped)

The brief flags Step 6 as optional and only if cleanly doable read-only. I skipped it. Reason: the brief and conventions both forbid me from running `mvn spring-boot:run`, and even local SQL logging would require either a config change (which I won't commit) or an env var override that I can't safely apply against an existing local stack without knowing what Igor has set up. Step 2's call-tracing is self-sufficient on this codebase — every cached method has a small enough caller graph that "every caller goes through the proxy" was directly verifiable from the source.

If Igor wants the corroboration anyway, the minimal local recipe (do not commit) is to add to `application-local.yaml` (or env-override):

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.springframework.cache: TRACE
```

Then exercise an authenticated endpoint twice within 60 seconds and look at the Spring Cache TRACE lines for `redisUserAuth` and `redisUserInfo`. The first request should log `Cache miss` then a `SELECT`; the second should log `Cache hit` and emit no SQL. After 60 seconds the auth cache should miss again — that's the TTL effect, not a bypass.

---

## Per-cache root cause statements (revised in session 2 with full chains)

1. **`redisBaseSites`** — **works correctly.** Verified across **11 distinct entry sites** including the per-request `BaseSiteFilter` and the write-path callers in `DefaultProductService.createProduct/updateProduct`. The chain `BaseSiteFilter → BaseSiteCacheService.getBaseSiteForCode → (self-call to non-cached) getAllBaseSites → BaseSiteService.getAllBaseSites (CROSS-BEAN, proxy fires)` is the request-path read; every hop crosses the proxy at the right boundary. Keys align (single bucket). Eviction is admin-only; appropriate for reference data. TTL 24 h. **No proxy bypass.** **No uncached request-path reads of base-site data.** Conclusion unchanged from session 1; now *shown* with the complete chain.
2. **`redisBaseSiteOverviews`** — **works correctly.** Same chain shape as `redisBaseSites`; consumed on the response path during `UserInfoDTO` / `ProductDetailsDTO` construction. Verified across 6 entry sites including `EntityUserInfoConverter`, `ProductDetailsConverter`, and the admin-user converters. **No proxy bypass.** Conclusion unchanged from session 1.
3. **`redisUserInfo`** — **works correctly.** Every caller (the two `DefaultUserFacade` methods at `:72` and `:77`) crosses the proxy. Dual-key eviction is comprehensive across the user-mutating paths (`saveUser` plus three product-state callers). 30-minute TTL is reasonable. Note: cold after every deploy until each user requests.
4. **`redisUserAuth`** — **works correctly, but designed with a 1-minute TTL** that limits its effectiveness on the per-request hot path. Not a bypass. The single caller is `FirebaseAuthFilter.doFilterInternal:67`, which is constructed in `ApplicationConfig.firebaseAuthFilter` with the proxy bean as a constructor argument — so the per-request call crosses the proxy correctly. If observed DB load is dominated by `findAuthDataByFirebaseUid`, the lever is TTL, not the cache wiring.
5. **`redisLanguage`** — **works correctly.** Per-request call site is `CurrentLanguageFilter.resolveRequestLanguage:55`, fired only when the `X-Lang` header is present; `languageService` is the proxy. The "preferred language" slot of `LanguageContext` is populated from `OglasinoAuthentication.getPreferredLanguage()` — already fed by `redisUserAuth`, no additional language lookup. **No proxy bypass on the request path.** Note for completeness: `DefaultTranslationService.indexTranslations` (`:63`, `@PostConstruct`) and `MissingExtraTranslationsService.printMissingTranslations` (`:31`, `@Profile("dev")` only) bypass `redisLanguage`/`redisLanguages` by calling `languageRepository` directly. Neither is on the request hot path. Flagged as adjacent observation, no fix recommended for the investigation.
6. **`redisLanguages`** — **works correctly within its (limited) scope.** The only `@Cacheable`-method caller is `CacheWarmupService.warmup`. No request-path consumer reads `getAllLanguages()` through the cache; the dev-only and startup-only direct-`languageRepository` callers above bypass it but aren't hot-path. The cache exists primarily so `CacheWarmupService` can drive per-language warmup of `redisLanguage`.
7. **`redisBaseCurrency`** — **partially bypassed via self-invocation.** External callers go through the proxy and benefit from the cache. But `DefaultBaseCurrencyService.convertToBaseCurrency()` (`DefaultBaseCurrencyService.java:91`) and `DefaultBaseCurrencyService.updateBaseCurrency()` (`:55`) self-invoke `getBaseCurrency()`, bypassing the proxy. The `convertToBaseCurrency` bypass is the load-bearing one: each price-range-filtered product search via `PriceQueryGenerator.getPriceRangeQuery` makes 2 redundant `currencyRepository.findByBaseCurrencyTrue()` queries. Fix would be to either (a) inject a `@Lazy BaseCurrencyService self;` reference and call `self.getBaseCurrency()`, (b) move `convertToBaseCurrency`/`updateBaseCurrency` to a separate bean, or (c) cache the base currency in a field after the first lookup since base currency is effectively immutable.

---

## What this investigation rules out

- The `BaseSite` caches are not bypassed — confirmed across the full filter chain, not just the immediate `@Cacheable` callers.
- The `Language` cache is not bypassed on the request hot path (one-shot `@PostConstruct` + dev-profile callers do bypass it but are out-of-scope of the load symptom).
- 404 / unknown-route requests do **zero** BaseSite or Language DB / cache work in the absence of `X-Base-Site` / `X-Lang` / Bearer headers (verified hop by hop in Step 5b).
- Keys do not mismatch on any cache.
- No cache is over-evicted.
- Caching is not silently disabled or falling back to in-memory.
- Redis is not silently unreachable (health probe would block readiness).
- There are no `HandlerInterceptor`s or other request-path components left untraced — only filters, all of which were walked in Step 5b.

## What this investigation surfaces (not in the brief's framing)

- `redisUserAuth` TTL of 1 minute is the most likely explanation for steady-state DB load on the auth path.
- `convertToBaseCurrency` self-invokes and produces 2 redundant DB hits per price-filtered search.
- Per-user caches are cold after every deploy.
- Two non-hot-path code paths bypass `redisLanguage`/`redisLanguages` by calling `languageRepository` directly: `DefaultTranslationService.indexTranslations` (`@PostConstruct`, one-shot at startup) and `MissingExtraTranslationsService.printMissingTranslations` (`@Profile("dev")` only). Neither contributes to steady-state DB load. Worth knowing in case translation-loading time at boot becomes a concern.

## Out of scope (per brief)

- No code changes proposed; fix briefs are a separate Mastermind step.
- Translation cache not investigated (read direct from Redis, not `@Cacheable`).
- Bot / 404 traffic not investigated.
- TTL tuning, cache redesign — not actioned, surfaced for Mastermind.
