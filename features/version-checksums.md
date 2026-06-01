# Version checksums

A backend feature that exposes per-namespace translation checksums and per-base-site catalog checksums via a single public endpoint, so mobile can poll one URL and re-fetch only the data that changed. Bundles the long-standing translation Redis-layer cleanup (raw `StringRedisTemplate` → Spring `@Cacheable`) into the same feature.

## Scope

In:
- New `VersionChecksumService` that computes content-based checksums at boot and on admin translation edits, AND owns the rebuild lifecycle for versioned Redis caches (translations and per-base-site catalog-containing base-site caches).
- New configuration keys for storing checksums (22 translation namespaces + 3 base sites = 25 keys, names listed in §4.1).
- New public endpoint `GET /api/public/versions` returning all 25 checksums in one response.
- Refactor of `DefaultTranslationService` from raw `StringRedisTemplate` + gzip + Base64 + synchronized rebuild to Spring's `@Cacheable` / `@CacheEvict` via `RedisCacheManager`.
- Split of the existing `redisBaseSites` cache (one key holding `List<BaseSiteDTO>`) into per-base-site keys (`redisBaseSite::<code>`). Each base site cached independently, rebuilt independently when its catalog checksum changes.
- Soft-replacement semantics for versioned cache rebuilds: build the new value, then atomically replace the old value in Redis. No `@CacheEvict` for versioned caches. Users never observe an empty cache.
- Boot-time rebuilds are synchronous (block readiness flip). Admin-time rebuilds are asynchronous via `@Async` (admin response returns immediately, background thread does the rebuild).
- Retirement of the dead `translations.version` counter and the `cache.translations.redis` toggle.

Out:
- Web adoption. Web has its own caching story; this contract is mobile-shaped.
- Mobile-side implementation. Mobile catch-up chats consume this contract later.
- User-scoped versioning (favorites, profile, etc.) — mobile re-fetches per-request for now.
- `/api/public/config` over-exposure. Pre-existing; tracked as a separate `issues.md` entry.
- `MaintenancePageController.toggle()` trust-boundary. Pre-existing; tracked in the deferred bucket from 2026-05-15. **Resolved by `expo-maintenance-split`** — `MaintenancePageController` is deleted and the toggle moved to admin-gated `POST /api/secure/admin/maintenance/toggle`.
- Removal of `Catalog.currentVersion` and `Category.fileVersion`. The audit's seam #7 establishes content-based catalog checksums make these counter columns useless, but removing them is a follow-up cleanup, not this feature.
- Refactor of `getAllBaseSites` call sites to consume per-base-site cache keys directly. The cache is split now for independent invalidation; callers continue to receive `List<BaseSiteDTO>` via a composition method that reads the three per-base-site keys and assembles the list. If the refactor proves load-bearing, it becomes its own Mastermind chat.
- Checksums for `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. These caches have no runtime write path (per the 2026-05-17 cache TTL decision) and are SQL-seeded only. They stay under `CacheWarmupService` ownership with unconditional boot-time warmup, no checksum tracking.

## Background

Mobile catch-up chats are queued in the Expo backlog (`state.md`). Several of them — translations, catalog, future feature adoptions — need a way to know "did the server's data change since I last cached this?" The existing `translations.version` integer counter is incremented only on admin single-key edits via `POST /api/secure/admin/translations/update`, never on SQL-seed reapplication at boot. A deploy that mutates seed values changes content without bumping the counter. Mobile would never re-fetch.

The catalog has no equivalent exposed counter at all. Two internal version columns (`Catalog.currentVersion`, `Category.fileVersion`) gate boot-time skip/sync logic but aren't exposed anywhere.

This feature replaces both with content-derived checksums computed from actual DB state after boot indexing completes. Mobile polls one endpoint, compares 25 checksums to its locally-cached values, re-fetches only the slices that changed.

## Decisions locked in Phase 1 and Phase 3

1. **Mobile is the only consumer.** Web has its own caching; not in scope.
2. **One endpoint, not two.** `GET /api/public/versions` returns translations and catalog in a single response.
3. **Per-namespace for translations, per-base-site for catalog.** 22 + 3 = 25 checksums in the response.
4. **Checksum-as-version.** No separate integer counter. Mobile compares strings; server stores the same string it returns.
5. **Truncated SHA-256.** Full hash is 64 hex chars; we truncate to first 16 hex chars (64 bits). Collision risk at 64 bits is negligible for "did this slice change" semantics, and the payload drops from ~1.6 KB to ~400 bytes.
6. **Persisted in `configuration` table.** Not in-memory. ~25 pre-seeded keys with empty initial values. First boot computes real values and writes them via `updateConfiguration`. Subsequent boots compare DB-state checksum to persisted checksum, write back only if different.
7. **Content-based, not counter-based.** Catches SQL seed changes (which counter-bumping misses), structural changes (new filter on a category), and base-site-scoped changes (new assignment row).
8. **`Cache-Control: max-age=300`.** Matches existing reference-data endpoint pattern (`/api/public/config`, `/api/public/translations`, `/api/public/baseSite`).
9. **Redis unification bundled into this feature.** Single feature, single spec, single closing decisions entry.

## 3. Architecture

### Cache ownership split

After this feature lands, two services own Redis cache lifecycle, each with one job:

- **`VersionChecksumService`** — owns versioned caches: `redisTranslations` and the new per-base-site `redisBaseSite::<code>` keys. Boot-time: compute checksum from DB, compare to persisted checksum in `configuration`, rebuild the Redis cache only if the checksum changed or the Redis key is empty. Admin-time (translations only): rebuild asynchronously via `@Async`.
- **`CacheWarmupService`** — owns non-versioned caches: `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. Boot-time: unconditional warmup, same behavior as today.

