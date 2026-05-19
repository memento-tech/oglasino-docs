# Session summary

**Repo:** oglasino-backend
**Branch:** dev (see "For Mastermind" — brief asked for `feature/connection-pool-hardening`, engineer stayed on `dev` per hard rule)
**Date:** 2026-05-17
**Task:** Investigate whether `redisBaseSites` needs a TTL.

## Implemented

- Read-only investigation. No code touched. Findings recorded below.

## Files touched

- (none)

## Tests

- Not run. Read-only session, no code modified.

---

## TTL configuration found

- **Where:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:50`
  ```java
  addListTypeConfig(
      BaseSiteDTO.class, "redisBaseSites", 1440, defaultConfig, cacheConfigs, objectMapper);
  ```
  The `1440` is the TTL in minutes, passed to `addListTypeConfig` which wraps it in `Duration.ofMinutes(ttl)` (line 105).
- **Current value:** 1440 minutes = **24 hours**.
- **Documentation explaining why:** none in code. No inline comment on the line, no Javadoc on `addListTypeConfig`, no commit message about this specific value. Contrast with `redisUserAuth` (line 41-46) which carries a deliberate four-line comment explaining the TTL is a backstop and that the real freshness mechanism is `@CacheEvict` on `DefaultUserService.saveUser`. `redisBaseSites` has no such note.

Adjacent: `redisBaseSiteOverviews` (line 51-57) is configured at **24 minutes** — three orders of magnitude shorter than `redisBaseSites` (24 hours). Flagged in "For Mastermind"; out of scope per brief.

## Mutation paths inventoried

I searched the entire `src/main` tree for:

- `baseSiteRepository.save` / `.delete` / `.saveAll` / `.deleteAll`
- save/delete on every related entity repo (`languageRepository`, `currencyRepository`, `catalogRepository`, `regionRepository`, `cityRepository`)
- `@Modifying` JPQL or `nativeQuery = true`
- `@CacheEvict` / `@CachePut` targeting `redisBaseSites` (zero hits)
- admin controllers / facade endpoints touching base-site reference data

Total mutation paths to `BaseSite` (or any entity reachable through the cached `BaseSiteDTO` graph: `Language`, `Currency`, `Catalog`, `Region`) in non-test code:

1. **`CatalogManager.initBaseSiteCatalog` — `baseSiteRepository.save(baseSite)`** at `src/main/java/com/memento/tech/oglasino/catalog/service/CatalogManager.java:101`. Sets `baseSite.setCatalog(catalog)` on a fresh base site. Boot-time only — invoked from `CatalogManager.onAppReady` (`@EventListener(ApplicationReadyEvent.class)`, `@Order(2)`, `@Transactional`).

2. **`CatalogManager.initBaseSiteCatalog` — `catalogRepository.save(catalog)`** at the same file, lines 99 and 107. Creates the `Catalog` row and bumps `currentVersion` / sets `topFilters`. Same boot-time `onAppReady` entry point. `BaseSiteDTO.catalog` (a `CatalogDTO` mapped via ModelMapper from the `Catalog` entity) reflects this entity's `topFilters` — the rest of `CatalogDTO`'s fields are unpopulated on this path (see adjacent observation).

Both of those carry **no `@CacheEvict`** and route through **no method that does**.

3. **No other mutation paths exist.** No admin/controller/facade endpoint writes to `BaseSite`, `Catalog`, `Language`, `Currency`, or `Region`. No `@Modifying` queries hit these tables. The repositories are exclusively read paths outside `CatalogManager` (e.g. `CatalogToJsonService.generateBaseSiteJSON` is `findAll`/`findById`, `DefaultUserFacade` calls `findBaseSiteIdsByRegionId`, etc).

Why the missing `@CacheEvict` is not a stale-data risk today:

- The boot sequence is `evict → CatalogManager.onAppReady (writes) → warmup → readiness flip to ACCEPTING_TRAFFIC`. Explicit at `CacheWarmupService.warmup` (file `src/main/java/com/memento/tech/oglasino/config/CacheWarmupService.java`, ordering documented in the Javadoc at lines 25-33 and in the `finally` flip at line 118).
- `RedisCacheEvictConfig.evictCachesOnStartup` (`src/main/java/com/memento/tech/oglasino/config/RedisCacheEvictConfig.java:39-53`) clears every managed cache including `redisBaseSites` at startup; `CacheWarmupService.warmup` re-populates it before traffic is admitted (line 71). So any boot-time mutations land before the cache is repopulated.
- Manual operator mutations (raw `psql UPDATE base_site …`) are expected to be paired with the admin evict endpoint exposed by `CacheAdminController` (`src/main/java/com/memento/tech/oglasino/admin/controller/CacheAdminController.java`, `POST /api/secure/admin/cache/evict[/{cacheName}]` and `/refresh`). The `BaseSiteController` Javadoc at file `src/main/java/com/memento/tech/oglasino/controller/BaseSiteController.java:26` explicitly documents this contract: *"Admin-driven changes evict via the cache admin controller, so 5-min staleness is acceptable."*

In short: the BaseSite cache has **no runtime write path at all**. Its contents are effectively immutable between deploys, with operator-triggered eviction as the documented escape hatch.

## Comparison to `redisTranslations`

Same intent, different mechanism, slightly different shape.

- **`redisTranslations` (precedent):** does not flow through Spring's `@Cacheable`. `DefaultTranslationService` writes raw entries via `RedisTemplate` keyed by `translations:{namespace}:{langCode}` (see `redisKey` at `DefaultTranslationService.java:51`). No TTL is configured on these keys (raw `SET`). Eviction is **explicit re-indexing** on every write: `updateTranslation` (`DefaultTranslationService.java:207-222`) mutates the row inside `@Transactional` and immediately calls `indexNamespaceLang(namespace, language)` to overwrite the affected Redis entry. Per `decisions.md` 2026-05-14 ("Redis caching & filter control-flow fixes"), this was confirmed to be a deliberate, correct pattern.

- **`redisBaseSites`:** uses Spring `@Cacheable` (`DefaultBaseSiteService.java:58`) with a 24h TTL on the cache. No `@CacheEvict` anywhere — but there is no runtime write path that needs one. Eviction-on-write is, in effect, satisfied because **there is no write at runtime**. The boot path runs before warmup repopulates; the admin path goes through `CacheAdminController.evictAll/evictOne/refresh`.

Does `BaseSite` match the translations pattern (no TTL, evict-on-every-write)? **Functionally yes, mechanically different.** Translations has runtime writes and pairs each with explicit eviction. BaseSite has no runtime writes — the equivalent guarantee is "no writes at all between boots, except operator-driven ones which go through the admin evict endpoint." The TTL on `redisBaseSites` is a backstop for the operator-forgot-to-evict case, equivalent to the role the comment on `redisUserAuth` at `RedisConfig.java:43-46` describes for its TTL.

## Verdict

**TTL not needed for correctness.** All semantically-meaningful mutation paths are covered:

- Boot-time `CatalogManager.save` writes land while the cache is empty (post-evict) and are followed by warmup before traffic admittance.
- Admin/operator writes flow through `CacheAdminController` which evicts explicitly.

The 24h TTL is a pure backstop against an unintentional out-of-band DB mutation (e.g. someone running raw SQL and forgetting to call `/api/secure/admin/cache/evict/redisBaseSites`). At 24h that's a weak backstop — the operator either notices and evicts within minutes, or doesn't notice and lives with the stale read for up to 24h. Removing the TTL replaces "stale for up to 24h" with "stale until next deploy or admin evict" — the same trade the team accepted for `redisTranslations`.

The connection-pool-incident angle reinforces this: per `decisions.md` 2026-05-14 and `features/connection-pool-hardening.md` item 3, the 24h TTL is itself a **known trigger** for the thundering-herd that the bug-fixer chat scoped out. Removing the TTL eliminates the daily-TTL-expiry trigger entirely (the post-deploy trigger was already closed by the warmup-gating fix). The structural single-flight / `@Transactional` fixes in the feature still need to land to make the *cold-cache* path safe at any TTL, but removing the TTL closes off the most-likely current trigger today.

Small follow-up implied by removal:

- Add a comment near the `addListTypeConfig` call for `redisBaseSites` (and over `addListTypeConfig` itself, since the second arg's role becomes ambiguous if some configs pass `Duration` and others pass minutes — see "For Mastermind" alternative-API note) mirroring the `redisUserAuth` style: explain that the cache is populated at warmup, has no runtime write path, and admin-driven manual evictions go through `CacheAdminController`.

## Recommendation

**Draft a small follow-up brief to remove the `redisBaseSites` TTL.** Mechanical scope is ~one line of edit at `RedisConfig.java:50`: convert the `addListTypeConfig` call for `redisBaseSites` so the produced `RedisCacheConfiguration` does not have an `entryTtl` applied (this likely requires either a sibling helper without `entryTtl(...)`, or a sentinel value such as `0` interpreted as "no expiry," or restructuring the helper to take a nullable `Duration`). Plus a four-line comment mirroring the `redisUserAuth` style at line 41-46.

That follow-up brief can fold into the `connection-pool-hardening` feature work or run as its own thin brief in this bug chat. Either way, the structural single-flight + remove-`@Transactional` fixes from the feature spec are still needed — removing the TTL closes the daily trigger but does not make the cold-cache path itself safe under concurrent misses (e.g. on first deploy after this change, or any future restart). The two changes are complementary.

## Cleanup performed

- none needed (read-only session, no code touched)

## Obsoleted by this session

- nothing

## Known gaps / TODOs

- none

## Conventions check

- Part 4 (cleanliness): N/A this session — nothing written, nothing to clean.
- Part 4a (simplicity): N/A this session — no code changes.
- Part 4b (adjacent observations): three flagged in "For Mastermind" below.
- Part 6 (translations): N/A this session — no translation keys touched.
- Part 7 (error contract): N/A this session.
- Part 11 (trust boundaries): N/A this session — no DTOs touched.

## For Mastermind

### Process / branch

- **Branch mismatch.** Brief asks for `feature/connection-pool-hardening` (create if missing, from main). Engineer stayed on `dev` per hard rule "No `git checkout` to a different branch." Read-only session so the branch is irrelevant for the work product, but flagging for awareness in case Mastermind wants the follow-up implementation brief to handle the branch creation explicitly. Working tree also has two pre-existing modifications on `dev` not made by this session: `pom.xml` and `OglasinoApplication.java`. Did not touch either.

### Adjacent observations (out of scope for this brief)

1. **`redisBaseSiteOverviews` TTL is 24 minutes — likely a unit-mix bug.**
   - **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:51-57`
   - **Detail:** Every other entry in the file uses minutes-as-an-int with values that read as hour-or-day intent: `redisUserInfo` 30, `redisUserAuth` 30, `redisBaseSites` 1440 (24h), `redisLanguage` 1440, `redisLanguages` 1440, `redisBaseCurrency` 1440. `redisBaseSiteOverviews` is the outlier at `24`. Reads like a transcription mistake — someone wrote `24` thinking hours when the helper consumes minutes. The overviews cache backs the same `BaseSiteFilter`-adjacent reference-data hot path; a 24-min TTL means the daily-TTL-expiry cold-cache trigger discussed in `features/connection-pool-hardening.md` item 3 actually expires on this cache **every 24 minutes**, not every 24 hours, and any concurrent burst at expiry can drain the pool on the overviews path the same way the brief's incident describes for the sites path. If the verdict for `redisBaseSites` is "no TTL needed," the same logic almost certainly applies to `redisBaseSiteOverviews` — and either way the 24-vs-1440 asymmetry deserves a deliberate decision, not a transcription accident.
   - **Severity guess:** medium (latent connection-pool exhaustion trigger 60× more frequent than `redisBaseSites`'s; mechanism is the same one that took out prod on 2026-05-14 and 2026-05-16).
   - **Out of scope:** brief is explicit that TTL changes to other caches are out of scope.

2. **`addListTypeConfig` second arg is `long ttl` (minutes) with no type-level safety.**
   - **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:73-108`
   - **Detail:** Both `addTypeConfig` and `addListTypeConfig` take `long ttl` then call `Duration.ofMinutes(ttl)`. The unit is implicit at every call site. Renaming to `long ttlMinutes` or taking a `Duration` directly would have prevented the suspected `redisBaseSiteOverviews` mistake above. If the follow-up brief introduces a "no expiry" sentinel for `redisBaseSites`, this is the natural moment to switch to `Duration`-or-`null` and drop the implicit unit.
   - **Severity guess:** low (cosmetic / footgun).
   - **Out of scope:** out of this brief; would belong to the implementation brief that removes the TTL.

3. **`CatalogDTO.categories` and `CatalogDTO.orderTypes` are unmapped on the BaseSite cache path.**
   - **Files:** `src/main/java/com/memento/tech/oglasino/dto/CatalogDTO.java` and `src/main/java/com/memento/tech/oglasino/entity/Catalog.java`
   - **Detail:** `CatalogDTO` declares `categories: List<CategoryDTO>` and `orderTypes: List<OrderTypeDTO>`, but the `Catalog` entity has no matching fields (only `name`, `baseSite`, `topFilters`, `currentVersion`). The `modelMapper.map(cat, CatalogDTO.class)` call at `DefaultBaseSiteService.java:106` therefore leaves `categories` and `orderTypes` null on every `BaseSiteDTO.catalog`. Either fields are populated elsewhere by a different code path (separate catalog endpoint hydrates them on demand) — fine — or they're stale leftover fields. Worth a five-minute look during the connection-pool-hardening feature, since `getAllBaseSites` returning a `CatalogDTO` with two perpetually-null collections is a footgun for a future engineer reading the DTO.
   - **Severity guess:** low (cosmetic; if a frontend ever started trusting these fields off the BaseSite payload it would surprise; today's clients don't).
   - **Out of scope:** unrelated to TTL question.

### Verdict-relevant judgment

The verdict above (TTL not needed) is one of the three the brief named, so no fourth-option escalation. If Mastermind wants me to take this to "TTL needed because path X doesn't evict," I'd want to hear which path I missed — my read of the source is that no runtime write path exists, but I trust the planning side to flag if I've overlooked an admin tool or a job that lives outside this repo.
