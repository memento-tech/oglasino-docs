# Audit — backend-calls-reduction (backend, read-only)

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-31
**Scope:** read-only. No code changes.

---

## Q1 — Does `/auth/firebase-sync` read `idToken` from the request body?

**No. The body `idToken` is dead — it is not read on any code path, and the endpoint does not even bind a field for it.** The endpoint `POST /api/auth/firebase-sync` is `AuthController.firebaseSync` (`src/main/java/com/memento/tech/oglasino/controller/AuthController.java:52-54`), which binds the body to `@RequestBody @Valid LoginRequest`. `LoginRequest` (`src/main/java/com/memento/tech/oglasino/dto/LoginRequest.java:6-18`) declares only one field — `displayName` — and **no `idToken` field at all**. So any `idToken` a client sends in the JSON body is an unbound, unknown property that Spring/Jackson silently ignores; there is no getter, no setter, nothing that could read it. The token the handler actually uses is taken exclusively from the `Authorization: Bearer <…>` header: `firebaseAuthService.resolveFirebaseToken(servletRequest)` (`AuthController.java:61`) and again inside `getOrCreateUser` (`AuthController.java:79` → `DefaultFirebaseAuthService.getOrCreateUser`, `DefaultFirebaseAuthService.java:75`). `resolveFirebaseToken` reads `request.getHeader("Authorization")` and returns the bearer substring, returning `null` for anything else — it never touches the request body (`DefaultFirebaseAuthService.java:152-165`). The `LoginRequest` body is consulted only for the optional `displayName` used on first-time user creation (`DefaultFirebaseAuthService.java:114-119`). **Conclusion: mobile can stop sending the body `idToken` safely — nothing reads it. The DTO field does not exist (`LoginRequest.java:6-18`); the token comes from the `Authorization` header (`DefaultFirebaseAuthService.java:152-165`).**

---

## Q2 — How are "active base sites" resolved per request, and is it cached?

**Two distinct backend paths exist. Both are partly Redis-cached (no TTL — evict-driven, warmed before traffic), but both re-run one small uncached Postgres query (`findActiveBaseSiteCodes`) on every call. The per-request `X-Base-Site` resolution runs for all clients (web + mobile), not mobile-only; the list-fetch endpoints are shared, not mobile-specific.**

### Resolution path (per-request, all clients)

Every request whose servlet path starts with `/api` passes through `BaseSiteFilter.doFilterInternal` (`src/main/java/com/memento/tech/oglasino/filter/BaseSiteFilter.java:24-32`), which reads the `X-Base-Site` header (falling back to the `baseSite` query param) and, when present, calls `baseSiteCacheService.getBaseSiteForCode(baseSiteCode)` to resolve and stash it in `BaseSiteContext`. That call chain is:

- `DefaultBaseSiteCacheService.getBaseSiteForCode` (`src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteCacheService.java:35-40`) → `getAllBaseSites()` → `baseSiteService.getAllBaseSites()`
- `DefaultBaseSiteService.getAllBaseSites` (`src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteService.java:58-63`) calls `baseSiteRepository.findActiveBaseSiteCodes()` then maps each code through `self.getBaseSiteByCode(code)`.

### Cached or DB-bound, and TTL

**Mixed — and there is one uncached DB hit per request.**

- `findActiveBaseSiteCodes()` (`src/main/java/com/memento/tech/oglasino/repository/BaseSiteRepository.java:32-33`, JPQL `SELECT b.code FROM BaseSite b WHERE b.active = true ORDER BY b.orderIndex ASC`) is **not cached** — it hits Postgres on **every** request that carries an `X-Base-Site` header (so, on the resolution path, every request). It is a tiny single-column indexed read, but it is unconditional and per-request.
- The per-code payload `getBaseSiteByCode(code)` is `@Cacheable(value = "redisBaseSite", key = "#code", sync = true)` (`DefaultBaseSiteService.java:65-70`) — a **Redis** hit after warmup, no DB.
- `getAllBaseSiteOverviews()` is `@Cacheable(value = "redisBaseSiteOverviews", …)` (`DefaultBaseSiteService.java:30-32`) — fully Redis-cached.
- **TTL:** both `redisBaseSite` and `redisBaseSiteOverviews` are configured with **TTL = null (no expiry)** in `RedisConfig.cacheManager` (`src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:69-77`). Per the comment there (`RedisConfig.java:65-68`), these reference-data caches have no runtime write path: they are populated by `CacheWarmupService` before traffic is admitted (readiness-gated) and refreshed only by explicit operator-driven eviction (`CacheAdminController` / `RedisCacheEvictConfig`). This contrasts with `redisUserInfo` / `redisUserAuth`, which carry a 30-minute backstop TTL (`RedisConfig.java:44-63`).

Net per resolution call: one uncached Postgres `findActiveBaseSiteCodes` query + N Redis hits (one per active base-site code) to rebuild the full `List<BaseSiteDTO>`, then a `.filter(...).findFirst()` to keep the single matching code.

### Per-request for all clients vs mobile-only

**Per-request for all clients — there is no mobile-only base-site endpoint.**

- The `X-Base-Site` resolution above runs in `BaseSiteFilter` for **every** `/api/**` request regardless of client (web SSR and mobile both send `X-Base-Site`), gated only on `path.startsWith("/api")` and the header being present (`BaseSiteFilter.java:24-30`).
- The dedicated *fetch* endpoints live on `BaseSiteController` (`src/main/java/com/memento/tech/oglasino/controller/BaseSiteController.java`, `@RequestMapping("/api/public/baseSite")`) and are shared, not mobile-specific: `GET /details` → `getAllBaseSiteData` → `baseSiteCacheService.getAllBaseSites()` (`BaseSiteController.java:34-39`, the "all active base sites" list mobile fetches); `GET /overviews` (`:42-47`); `GET /{targetBaseSiteCode}` (`:25-31`); `GET /regions/{…}` (`:49-55`). All four attach `ReferenceDataCacheControl.INSTANCE` HTTP cache headers — `Cache-Control: public, max-age=300, stale-while-revalidate=86400` (`src/main/java/com/memento/tech/oglasino/config/ReferenceDataCacheControl.java:8-11`) — so an HTTP-cache-respecting client could avoid re-fetching `/details` for 5 minutes. The backend does not, however, gate or dedupe these calls server-side; if mobile fetches active base sites on every request, the backend serves each one (Redis-backed for the payload, plus the per-call `findActiveBaseSiteCodes` Postgres query when the resolution path is also exercised).

**Call sites summary:**
- Resolution (all clients, per request): `BaseSiteFilter.java:24-32` → `DefaultBaseSiteCacheService.java:35-40` → `DefaultBaseSiteService.java:58-63` → uncached `BaseSiteRepository.java:32-33` + cached `DefaultBaseSiteService.java:65-70`.
- List-fetch endpoint mobile most likely calls: `BaseSiteController.java:34-39` (`GET /api/public/baseSite/details`) → `DefaultBaseSiteFacade.getAllBaseSiteData` (`src/main/java/com/memento/tech/oglasino/facade/impl/DefaultBaseSiteFacade.java:26-29`) → `getAllBaseSites()` (same chain as resolution).
- Cache config (no TTL): `RedisConfig.java:69-77`. HTTP cache header (5 min): `ReferenceDataCacheControl.java:8-11`.