Both services complete before `ApplicationReadyEvent` listeners with `@Order(10)` flip readiness to `ACCEPTING_TRAFFIC`. Mobile cannot poll any endpoint until both have finished their boot work.

### Boot flow (after this feature lands)

1. Spring context refresh. `DefaultTranslationService` bean is constructed but no longer aggressively populates Redis in `@PostConstruct` (see §6 — warmup moved out).
2. `ApplicationRunner` beans: `RedisCacheEvictConfig.evictCachesOnStartup()` clears managed Redis caches.

   **Amended semantics:** `evictCachesOnStartup` no longer clears versioned caches (`redisTranslations`, `redisBaseSite::*`). It continues to clear non-versioned reference caches and user-scoped caches as today. Versioned caches are evicted only by `VersionChecksumService` when it detects a checksum change.

   Rationale: clearing versioned caches on every boot defeats the soft-replacement model — Redis would be empty between cache clearing and `VersionChecksumService.@Order(3)` running, breaking the "users never see an empty cache" invariant.

3. `@Order(1)` `DefaultConfigurationService.onAppReady()` loads `configurationCache` from DB.
4. `@Order(2)` `CatalogManager.onAppReady()` syncs catalog JSON to DB.
5. **`@Order(3)` `VersionChecksumService.onAppReady()`** — new. Synchronous. For each versioned cache: compute checksum from current DB state, compare to persisted checksum. If different OR Redis key is empty, rebuild the Redis value and atomically replace. Persist the new checksum to `configuration` via `updateConfiguration`. Otherwise skip — cache is correct.
6. `@Order(9)` `CacheWarmupService.warmup()` — unconditional warmup of non-versioned caches only.
7. `@Order(10)` Readiness flip to `ACCEPTING_TRAFFIC` via `AvailabilityChangeEvent` (per the 2026-05-14 connection-pool decision).

Steps 5 and 6 both complete before step 7. Boot is slightly slower on a deploy that changes versioned data (rebuild work runs synchronously) and slightly faster on a deploy that doesn't (no warmup of caches that don't need it).

### Admin edit flow (translations)

When `DefaultTranslationService.updateTranslation` runs:

1. DB write via JPA (existing behavior).
2. `backendTranslations` rebuild if namespace is `BACKEND_TRANSLATIONS` (existing behavior).
3. Admin endpoint returns 200 to the admin user immediately. The remaining steps run on a background thread via `@Async`.
4. **Background, `@Async`:** `VersionChecksumService.rebuildTranslationCacheAsync(namespace, language)`:
   - Read all translations for `(namespace, language)` from DB.
   - Build the new `Map<String, String>` value.
   - Atomically replace the Redis key (`redisTranslations::<NAMESPACE>:<langCode>`) — write the new value, which overwrites the old. No prior `@CacheEvict`.
   - Recompute the namespace's checksum from DB and persist to `configuration` via `updateConfiguration`.

During the brief window between admin response (step 3) and async completion (step 4), end-user reads continue to see the old cached value. The contract guarantees mobile sees a consistent (checksum, content) pair eventually, not instantly. The `/api/public/versions` endpoint reflects the old checksum until the async rebuild completes; mobile re-fetches when the checksum updates.

### Admin edit flow (catalog)

Catalog has no runtime admin path per the audit. Boot recomputation at `@Order(3)` is the only trigger.

### Read flow (endpoint)

1. Request hits `GET /api/public/versions`.
2. Controller reads 25 keys from `ConfigurationService.getConfig(...)` — in-memory `HashMap` lookups, no DB query.
3. Returns a JSON object (shape in §4.2).
4. Response carries `Cache-Control: max-age=300, public, stale-while-revalidate=86400` matching the existing `REFERENCE_DATA_CACHE` constant.

## 4. The contract

### 4.1 Configuration keys to seed

22 translation namespaces (the full `TranslationNamespace` enum — engineer verifies at Phase 5 against the enum source):

```
translations.checksum.COMMON
translations.checksum.COMMON_SYSTEM
translations.checksum.ERRORS
translations.checksum.VALIDATION
translations.checksum.PAGING
translations.checksum.BUTTONS
translations.checksum.INPUT
translations.checksum.DIALOG
translations.checksum.HEADER
translations.checksum.FOOTER
translations.checksum.NAVIGATION
translations.checksum.INTRO
translations.checksum.EXTRA_PRODUCTS
translations.checksum.COOKIES
translations.checksum.MESSAGES_PAGE
translations.checksum.DASHBOARD_PAGES
translations.checksum.ADMIN_PAGES
translations.checksum.ABOUT_PAGE
translations.checksum.FREE_ZONE_PAGE
translations.checksum.PRICING_PAGE
translations.checksum.METADATA
translations.checksum.BACKEND_TRANSLATIONS
```

3 catalog base sites (from `BaseSite.code` — engineer verifies at Phase 5 against `baseSiteRepository.findAll()`):

```
catalog.checksum.rs
catalog.checksum.rsmoto
catalog.checksum.me
```

All 25 seeded in `data-configuration.sql` with initial value `''` (empty string). The seed's `ON CONFLICT (id) DO NOTHING` ensures the empty value is only applied on first boot.

