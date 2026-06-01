# Redis cache infrastructure

How Redis is used in `oglasino-backend`. Reference document for engineers touching cache code.

## Managed caches

The backend has 8 admin-managed caches plus internal Spring caches not exposed to the admin surface. The 8 admin-managed caches are enumerated in `AdminCacheDescriptor` (see `oglasino-backend/.../AdminCacheDescriptor.java`):

| Cache name | Owner | Versioned | Warmup support |
|---|---|---|---|
| `redisUserInfo` | per-user lazy load | No | No (per-request) |
| `redisUserAuth` | per-user lazy load | No | No (per-request) |
| `redisBaseSiteOverviews` | `CacheWarmupService` | No | Yes |
| `redisLanguage` | `CacheWarmupService` | No | Yes |
| `redisLanguages` | `CacheWarmupService` | No | Yes |
| `redisBaseCurrency` | `CacheWarmupService` | No | Yes |
| `redisBaseSite` | `VersionChecksumService` | Yes | Yes |
| `redisTranslations` | `VersionChecksumService` | Yes | Yes |

**Versioned caches** carry a content-derived checksum persisted in `configuration`. Boot rebuild is conditional — only when checksum changed or Redis is empty. Admin edits trigger async rebuild (`@Async` via `VersionChecksumService.rebuildTranslationCacheAsync`).

**Non-versioned caches** are unconditionally rebuilt at boot by `CacheWarmupService`. No content checksum.

**User caches** populate lazily on per-request reads. Admin can delete (e.g., to force re-auth) but not warmup — they have no precomputed shape.

## Soft replacement

For versioned caches, content updates use **soft replacement**: the rebuild path writes the new value via `cacheManager.getCache(name).put(key, value)`, atomically replacing the old. Never `DEL` first. Users never see empty cache during a rebuild.

The boot recompute path follows the same order:

1. Compute new checksum from DB.
2. Check Redis key existence.
3. If rebuild needed: build new value from DB, write to Redis atomically.
4. **Only after Redis write succeeds**: persist new checksum to `configuration`.

If a Redis write fails mid-rebuild, the checksum is not persisted; next boot retries.

## Boot ordering

1. **`@Order(1)`** — `DefaultConfigurationService.onAppReady`: loads `configuration` table into in-memory cache.
2. **`@Order(2)`** — `CatalogManager.initAll`: syncs catalog data into DB if needed.
3. **`@Order(3)`** — `VersionChecksumService.onAppReady`: rebuilds versioned caches (per-base-site `redisBaseSite::*` and per-namespace-per-language `redisTranslations::*`). Conditional on checksum change or empty Redis.
4. **`@Order(10)`** — `CacheWarmupService.warmup`: unconditionally rebuilds non-versioned caches and flips readiness to `ACCEPTING_TRAFFIC` via `AvailabilityChangeEvent`.

Anything mobile or web requests after readiness flip sees warm versioned + non-versioned caches.

## Spring cache key format

Spring's `RedisCacheManager` writes keys as `<cacheName>::<key>` by default. For example:
- `redisBaseSite::rs`, `redisBaseSite::rsmoto`, `redisBaseSite::me`
- `redisTranslations::COMMON:en`, `redisTranslations::COMMON:sr`, etc.

The `::` separator comes from `CacheKeyPrefix.simple()`. The `VersionChecksumService.redisKeyExists` helper constructs keys using this format.

## `@Lazy` self-injection pattern

When a class with `@Cacheable` methods calls its own cached methods, the call must go through the Spring proxy or `@Cacheable` is bypassed. Use `@Lazy` self-injection:

```java
@Lazy
@Autowired
private DefaultBaseSiteService self;

public List<BaseSiteDTO> getAllBaseSites() {
    return baseSiteRepository.findActiveBaseSiteCodes().stream()
        .map(self::getBaseSiteByCode)  // goes through @Cacheable proxy
        .toList();
}

@Cacheable(value = "redisBaseSite", key = "#code", sync = true)
public BaseSiteDTO getBaseSiteByCode(String code) {
    return loadBaseSiteByCodeFromDb(code);
}
```