### 4.2 Response shape

```json
{
  "translations": {
    "COMMON": "a3f2b1c8d4e7f091",
    "COMMON_SYSTEM": "8b41e2c3a9d7f124",
    "ERRORS": "...",
    "...": "..."
  },
  "catalog": {
    "rs": "c19e7a3b4f8d2105",
    "rsmoto": "...",
    "me": "..."
  }
}
```

22 string fields under `translations`, 3 under `catalog`. Each value is a 16-character lowercase hex string (truncated SHA-256). Field names match the namespace enum value (for translations) and the base-site code (for catalog).

If a checksum is empty (first request after fresh boot before checksum computation completed — should be impossible per §3 readiness ordering, but defensive), the field carries `""`. Mobile treats empty as "always re-fetch" or as a sentinel; mobile-side decision.

### 4.3 HTTP behavior

- Method: `GET`.
- Auth: none (public).
- Rate limit: none (matches `/api/public/config`, `/api/public/translations`).
- `Cache-Control`: `max-age=300, public, stale-while-revalidate=86400`.
- Content-Type: `application/json`.
- Status: always 200 unless the application is in `REFUSING_TRAFFIC` (in which case the Cloudflare router worker serves a maintenance page).
- Error contract Part 7: this endpoint cannot fail on validation (no inputs). A 500 here is a genuine server bug, never used for "checksums not ready."

### 4.4 Checksum computation — translations

For each namespace, fetch all rows: `Translation` entities where `namespace = <ns>`, across all languages.

Input string per row:
```
<key>|<value>
```

Where `<key>` is `translation_key` and `<value>` is `translation_value`. ID and language ID are deliberately excluded — mobile cares about displayed content, not row plumbing.

Combine all rows for the namespace into one input by:
1. Sorting rows by `(language_code, key)` lexicographically.
2. Joining with `\n` between rows.

Hash the combined input with SHA-256. Take the first 16 hex characters of the lowercase hex representation.

**Verification:** if a row's value changes, checksum changes. If a row is reseeded with the same content under a different ID, checksum does not change. If a key is added or removed, checksum changes. If two languages get the same key with different values, both contribute and ordering by `(language_code, key)` makes the input deterministic across boots.

### 4.5 Checksum computation — catalog

For each base site, walk the catalog state via `CatalogCategoryAssignment` rows for that base site.

Input shape per base site:
```
<category_key>|<labelKey>|<route>|<iconId>|<freeZone>|<position>
  <filter_key>|<displayOrder>|<basic>|<enabled>
    <option_labelKey>
    <option_labelKey>
    ...
    range|<rangeKey>|<rangePrefix>|<rangeSuffix>
      <range_option_longValue>
      ...
  <filter_key>|...
<category_key>|...
```

Categories sorted by `key` ascending. Filters within a category sorted by `filterKey` ascending. Filter options sorted by `labelKey` ascending. Range options sorted by numeric value ascending (the `FilterRange.options` field is `List<Long>`, not an entity collection). Use `\n` as row separator throughout (indentation in the example above is illustrative — actual input uses field separators, not whitespace indentation, so the engineer's serialization is unambiguous).

Engineer settles the exact serialization format at Phase 5 — the constraint is deterministic, content-derived, and stable across boots. The example above defines the fields; the precise delimiter scheme is engineer's call as long as it's documented in code.

Hash with SHA-256, truncate to 16 hex chars.

**Verification:** a new category assigned to `me` changes `catalog.checksum.me` but not `rs` or `rsmoto`. A new filter on a category shared across all three base sites changes all three. An option reordering changes the checksum only if the option label set itself changed (sort is by `labelKey`, not by stored order — so position changes don't matter at the option level). For category and filter ordering at the parent level, `position` and `displayOrder` are included in the input string, so reordering does change the checksum.

## 5. New components

### 5.1 VersionChecksumService

New file, package alongside `DefaultTranslationService` and `CatalogManager` (engineer picks the right package — likely `service/impl` or a dedicated `versioning` subpackage).

Owns:
- Checksum computation for translations and per-base-site catalog state.
- Boot-time conditional cache rebuild for versioned caches.
- Async rebuild of translation caches on admin edits.

Public surface:
- `void onAppReady()` annotated `@EventListener(ApplicationReadyEvent.class) @Order(3)`. Synchronous. For each versioned cache: compute checksum, compare to persisted, rebuild and replace if changed or Redis empty.
- `void rebuildTranslationCacheAsync(TranslationNamespace namespace, Language language)` annotated `@Async`. Called from `DefaultTranslationService.updateTranslation` after DB write and admin response.

Internal helpers:
- `String computeTranslationChecksum(TranslationNamespace namespace)` — DB read + hash per §4.4.
- `String computeCatalogChecksum(BaseSite baseSite)` — DB read + hash per §4.5.
- `void rebuildTranslationCache(TranslationNamespace namespace, Language language)` — synchronous Redis rebuild (used by both boot and async paths; async path wraps it).
- `void rebuildBaseSiteCache(BaseSite baseSite)` — synchronous Redis rebuild for one base site.
- `boolean redisKeyExists(String cacheName, String key)` — Redis-level existence check, used by the boot-time "Redis empty OR checksum changed" gate.
- `void persistChecksum(String configKey, String newChecksum)` — calls `updateConfiguration`.

`@Async` configuration: `@EnableAsync` on the application class or a dedicated configuration class if not already present. An `Executor` bean must be provided — if the codebase doesn't have one, this feature adds it. Engineer verifies in Phase 5. Default executor configuration is acceptable; the workload (one namespace × one language Redis rebuild) is light.

### 5.2 VersionController

New controller. Single endpoint.

```java
@RestController
@RequestMapping("/api/public/versions")
public class VersionController {

    private final ConfigurationService configurationService;

    @GetMapping
    public ResponseEntity<VersionsResponseDTO> getVersions() {
        // build response from configurationService.getConfig(...) calls
        return ResponseEntity.ok()
            .cacheControl(REFERENCE_DATA_CACHE)
            .body(response);
    }
}
```

Reuses the `REFERENCE_DATA_CACHE` constant — engineer extracts it to a shared location if not already there (`ConfigurationController`, `TranslationsController`, `BaseSiteController` all carry their own copy today; Part 4a calls for consolidating to one constant when the third caller appears, and we're now at the fourth).

### 5.3 VersionsResponseDTO

```java
public record VersionsResponseDTO(
    Map<String, String> translations,
    Map<String, String> catalog
) {}
```

Keys in the inner maps are namespace names (uppercase enum string) and base-site codes (lowercase string) respectively. Values are 16-char hex strings.

### 5.4 Per-base-site cache split in DefaultBaseSiteService

The existing `redisBaseSites` cache (single key holding `List<BaseSiteDTO>`) splits into per-base-site keys.

New cache definition in `RedisConfig`:
- `redisBaseSite` — value type `BaseSiteDTO`, no TTL, no runtime mutation expected outside `VersionChecksumService` rebuilds.
- The existing `redisBaseSites` cache definition is removed.

New method on `DefaultBaseSiteService`:
- `BaseSiteDTO getBaseSiteByCode(String code)` annotated `@Cacheable(value = "redisBaseSite", key = "#code", sync = true)`.

Existing `getAllBaseSites()` method stays for caller compatibility. Its implementation changes to:
- Read all known base-site codes from `baseSiteRepository.findAll()` (cheap, one query).
- For each code, call `getBaseSiteByCode(code)` — which goes through the per-key cache.
- Return the composed list.

This is Path 1 per the chat decision: the cache is split now, callers stay as-is. Three Redis reads per `getAllBaseSites` call instead of one. A future feature refactors callers to fetch the specific base site they need.

### 5.5 CacheWarmupService

Stays in place. Continues to warm `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. Removed from its scope: `redisBaseSites` (gone — replaced by per-base-site keys owned by `VersionChecksumService`). Order stays as-is or moves to `@Order(9)` to make the relative ordering with `VersionChecksumService.@Order(3)` explicit. Engineer's call.

## 6. Translation cache refactor (Redis unification)

Migrates `DefaultTranslationService` from raw `StringRedisTemplate` + gzip + Base64 + JVM-synchronized rebuild to Spring `@Cacheable` / `@CacheEvict` via `RedisCacheManager`. This is the long-standing tech-debt cleanup bundled into this feature per Phase 3 Path A.

### 6.1 New cache definition in RedisConfig

Add a new managed cache `redisTranslations`. Mirror the existing reference-data cache shape from the 2026-05-17 cache TTL decision: no TTL, no runtime mutation expected outside admin edits (which use `@CacheEvict`).

```java
addTypeConfig(configs, "redisTranslations", null, new TypeReference<Map<String, String>>() {});
```

Key shape: `redisTranslations::{NAMESPACE}:{langCode}` (Spring's default colon-separator pattern, no manual key construction).

Value shape: `Map<String, String>` (translation key → value), serialized via the existing `JacksonJsonRedisSerializer` configured for the cache manager. No gzip. No Base64.

Engineer verifies in Phase 5 that the existing `addTypeConfig` helper supports `Map<String, String>` parametrization; if it doesn't, the helper gains a one-line overload.

### 6.2 DefaultTranslationService changes

Replace the cache-miss + gzip + synchronized rebuild path with a `@Cacheable` annotation:

```java
@Cacheable(value = "redisTranslations", key = "#namespace.name() + ':' + #language.code()", sync = true)
public Map<String, String> getTranslationsForNamespace(TranslationNamespace namespace, Language language) {
    return loadFromDb(namespace, language);
}
```

`sync = true` provides Spring's per-key single-flight semantics — equivalent to the current `synchronized` lock on `(namespace + "_" + language).intern()` but using Spring's standard mechanism. This protects against the cold-Redis-key case (Redis crash, manual flush) where multiple concurrent requests would otherwise race to populate.

**No `@CacheEvict`.** Admin updates don't evict; they trigger an async rebuild via `VersionChecksumService.rebuildTranslationCacheAsync` (see §3 "Admin edit flow" and §5.1). The rebuild writes the new value over the old atomically — Redis's `SET` is atomic, replacing the previous value. End users never observe a missing key during the swap.

Concretely, the admin update path inside `DefaultTranslationService.updateTranslation` looks like:

```java
@Transactional
public void updateTranslation(...) {
    // existing: find language, find Translation entity, update value (JPA dirty-tracking persists)
    // existing: if BACKEND_TRANSLATIONS, rebuild in-memory map

    // new: trigger async cache rebuild
    versionChecksumService.rebuildTranslationCacheAsync(namespace, language);
}
```

The async method does the DB re-read, Redis write, and checksum persist. Admin response returns as soon as the transaction commits; the rebuild runs in the background.

### 6.3 Boot warmup for translations

The current `@PostConstruct initTranslations()` aggressively populates Redis for every namespace × language pair on every boot.

After this feature lands, `VersionChecksumService.onAppReady()` at `@Order(3)` handles the translation Redis cache. For each `(namespace, language)` pair:
- Compute checksum from DB.
- Compare to persisted checksum in `configuration`.
- Check Redis key existence.
- If checksum changed OR Redis key missing: rebuild Redis (synchronous, build-then-replace), persist new checksum.
- Else: skip — Redis is already correct from the previous boot.

The `@PostConstruct` is removed from `DefaultTranslationService`. The `indexBackendTranslations()` method (the in-memory `backendTranslations` map) needs a new home — engineer picks between:
- A method called from `VersionChecksumService.onAppReady()` after translation rebuild.
- A separate `@EventListener(ApplicationReadyEvent.class)` with appropriate `@Order`.
- Keeping `@PostConstruct` on `DefaultTranslationService` (the in-memory map doesn't depend on Redis or `ConfigurationService`, so it can populate during context refresh).

Engineer's call in Phase 5. The `BACKEND_TRANSLATIONS` namespace's admin-edit path stays: `updateTranslation` continues to call `indexBackendTranslations()` synchronously inside the transaction (not async — the in-memory map must be coherent with the DB write).

### 6.4 Removed

- The `cache.translations.redis` config toggle. Always-on cache, no toggle needed. Remove the seed row from `data-configuration.sql` and the read site in `DefaultTranslationService.getTranslationsForNamespace`. Per Part 4a, configuration is for values that vary across environments; this value has one setting (`true`) and no foreseeable second setting.
- `GZipService` calls from `DefaultTranslationService`. If `GZipService` has no other callers in the codebase, delete the file. Engineer verifies in Phase 5 via grep.
- Raw `StringRedisTemplate` field in `DefaultTranslationService`. If no other callers in the file, remove the bean injection.
- The `synchronized (lockKey.intern())` block. Spring's `@Cacheable(sync = true)` replaces it.
- `getOrRebuild`, `redisKey`, the manual JSON ↔ gzip ↔ Base64 encoding helpers. Dead with the new cache manager doing serialization.
- The dead `translations.version` counter — `updateTranslationsVersion()` method, the call from `updateTranslation`, the seed row in `data-configuration.sql`. Engineer verifies in Phase 5 that nothing else consumes `translations.version` (audit confirmed: nothing did).

## 7. Trust boundaries (Part 11)

The endpoint exposes derived state from the server's own database. No client input. No moderation, authorization, or state-transition decisions depend on any client-supplied value.

Audit §5 verified: checksums reveal "translations changed" or "catalog for base site X changed" — they don't reveal the content. The translation/catalog content itself is already public via existing endpoints. No new information leak.

The `VersionChecksumService` reads from DB and writes to `ConfigurationService`. Both are server-internal. No trust boundary crossings.

## 8. SQL migrations

Per conventions Part 12 (pre-production schema fold), changes to `data-configuration.sql` edit the file in place:
- Add 22 + 3 = 25 new rows for checksum keys, all with empty initial value, `ON CONFLICT (id) DO NOTHING`.
- Remove the `cache.translations.redis` row.
- Remove the `translations.version` row.

No new V2 migration. Schema columns are not touched. (`Catalog.currentVersion` and `Category.fileVersion` stay for now — see "Out" in §1.)

`RedisCacheEvictConfig` changes are not SQL but are part of the schema-level cache topology:
- Remove `redisBaseSites` from the evict-on-startup list (cache no longer exists).
- Add `redisBaseSite` to the do-NOT-evict list. Engineer verifies the eviction config's shape — if it's an allowlist of caches to clear, `redisBaseSite` is simply not added. If it's a denylist of caches to skip, `redisBaseSite` is added explicitly.
- Add `redisTranslations` to the do-NOT-evict list with the same shape.

## 9. Tests

### 9.1 New tests

- `VersionChecksumServiceTest` — unit tests for `computeTranslationChecksum` (deterministic output for fixed input, value change produces different output, key-only changes produce different output, language-rows in the namespace contribute to one hash, sorting is stable).
- `VersionChecksumServiceTest` — unit tests for `computeCatalogChecksum` (per-base-site isolation, new filter on a category changes the parent checksum, option labelKey changes change the checksum).
- `VersionChecksumServiceTest` — boot-flow integration tests: (a) checksum matches persisted, Redis populated → no rebuild; (b) checksum matches persisted, Redis empty → rebuild; (c) checksum changed → rebuild and persist new checksum.
- `VersionChecksumServiceTest` — async rebuild test: `rebuildTranslationCacheAsync` updates Redis value atomically (no observable empty-key window from a concurrent reader) and persists the new checksum.
- `VersionControllerTest` — integration test verifying response shape, cache header, status 200, all 25 keys present.
- Boot-listener integration test verifying `VersionChecksumService.onAppReady` runs at `@Order(3)`, after `@Order(1)` and `@Order(2)`, before `@Order(10)`.

### 9.2 Existing tests to update

- `DefaultTranslationServiceTest` (or equivalent) — update for the `@Cacheable` refactor. The synchronized-rebuild test path is gone. The admin-edit-triggers-async-rebuild test path replaces the previous evict-and-rebuild assertions.
- `DefaultBaseSiteServiceTest` (or equivalent) — update for the per-base-site cache split. Existing `getAllBaseSites` test stays (verifies composition still returns the list); add a test for the new `getBaseSiteByCode` method and its `@Cacheable` shape.
- `ConfigurationSeedTest` (if it iterates `Configuration.values` exhaustively) — add the new 25 keys, remove the two retired keys.
- `CacheWarmupServiceTest` (if exists) — update to reflect the removal of `redisBaseSites` warmup. `redisBaseSite` is not in `CacheWarmupService`'s scope.

### 9.3 Verification commands

Per conventions Part 4:
- `./mvnw spotless:check`
- `./mvnw test` — full suite green. Expected count: current baseline (502 at last check) plus ~15-20 new tests.

## 10. Definition of done

Code:
- `VersionChecksumService` exists, runs at `@Order(3)` synchronously on boot, owns rebuild lifecycle for `redisTranslations` and `redisBaseSite::*` caches.
- Boot-time rebuild logic: for each versioned cache, rebuild only if Redis key empty OR checksum changed. Otherwise skip.
- Boot-time rebuilds use soft replacement (build new value, atomic Redis `SET` over the old key).
- `@Async` infrastructure in place (`@EnableAsync` plus an `Executor` bean if not already present).
- `DefaultTranslationService.updateTranslation` triggers `versionChecksumService.rebuildTranslationCacheAsync` after the DB write. Admin response returns immediately.
- Async rebuild atomically replaces Redis value and persists new checksum to `configuration`.
- `VersionController` exposes `GET /api/public/versions` with the locked response shape and cache header.
- `REFERENCE_DATA_CACHE` constant extracted to a shared location (Part 4a).
- `DefaultTranslationService` cache layer migrated to `@Cacheable` with `sync = true`. No `@CacheEvict` for versioned caches.
- `redisBaseSites` cache replaced by per-base-site `redisBaseSite` cache. `DefaultBaseSiteService.getBaseSiteByCode(code)` is the new `@Cacheable` entry point. `getAllBaseSites` becomes a composition.
- `CacheWarmupService` no longer warms `redisBaseSites`. Continues to warm `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`.
- `RedisCacheEvictConfig` updated: versioned caches (`redisTranslations`, `redisBaseSite`) are NOT cleared on boot.
- `data-configuration.sql` seeds the 25 new keys, removes the 2 retired keys.
- `cache.translations.redis` toggle removed end-to-end.
- `translations.version` counter removed end-to-end.
- `GZipService` removed if no other callers.

Verification:
- `./mvnw spotless:check` clean.
- `./mvnw test` green, all new tests passing.
- Manual smoke 1: hit `/api/public/versions` against a local stack, verify response shape and 25 fields.
- Manual smoke 2: mutate a translation via admin endpoint, verify admin response returns fast (<200ms) and the affected namespace's checksum changes in the next `/api/public/versions` response after async rebuild completes (typically <1s). During the window, verify the old translation value is still served (no empty cache).
- Manual smoke 3: change a value in a translation SQL seed file, restart the app, verify the affected namespace's checksum changes, Redis is rebuilt, mobile would see the new value.
- Manual smoke 4: restart the app with no data changes, verify `VersionChecksumService` skips rebuild (log line or test instrumentation confirms skip path).

Docs:
- This spec (the amendments in this brief).
- `decisions.md` closing entry on feature shipping.
- `state.md` row added to active features, then flipped to `shipped` at close.
- `issues.md` entries for any adjacent observations Phase 5 surfaces that are out of scope.

## 11. Phase 5 brief order (planned)

The engineer agent in `oglasino-backend` runs these in sequence. Each closes before the next opens.

1. **Brief 1 — `VersionChecksumService` scaffold + seed config keys.** Create the service shell, the `@EventListener(ApplicationReadyEvent.class) @Order(3)` entry point, both `computeXChecksum` methods, the `persistChecksum` helper, the `redisKeyExists` helper. Edit `data-configuration.sql` to add 25 new rows and remove the 2 retired rows. Engineer verifies the namespace list against `TranslationNamespace.values()` and the base-site list against `baseSiteRepository.findAll()`. Unit tests for checksum computation. Does not wire any Redis rebuild yet.

2. **Brief 2 — Per-base-site cache split.** Add `redisBaseSite` cache to `RedisConfig`. Add `getBaseSiteByCode` method on `DefaultBaseSiteService` with `@Cacheable(value = "redisBaseSite", key = "#code", sync = true)`. Rewrite `getAllBaseSites` as composition. Remove `redisBaseSites` cache definition. Update `CacheWarmupService` to drop `redisBaseSites` warmup. Update `RedisCacheEvictConfig` so `redisBaseSite` is not evicted on boot. Update `DefaultBaseSiteServiceTest`. Existing call sites untouched (Path 1 per chat decision).

3. **Brief 3 — Wire base-site cache rebuild into `VersionChecksumService`.** Extend `onAppReady` to iterate base sites: compute checksum, compare to persisted, check Redis key existence, soft-rebuild and persist if needed. Integration tests for the boot-time logic.

4. **Brief 4 — Translation cache refactor.** Migrate `DefaultTranslationService.getTranslationsForNamespace` to `@Cacheable(value = "redisTranslations", key = "...", sync = true)`. Remove `@PostConstruct initTranslations()`. Decide where `indexBackendTranslations` lives. Remove `GZipService` usage, `cache.translations.redis` config read, the synchronized rebuild lock, `getOrRebuild`, `redisKey`, JSON ↔ gzip ↔ Base64 helpers, raw `StringRedisTemplate` field. Remove `updateTranslationsVersion`. Add `redisTranslations` cache to `RedisConfig`. Update `RedisCacheEvictConfig`.

5. **Brief 5 — Wire translation cache rebuild into `VersionChecksumService` (boot path).** Extend `onAppReady` to iterate translation namespaces × languages: compute checksum, compare to persisted, check Redis key existence, soft-rebuild and persist if needed. Integration tests for the boot-time translation path.

6. **Brief 6 — Async admin path + `VersionController`.** Add `@EnableAsync` and `Executor` bean if needed. Add `rebuildTranslationCacheAsync` method to `VersionChecksumService` annotated `@Async`. Wire `DefaultTranslationService.updateTranslation` to call it. Create `VersionController`, `VersionsResponseDTO`. Extract `REFERENCE_DATA_CACHE` constant. Integration tests for the endpoint and the async rebuild path.

7. **Brief 7 — Remove `/baseSite/details` usage from `oglasino-web`** (scope expansion from chat). Delete dead `getAllBaseSites()` server action. Migrate `sitemap.ts` from `/public/baseSite/details` to `/public/baseSite/overviews`. Backend endpoint stays in place (mobile may use it).

8. **Brief 8 — Admin translation UX upgrade** (`oglasino-web`). Per §12 of this spec. Lazy load on `/admin/translations`. Mirror cache page UI/UX. Relocate refresh button to its proper admin home. Pre-fill edit dialog new-value. No-op early return. Toasts on save. Result count display. Empty-state helper message. Engineer audits the admin surface to identify the cache page and the refresh button's new home.

9. **Brief 9 — Seed admin translation keys** (`oglasino-backend`). 13 keys × 4 locales = 52 rows in `ADMIN_PAGES` namespace covering the translation keys referenced by Brief 8's admin UX work. EN final; RS / RU / CNR Mastermind-drafted placeholders queued for native-translator review at feature close.

10. **Brief 10 — Unified per-cache delete + warmup endpoints** (`oglasino-backend`, scope expansion from chat). Adds `AdminCacheDescriptor` enum listing all 8 admin-managed caches (6 with warmup support, 2 user caches without). New endpoints: `GET /api/secure/admin/cache/list`, `POST /api/secure/admin/cache/{cacheName}/clear`, `POST /api/secure/admin/cache/{cacheName}/warmup`. `CacheWarmupService.warmup()` split into four per-cache methods. `VersionChecksumService.rebuildBaseSiteCachesIfNeeded` and `rebuildTranslationCachesIfNeeded` widened to public. Old bulk endpoints (`/evict`, `/warmup`, `/refresh`) updated to cover all 8 caches; old per-cache evict endpoint retained for web backwards compatibility.

11. **Brief 11 — Unify admin cache page** (`oglasino-web`, scope expansion from chat). Consumes Brief 10's endpoints. Single unified list of all 8 caches sourced from `GET /api/secure/admin/cache/list`. Per-row buttons: "Obriši" for all, additional "Obriši + Zagrej" for the 6 warmup-supported caches (orchestrated client-side: clear, then warmup). Standalone "Keš prevoda" section deleted; `redisTranslations` joins the unified list. `RefreshTranslationCacheButton` file deleted.

12. **Brief 12 — Seed admin cache page translation keys** (`oglasino-backend`). 3 keys × 4 locales = 12 rows in `ADMIN_PAGES` namespace for Brief 11's button labels and toast messages. EN final; RS / RU / CNR Mastermind-drafted placeholders queued for native-translator review at feature close. RU drafts include `''` (soft-sign) convention per Brief 9 learning.

13. **Brief 13 — Delete old admin cache endpoints** (`oglasino-backend`, closing cleanup). 5 old endpoints (`GET /admin/cache`, `POST /admin/cache/evict/{name}`, `POST /admin/cache/evict`, `POST /admin/cache/warmup`, `POST /admin/cache/refresh`) deleted from `CacheAdminController` after Brief 11 migrated web to the new endpoints. 112 lines removed total.

Mobile adoption is not in scope; queues into the Expo backlog as `oglasino-expo-version-checksums-<n>` per the 2026-05-17 slug discipline.

## 12. Admin translation UX upgrade

A scope expansion landed during the feature's Phase 5 to upgrade the admin translation surface in `oglasino-web`. Driven by audit findings (`oglasino-web/.agent/audit-version-checksums-web.md` Section 1) that identified seven concrete pain points on the current `/admin/translations` page.

### 12.1 Scope

In:
- Lazy load — page open shows an empty state, not a 21-namespace fetch.
- Mirror the cache page UI/UX pattern (engineer identifies the cache page during the brief and matches its component layout).
- Remove the refresh button from the translations page.
- Add a refresh-translations-cache button to whatever admin page is the established home for "refresh"-style controls (engineer audits to identify; if no obvious home exists, flags in session).
- Pre-fill the edit dialog's new-value textarea with the current value (no more retyping for one-character edits).
- Toast on save success and on save failure.
- No-op early return in `AdminConfigChangeDialog.onApplyInternal` — `return` after `onClose()` on the unchanged-value path.
- Result count display above the translation table after fetch.
- Empty state when no namespace selected and no filter typed: helper message ("Select a namespace or type to search" or similar — engineer matches existing helper-text patterns).

Out:
- Version/checksum visibility on the admin surface. The `/api/public/versions` endpoint serves mobile; admin doesn't need it.
- Pagination. With lazy load + namespace scoping, result sets are already bounded. Revisit if a single namespace ever exceeds ~1000 rows.
- Debounced filters. Apply-button-driven filters stay as-is.
- Column sort. Out of scope.
- Multi-language edit view (edit one key across all languages in one form). Would require a new UI shape and possibly a bulk-update backend endpoint. Separate feature if pursued.
- Backend changes. This is a `oglasino-web`-only brief.

### 12.2 Design

**Lazy load.** The current `TranslationsTable` fetches on mount via `useEffect`. Change: do not fetch until either (a) a namespace is selected via the filter, or (b) at least one of the text filters (key, value) is non-empty. When neither condition is met, render the helper-message empty state instead of an empty table.

The trigger: the existing `params` reactivity is fine — `TranslationsTable` already re-fetches when `params` change. The change is gating the fetch on having useful filter content. If `params.namespace === undefined && !params.key && !params.value`, skip the fetch and render the helper message.

The current "21 parallel API calls when no namespace is selected" behavior in `getFilteredTranslations` remains reachable post-gate. When namespace is empty but key OR value has content, `hasFilters` is true, the fetch fires, and the service legitimately iterates all 21 namespaces — that's the "find this key across all namespaces" search path. The 21-call loop is kept as the legitimate cross-namespace search implementation.

**Refresh button relocation.** The current `RefreshCacheButton` on the translations page calls `refreshTranslationsCache()` (which hits `GET /secure/admin/translations/refresh-cache`). The button moves to whichever admin page is the established home for "refresh"-style operations — the cache page (if such a page exists), the admin configuration page, or a generic admin tools page. Engineer audits the admin surface to find the right home. If multiple candidates exist, engineer picks based on conceptual fit (where else are caches managed?). If no clear home exists, engineer flags and we pick together.

When relocated, the button needs feedback: spinner during fetch, toast on success and failure. Toast pattern follows whatever the codebase already uses (likely `react-hot-toast` or similar — engineer matches).

**Pre-fill new-value textarea.** Current `AdminConfigChangeDialog` initializes `newValue` as `''`. Change: initialize as `config.value`. Tiny change; large UX impact for any edit shorter than a full rewrite.

**No-op early return.** Current `onApplyInternal` calls `onClose()` when `config.value === newValue` but falls through to `onApply(newValue)`. Add `return` after `onClose()` on the unchanged-value branch.

**Toast on save.** When `updateTranslation()` resolves successfully, fire a success toast (e.g., "Translation updated"). When it rejects or returns failure, fire an error toast (replacing the current inline generic error, or in addition to it — engineer's call on whether to keep the inline error as well).

**Result count.** After fetch resolves, display the count of translations returned above the table. Something like "23 translations" or "23 results." Static text, no interactivity needed.

**Empty state helper.** When the lazy-load gate is active (no namespace, no filter), render a helper message in the table's body area: "Select a namespace or type to search." Plain text in the existing empty-state styling pattern.

### 12.3 Mirror the cache page

Engineer identifies the admin cache page (e.g., `/admin/cache` or similar) during the brief, reviews its component layout, and mirrors it where reasonable for the translations page. The goal is visual and interaction consistency across admin surfaces, not a forced rewrite — if the current translations page already shares the dominant admin layout, the "mirror" is mostly verification.

If the cache page uses specific component patterns (filter panel shape, table header style, action button placement) that the translations page doesn't, align the translations page to those patterns.

### 12.4 Tests

Update `TranslationsTable` tests for the lazy-load gate:
- `does not fetch when no namespace and no filters`
- `fetches when namespace is selected`
- `fetches when key filter is non-empty`
- `fetches when value filter is non-empty`
- `shows helper message when no namespace and no filters`
- `shows result count after fetch`

Update `AdminConfigChangeDialog` tests:
- `new value textarea initializes with current value`
- `no-op edit does not call onApply`
- `success toast fires on save success`
- `error toast fires on save failure`

Update tests for the relocated refresh button at its new home, mirroring whatever test pattern lives there.

Out-of-scope test changes (removal): tests that asserted the 21-parallel-call behavior of `getFilteredTranslations` are removed or rewritten as the unreachable path is removed.

### 12.5 Definition of done

Code:
- `TranslationsTable` does not fetch on mount without a namespace or filter content.
- Helper message renders in the empty state.
- Result count renders above the table after fetch.
- `RefreshCacheButton` removed from `/admin/translations`.
- A refresh-translations-cache button exists at its new admin home, with spinner + toast feedback.
- `AdminConfigChangeDialog`'s new-value textarea pre-fills with current value.
- `AdminConfigChangeDialog` no-op edit returns early.
- Save success and save failure both fire toasts in the dialog flow.
- Translations page visually and interactionally consistent with the cache page.

Verification:
- `npm run lint` clean.
- `npx tsc --noEmit` clean.
- `npm test` green.
- Manual smoke: open `/admin/translations`, verify empty state on first load, verify fetch fires when namespace selected or filter typed, verify edit dialog pre-fills, verify toast on save, verify refresh button is gone from translations page and present at its new home with working feedback.

### 12.6 Out

- Version/checksum visibility on admin.
- Pagination, debounced filters, column sort, multi-language view (each could be a future feature).
- Backend changes.