Documented as a blessed pattern in [`conventions.md`](../meta/conventions.md) Part 13. Precedents in this codebase: `DefaultBaseCurrencyService`, `DefaultBaseSiteService`, `DefaultTranslationService`.

## `TransactionTemplate` for boot rebuilds

When a boot path needs read-only transactional context for lazy-load access **and** a separate read-write call for persistence (e.g., update `configuration`), wrapping the entire method in `@Transactional(readOnly = true)` would propagate the read-only flag to the persistence call and Postgres rejects the write.

Use `TransactionTemplate` to scope read-only transactions narrowly:

```java
TransactionTemplate readOnlyTx;

void rebuildBaseSiteCachesIfNeeded() {
    for (BaseSite bs : baseSites) {
        String currentChecksum = readOnlyTx.execute(status -> computeCatalogChecksum(bs));
        // ... checksum compare, Redis write ...
        persistChecksum(configKey, currentChecksum);  // implicit read-write context
    }
}
```

Documented as a blessed pattern in [`conventions.md`](../meta/conventions.md) Part 13. Precedent in this codebase: user-deletion feature (2026-05-13), `VersionChecksumService` (this feature).

## Loader extraction for cache-bypass reads

When a rebuild path needs the same DB read that `@Cacheable` performs on cache miss — without touching the cache — extract the DB-read body into a separate `loadXFromDb` method. The `@Cacheable` method becomes a thin wrapper:

```java
@Override
@Cacheable(value = "redisBaseSite", key = "#code", sync = true)
@Transactional(readOnly = true)
public BaseSiteDTO getBaseSiteByCode(String code) {
    return loadBaseSiteByCodeFromDb(code);
}

@Override
@Transactional(readOnly = true)
public BaseSiteDTO loadBaseSiteByCodeFromDb(String code) {
    // The DB-read body. Direct repository calls, mapToDTO, return.
}
```

`VersionChecksumService.rebuildBaseSiteCache` calls `loadBaseSiteByCodeFromDb` directly (bypasses cache), then writes the result to Redis. Pattern repeated for `loadTranslationsFromDb`.

## Admin cache page contract

The admin surface at `oglasino-web` `/admin/cache` consumes three backend endpoints, all under `/api/secure/admin/cache`:

- `GET /list` — returns `[{ name, warmupSupported }]` for all 8 admin-managed caches. Frontend renders rows from this list.
- `POST /{cacheName}/clear` — evicts the named cache (validated against `AdminCacheDescriptor`). 204 success, 400 unknown.
- `POST /{cacheName}/warmup` — populates the named cache, routing internally to the right service. 204 success, 400 unknown or warmup-unsupported.

The "Obriši + Zagrej" per-row action on the admin page is two sequential client-side calls (clear, then warmup). Bulk operations on the admin page (Obriši sve, Osveži sve, Samo zagrevanje) are also client-side orchestrations via `Promise.all` over per-cache endpoints.

## Public versioning endpoint

`GET /api/public/versions` returns:

```json
{
  "translations": {
    "COMMON": "a3f2b1c8d4e7f091",
    "VALIDATION": "...",
    ... 22 namespaces total
  },
  "catalog": {
    "rs": "...",
    "rsmoto": "...",
    "me": "..."
  }
}
```

`Cache-Control: max-age=300, public, stale-while-revalidate=86400`. Public route under `permitAll()`. Designed for mobile (Expo) to poll, compare to last-seen checksums, and selectively refetch changed namespaces or base sites.

## See also

- [`features/version-checksums.md`](../features/version-checksums.md) — feature spec.
- [`conventions.md`](../meta/conventions.md) Part 13 — `@Lazy` self-injection and `TransactionTemplate` patterns.
- [`decisions.md`](../decisions.md) 2026-05-17 — cache TTL precedent (no TTL on reference-data caches, freshness owned by explicit eviction/rebuild).
- [`decisions.md`](../decisions.md) 2026-05-14 — `AvailabilityChangeEvent` readiness gate, warmup-gated boot.
