# Session summary

**Repo:** oglasino-backend
**Branch:** dev (engineer stayed on the branch Igor had checked out per hard rule; brief noted "likely `dev` or `feature/connection-pool-hardening`" and accepted either)
**Date:** 2026-05-17
**Task:** Investigate TTL necessity for the six remaining Redis caches plus the translations cache.

## Implemented

- Read-only investigation. No code touched. Per-cache findings recorded below.

## Files touched

- (none)

## Tests

- Not run. Read-only session, no code modified.

---

## 1. `redisBaseSiteOverviews`

### TTL configuration found

- **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:51-57`
  ```java
  addListTypeConfig(
      BaseSiteOverviewDTO.class,
      "redisBaseSiteOverviews",
      24,
      defaultConfig,
      cacheConfigs,
      objectMapper);
  ```
- **Value:** 24 **minutes**. Confirmed by `addListTypeConfig` wrapping in `Duration.ofMinutes(ttl)` at line 105.
- **Documenting comment:** none.

### Mutation paths inventoried

Cache backs `DefaultBaseSiteService.getAllBaseSiteOverviews()` (`DefaultBaseSiteService.java:30-55`). The cached `BaseSiteOverviewDTO` carries: `id`, `domain`, `code`, `iconId`, `labelKey`, `flagImageKey`, `defaultLanguage` (LanguageDTO), `allowedLanguages` (List<LanguageDTO>). So mutations to `BaseSite`, `Language`, or `Language`-via-FK on `BaseSite` would affect this cache.

- **`BaseSite` mutations:** identical inventory to `redisBaseSites` (see prior session `2026-05-17-oglasino-backend-base-site-ttl-investigation-1.md`). Only path is `CatalogManager.initBaseSiteCatalog` → `baseSiteRepository.save(baseSite)` at `CatalogManager.java:101`, boot-only via `@EventListener(ApplicationReadyEvent.class) @Order(2)`.
- **`Language` mutations:** **zero runtime paths.** Confirmed by `grep -rn "languageRepository\." ... | grep -E "save|delete|@Modifying"` returning no hits. `LanguageRepository` is read-only (no `@Modifying` queries, no save/delete calls anywhere in `src/main`). Languages are seeded via SQL migration, never mutated through code.
- **Indirect via FK:** the projection `BaseSiteOverviewProjection.defaultLanguageCode` is read in the JPQL at `BaseSiteRepository.findActiveBaseSiteOverviews` (line 15-30). Changing a `BaseSite.defaultLanguage` FK is a `BaseSite` save covered above.

No `@CacheEvict` targets `redisBaseSiteOverviews` anywhere. Confirmed by `grep -rn "@CacheEvict" ... | grep redisBaseSiteOverviews` → zero hits.

### Boot-time / warmup coverage

- `RedisCacheEvictConfig.evictCachesOnStartup` (`RedisCacheEvictConfig.java:39-53`) explicitly clears this cache name (registered at line 27).
- `CacheWarmupService.warmup` at `CacheWarmupService.java:74` warms it: `warmOne("redisBaseSiteOverviews", () -> sizeOf(baseSiteService.getAllBaseSiteOverviews()))`. Runs at `@Order(10)` after `CatalogManager.onAppReady @Order(2)`. Sequence is `evict → catalog-write → warm → ready`.

### Comparison to translations and `redisBaseSites` precedents

- Matches the **`redisBaseSites` shape exactly**: no runtime mutations, boot-only writes upstream of warmup, admin-manual eviction via `CacheAdminController`.
- Diverges from translations: translations has a runtime write path (`updateTranslation`) paired with re-indexing; this cache has no runtime write path at all.

### Verdict

**TTL not needed.** Same correctness story as `redisBaseSites`. The 24-minute value is the **unit-mix transcription bug** flagged in the prior session's "For Mastermind" item 1 — the rest of the file uses `1440` for daily reference-data caches; `24` reads as someone writing hours where the helper consumes minutes. Whether it was the intended value or a slip is moot, because the cache has no correctness need for any TTL.

**Implementation note (if removed):** fold into the same brief that removes the `redisBaseSites` TTL — one call site at `RedisConfig.java:51-57`, same helper (`addListTypeConfig`), same Mastermind-noted helper-signature refactor opportunity. Add a comment near the line referencing the boot/warmup/admin-evict story.

---

## 2. `redisUserInfo`

### TTL configuration found

- **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:38-39`
  ```java
  addTypeConfig(
      UserInfoDTO.class, "redisUserInfo", 30, defaultConfig, cacheConfigs, objectMapper);
  ```
- **Value:** 30 minutes.
- **Documenting comment:** none at the cache-config site, but the `@CacheEvict` mechanism is documented at `DefaultUserService.saveUser` (`DefaultUserService.java:62-81`) with an inline rationale explaining the dual-key shape and the 30-minute staleness window if eviction failed. The TTL itself is implicitly justified there ("either lookup would return pre-update profile data for up to 30 minutes").

### Mutation paths inventoried

Cache backs two `@Cacheable` methods on `DefaultUserService` (`DefaultUserService.java:33-48`):
- `getUserInfoForFirebaseUid` keyed by `#firebaseUid`
- `getUserInfoForId` keyed by `#userId`

So **two entries exist per user** in `redisUserInfo` (one per lookup style).

The cached `UserInfoDTO` (`mapProjectionToUserInfo`, lines 113-160) is hydrated from:
- `UserInfoProjection` (the User row's scalars).
- `userTranslationsService.getUserShortBio(projection.getId())` — reads `UserTranslation` rows.
- `baseSiteCacheService.getBaseSiteForCode` / `getBaseSiteOverviewForCode` (reads other caches).
- `region` / `city` lookups off the resolved BaseSite.

Mutation paths:

1. **`DefaultUserService.saveUser`** (`DefaultUserService.java:62-81`). Carries `@Caching(evict = { @CacheEvict("redisUserInfo", key = "#currentUser.id"), @CacheEvict("redisUserInfo", key = "#currentUser.firebaseUid"), @CacheEvict("redisUserAuth", key = "#currentUser.firebaseUid") })`. Both `redisUserInfo` keys evicted on every save. Default `beforeInvocation = false` means SpEL evaluates after `userRepository.save` completes — `id` is populated by then on new-user creation.

2. **`DefaultUserService.evictUserInfoCache`** (`DefaultUserService.java:101-111`). Same dual-key shape as saveUser; a separate method that only evicts (no write) for callers that mutate via something other than `userService.saveUser`.

3. **`DefaultUserTranslationsService.generateUserTranslations`** (`DefaultUserTranslationsService.java:36-52`) → `userTranslationRepository.saveAll(allTranslations)` at line 51. Mutates the `UserTranslation` rows backing `UserInfoDTO.shortBio`. **No `@CacheEvict`** on this method or path. Today this is correct purely by ordering at the only caller (`DefaultUserFacade.updateCurrentUserData`, line 154-156): `saveUser` runs first → cache is evicted → `generateUserTranslations` runs → next read repopulates with the new shortBio. The risk is documented in code at `DefaultUserFacade.java:108-122` as a known deferred follow-up. The TTL is the 30-minute backstop if anyone reorders those two calls or decouples translation regeneration from the user-update flow.

4. **Callers that write `User` (every one of these flows through saveUser):**
   - `DefaultFirebaseAuthService.createUserSynchronized` (`DefaultFirebaseAuthService.java:121`) — new-user creation. ✓ saveUser.
   - `DefaultUserFacade.updateCurrentUserData` (line 154). ✓ saveUser.
   - `DefaultUserFacade.assignUserRegionAndCity` (line 206). ✓ saveUser.
   - `DefaultUsersFacade.disableUser` (`DefaultUsersFacade.java:42-46`). ✓ saveUser.
   - `DefaultUsersFacade.enableUser` (line 50-56). ✓ saveUser.
   - `TestUsersImportService.initAll` — `@Profile({"!prod"})`, boot-only, predates warmup (`@Order(3)` < `@Order(10)`). Not a runtime concern.

5. **No `@Modifying` JPQL or native SQL on `User`.** Confirmed by `grep -n "@Modifying" UserRepository.java` → zero hits.

What would break under stale read: a profile-update sub-flow that bypasses `saveUser`/`evictUserInfoCache` would leak pre-update display name, profile image, region/city, phone-call preference, or rating into the next 30 minutes of reads for that user.

### Boot-time / warmup coverage

- `RedisCacheEvictConfig.evictCachesOnStartup` clears the cache at startup (`RedisCacheEvictConfig.java:25`).
- **`CacheWarmupService` does NOT warm `redisUserInfo`** — per-user data, populates on demand on first authenticated request per user. `CacheAdminController.java:26` documents this: *"It does NOT re-warm `redisUserInfo` / `redisUserAuth` — those are per-user and populate on demand."*

### Comparison to translations and `redisBaseSites` precedents

- **Diverges from `redisBaseSites`:** has a real runtime write path (`saveUser`), explicitly paired with `@CacheEvict`. Eviction is the correctness mechanism; TTL is a backstop.
- **Same shape as the translations precedent in spirit:** runtime write paired with explicit invalidation. Mechanism is different (`@CacheEvict` vs direct `RedisTemplate.set` re-index), but the contract is the same.
- **Twin to `redisUserAuth`:** identical 30-minute backstop, identical `saveUser` evict-pairing, populated on demand (not by warmup), one specific known-deferred path (`generateUserTranslations`) that relies on ordering.

### Verdict

**TTL is deliberate, load-bearing as backstop.** Removing it would convert the existing `generateUserTranslations` ordering coupling from "stale for 30 min in the failure mode" to "stale until next manual evict or deploy in the failure mode" — a real regression in cache hygiene for a path the team has explicitly decided to leave in place for now. The documentation lives at the eviction site (`DefaultUserService.saveUser` lines 65-68 and 71-72) rather than at the cache-config line.

**Implementation note (if kept, as recommended):** add a four-line comment at `RedisConfig.java:38-39` mirroring the `redisUserAuth` comment style (line 41-46), pointing the reader at `DefaultUserService.saveUser` / `evictUserInfoCache` as the freshness mechanisms and noting the known `generateUserTranslations` deferred follow-up. This aligns with the brief's "For Mastermind on completion" guidance for `no-deliberate` rows.

---

## 3. `redisUserAuth`

### TTL configuration found

- **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:41-47`
  ```java
  // Auth-data cache — keyed by firebaseUid, populated by FirebaseAuthFilter on every
  // authenticated request. TTL (30 min, matching redisUserInfo) is a backstop only; freshness
  // is owned by the explicit @CacheEvict on DefaultUserService.saveUser. Without this cache,
  // every request hammers the DB for `select * from users where firebase_uid=?` plus a lazy
  // load of preferred_language.
  addTypeConfig(
      AuthenticatedUserDTO.class, "redisUserAuth", 30, defaultConfig, cacheConfigs, objectMapper);
  ```
- **Value:** 30 minutes.
- **Documenting comment:** yes, four lines explicitly explaining the backstop role.

### Mutation paths inventoried

Cache backs `DefaultFirebaseAuthService.getCachedAuthData(String firebaseUid)` (`DefaultFirebaseAuthService.java:63-64`), `@Cacheable("redisUserAuth", key = "#firebaseUid")`. Populated on every authenticated request by `FirebaseAuthFilter`.

The cached `AuthenticatedUserDTO` (`AuthenticatedUserDTO.java:14-22`) carries: `userId`, `firebaseUid`, `disabled`, `userRole`, `subscriptionType`, `subscriptionActive`, `baseSiteId`, `preferredLanguage` (LanguageDTO).

Auth-relevant fields and their mutation paths:

- `disabled` — `DefaultUsersFacade.disableUser` / `enableUser` (`DefaultUsersFacade.java:42`, `:52`). Both route through `userService.saveUser`. ✓
- `userRole` — set at user creation in `DefaultFirebaseAuthService.createUserSynchronized` (line 116, 118), which calls `userService.saveUser(newUser)` at line 121. No runtime path mutates `userRole` after creation. ✓ (boot-of-session covered by saveUser)
- `subscriptionType` / `subscriptionActive` — set at creation in the same `createUserSynchronized` method (lines 108-109). No runtime path mutates these after creation (no subscription upgrade/cancellation flow exists in the codebase today). ✓
- `baseSiteId` — `DefaultUserFacade.assignUserRegionAndCity` (line 201) sets `user.setBaseSite(newBaseSite)` then calls `saveUser`. ✓
- `preferredLanguage` — set at user creation in `createUserSynchronized` (line 110-112). No runtime mutation path exists for `User.preferredLanguage`. The `Language` entity itself has zero runtime write paths (see §4/§5 below), so the cached LanguageDTO snapshot is immutable. ✓

All saves route through `DefaultUserService.saveUser`, which evicts both `redisUserAuth` and `redisUserInfo` on the same `@Caching` annotation. Confirmed against `decisions.md` 2026-05-14 ("Connection-pool exhaustion incident") which already established this and recorded the constraint: any future code path mutating an auth-relevant `User` field must route through `saveUser` (or carry its own `@CacheEvict` for `redisUserAuth`).

What would break under stale read (the failure mode the TTL backstops): a future code path that mutates `disabled`/`userRole`/`subscriptionType`/`subscriptionActive` directly without going through `saveUser` would let the cached auth snapshot return pre-mutation values for up to 30 minutes. The user-deletion feature in the backlog is the named risk — the `decisions.md` 2026-05-14 entry explicitly states: *"the planned user-deletion feature must evict `redisUserAuth` for the deleted user's `firebaseUid`."*

### Boot-time / warmup coverage

- `RedisCacheEvictConfig.evictCachesOnStartup` clears the cache at startup.
- **`CacheWarmupService` does NOT warm `redisUserAuth`** — per-user, populates on demand. `CacheAdminController.java:26` documents this.

### Comparison to translations and `redisBaseSites` precedents

- **Diverges from `redisBaseSites`:** runtime write paths exist, explicit `@CacheEvict` pairs.
- **Mirrors the translations precedent in spirit:** runtime writes paired with explicit invalidation.
- **Twin to `redisUserInfo`** as documented above.

### Verdict

**TTL is deliberate, documented, load-bearing.** Comment at `RedisConfig.java:41-46` still matches reality — every auth-relevant mutation path routes through `saveUser`'s `@CacheEvict`, as the 2026-05-14 decision recorded. The 30-minute backstop is the documented insurance against a future path bypassing `saveUser` (most concretely: the planned user-deletion feature, which must add its own eviction; the user-disabling enforcement feature in the backlog touches the same surface).

**Implementation note (kept):** no changes recommended. The existing comment already covers what the brief's "For Mastermind on completion" guidance asks for `no-deliberate` rows.

---

## 4. `redisLanguage`

### TTL configuration found

- **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:59-60`
  ```java
  addTypeConfig(
      LanguageDTO.class, "redisLanguage", 1440, defaultConfig, cacheConfigs, objectMapper);
  ```
- **Value:** 1440 minutes (24 hours).
- **Documenting comment:** none.

### Mutation paths inventoried

Cache backs `DefaultLanguageService.getLanguageByCode(String langCode)` (`DefaultLanguageService.java:26-29`), `@Cacheable("redisLanguage", key = "#langCode")`.

`Language` entity mutation paths in `src/main`: **zero.**

- `grep -rn "languageRepository\." ... | grep -E "save|saveAll|delete|deleteAll"` → no hits.
- `grep -n "@Modifying" LanguageRepository.java` → no hits.
- `LanguageRepository` has only two methods (`findAllLanguages`, `findLangByCode`), both DTO projections — no entity is returned that callers could mutate and save.
- Languages are seeded via SQL migration (per `decisions.md` and CLAUDE.md), never mutated through code.

`@CacheEvict("redisLanguage", ...)` anywhere: **zero hits.**

What would break under stale read: a Language row's `label` or `active` flag changing without eviction would let `getLanguageByCode` return the pre-change value for up to 24 hours. Since no runtime mutation exists, this failure mode requires a human running raw SQL — same operator-forgot-to-evict scenario as `redisBaseSites`, with `CacheAdminController` as the documented remedy.

### Boot-time / warmup coverage

- `RedisCacheEvictConfig.evictCachesOnStartup` clears the cache at startup (registered at line 29).
- `CacheWarmupService.warmup` warms it per-code at `CacheWarmupService.java:93-107`: after warming `redisLanguages` (the list), iterates each `LanguageDTO` and calls `languageService.getLanguageByCode(lang.code())` to populate the per-code cache. Inside a `try/catch` per language so one failure doesn't block others.

### Comparison to translations and `redisBaseSites` precedents

- **Matches `redisBaseSites` shape exactly:** no runtime mutations, populated by warmup before `ACCEPTING_TRAFFIC`, admin-manual eviction available via `CacheAdminController`.
- **Diverges from translations:** translations has a runtime write path; this cache does not.

### Verdict

**TTL not needed.** Identical correctness story to `redisBaseSites`: no runtime write path, warmup covers the post-deploy cold cache, admin evict is the documented manual escape hatch.

**Implementation note:** fold into the `redisBaseSites` removal brief. One call site at `RedisConfig.java:59-60`. Comment mirroring the redisUserAuth-style explanation.

---

## 5. `redisLanguages`

### TTL configuration found

- **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:61-62`
  ```java
  addListTypeConfig(
      LanguageDTO.class, "redisLanguages", 1440, defaultConfig, cacheConfigs, objectMapper);
  ```
- **Value:** 1440 minutes (24 hours).
- **Documenting comment:** none.

### Mutation paths inventoried

Cache backs `DefaultLanguageService.getAllLanguages()` (`DefaultLanguageService.java:20-23`), `@Cacheable("redisLanguages")`. Same underlying `Language` entity as `redisLanguage` — see §4. Zero runtime mutation paths. Zero `@CacheEvict` annotations.

### Boot-time / warmup coverage

- `RedisCacheEvictConfig.evictCachesOnStartup` clears (registered at line 30).
- `CacheWarmupService.warmup` at lines 84-91 warms first (its result feeds the per-code warmup at lines 93-107). On warmup failure of `redisLanguages`, the per-code loop is skipped via the `languages != null` guard.

### Comparison to translations and `redisBaseSites` precedents

Same as `redisLanguage` (§4): matches the `redisBaseSites` shape; diverges from translations.

### Verdict

**TTL not needed.** Same reasoning as `redisLanguage`.

**Implementation note:** fold into the same removal brief. Single call site at `RedisConfig.java:61-62`.

---

## 6. `redisBaseCurrency`

### TTL configuration found

- **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:64-65`
  ```java
  addTypeConfig(
      Currency.class, "redisBaseCurrency", 1440, defaultConfig, cacheConfigs, objectMapper);
  ```
- **Value:** 1440 minutes (24 hours).
- **Documenting comment:** none. (Adjacent observation: this cache stores the raw `Currency` JPA entity, not a DTO. Mentioned in "For Mastermind" §3.)

### Mutation paths inventoried

Cache backs `DefaultBaseCurrencyService.getBaseCurrency()` (`DefaultBaseCurrencyService.java:51-55`), `@Cacheable("redisBaseCurrency")`. Returns the single `Currency` row where `baseCurrency = true`.

`Currency` entity mutation paths in `src/main`: **zero.**

- `grep -rn "currencyRepository" ... | grep -v "import\|@Autowired"` returns only `findByBaseCurrencyTrue` and `findAll`. No save/delete.
- `CurrencyRepository` is minimal (`findByBaseCurrencyTrue` only) — no `@Modifying` queries.
- Currencies are seeded via SQL migration.

**Critical clarification — `DefaultBaseCurrencyService.updateBaseCurrency` does NOT mutate the `Currency` table.** Despite the name, that method (lines 57-92) only:
1. Reads the base currency via `self.getBaseCurrency()` (which hits the cache).
2. Loops over `currencyRepository.findAll()` and fetches per-currency exchange rates from the external `exchangerate-api.com` REST endpoint.
3. Stores the rates in an in-memory `currencyCodeRateMap` (line 88).

No `currencyRepository.save(...)` call. No `Currency` row write. So the `@Scheduled(fixedRate = 86400000)` daily run does not invalidate the cached entity. The 2026-05-14 `@Lazy` self-injection fix made `convertToBaseCurrency` and `updateBaseCurrency` correctly call `self.getBaseCurrency()` so reads of the base currency hit the cache — but that fix concerns *reading*, not *writing*. There is no write path to the `Currency` table at runtime.

`@CacheEvict("redisBaseCurrency", ...)` anywhere: **zero hits.**

### Boot-time / warmup coverage

- `RedisCacheEvictConfig.evictCachesOnStartup` clears (registered at line 31).
- `CacheWarmupService.warmup` warms at lines 75-81: `warmOne("redisBaseCurrency", () -> { var currency = baseCurrencyService.getBaseCurrency(); return currency != null ? 1 : 0; })`.
- Note also `DefaultBaseCurrencyService.onAppReady` at `@Order(1)` calls `updateBaseCurrency()` to populate the in-memory rates map at startup; that runs before warmup and is orthogonal to the cache entry itself.

### Comparison to translations and `redisBaseSites` precedents

- **Matches `redisBaseSites` shape:** no runtime entity mutations, warmup covers cold cache.
- **Diverges from translations:** translations has a runtime write path; this cache does not.

### Verdict

**TTL not needed.** Same correctness story as `redisBaseSites` / `redisLanguage*`.

**Implementation note:** fold into the same removal brief. Single call site at `RedisConfig.java:64-65`.

---

## 7. Translations cache (raw Redis — no `@Cacheable` cache region)

### TTL configuration found

There is no `@Cacheable`-managed cache region for translations. `DefaultTranslationService` writes directly via `StringRedisTemplate` (`DefaultTranslationService.java:46`) using the key pattern `translations:{namespace}:{langCode}` (constant at line 40, formatter at lines 51-53). Writes use `redisTemplate.opsForValue().set(redisKey, encoded)` (line 96) — **no TTL argument passed**, so entries persist indefinitely until overwritten or explicitly deleted.

- **Value:** none (no expiry).
- **Documenting comment:** the no-TTL design is recorded in `decisions.md` 2026-05-14 ("Redis caching & filter control-flow fixes," point 2 and the wider context of point 4) but is not commented in `DefaultTranslationService` itself.

### Mutation paths inventoried

`Translation` entity mutation paths in `src/main`:

1. **`DefaultTranslationService.updateTranslation`** (`DefaultTranslationService.java:206-222`), `@Transactional`. Reads the row, mutates via Hibernate dirty-tracking (`original.setTranslationValue(...)` at line 217), then calls `indexNamespaceLang(translationUpdateRequest.getNamespace(), language)` at line 220. `indexNamespaceLang` (lines 74-105) overwrites the `translations:{namespace}:{langCode}` Redis key with the freshly-read DB contents. This is the explicit cache-invalidation mechanism — overwrite-on-write rather than evict-on-write, same semantic outcome.
2. **No other runtime write path to the `Translation` table.** Confirmed by `grep -rn "translationRepository\." ... | grep -v userTranslation/reviewTranslation/productTranslation` — the system `TranslationRepository` is referenced only inside `DefaultTranslationService`, and the only mutation is the dirty-tracking `setTranslationValue` inside `updateTranslation`. No `@Modifying`, no other `save`.

`updateTranslation` is called from `TranslationsController` admin endpoints (admin-only path). Re-indexing fires for *that namespace + language* pair only — exactly the right scope.

### Boot-time / warmup coverage

- `DefaultTranslationService.initTranslations` (line 55-58, `@PostConstruct`) calls `indexTranslations()` (lines 60-71) which iterates every `TranslationNamespace × Language` combination and overwrites the Redis key with current DB contents. This runs **before** `ApplicationReadyEvent` listeners (PostConstruct fires during bean creation), so translations are fully re-indexed on every boot before `CacheWarmupService` and before traffic is admitted.
- `RedisCacheEvictConfig` does **not** manage the translations keys (they live outside its cache-name set and are not Spring-Cache-managed entries) — but this doesn't matter because `indexTranslations` at PostConstruct does a full SET-overwrite on every key, equivalent to an evict + re-warm in one step.

### Comparison to translations and `redisBaseSites` precedents

This **is** the translations precedent. The brief asks to confirm-not-investigate. Confirmed:
- No TTL on the Redis entries.
- Every write to the `Translation` table goes through `updateTranslation` (only path).
- `updateTranslation` re-indexes the affected `(namespace, language)` Redis key in the same `@Transactional` boundary.
- Boot-time `@PostConstruct` re-indexes everything, ensuring no stale entries survive a deploy.

### Verdict

**No TTL by design, write-then-reindex on every mutation. Deliberate and correct.** Matches the `decisions.md` 2026-05-14 record. No change recommended.

One caveat surfaced and routed to "For Mastermind" as an adjacent observation (§2): `updateTranslation` does **not** call `indexBackendTranslations`, so the in-memory `backendTranslations` map (which `getBackendTranslation` reads from, for push-notification bodies / email content / anywhere the backend emits translated text directly without going through the frontend's Redis path) can go stale until app restart if an admin edits a `BACKEND_TRANSLATIONS` namespace key. This is not a TTL question — there is no TTL involved, there is no Redis involved on the in-memory map path — but it does affect translation freshness, so it surfaced during the investigation. Out of scope for this brief; flagged below.

---

## Summary table

| Cache | Current TTL | Verdict | Remove TTL? |
| --- | --- | --- | --- |
| `redisBaseSiteOverviews` | 24 min | TTL not needed | yes |
| `redisUserInfo` | 30 min | TTL is deliberate, load-bearing as backstop (mirrors `redisUserAuth`; documented at `DefaultUserService.saveUser`, not at the cache-config line) | no-deliberate |
| `redisUserAuth` | 30 min | TTL is deliberate, documented, load-bearing (`decisions.md` 2026-05-14; comment at `RedisConfig.java:41-46` still matches reality) | no-deliberate |
| `redisLanguage` | 1440 min | TTL not needed | yes |
| `redisLanguages` | 1440 min | TTL not needed | yes |
| `redisBaseCurrency` | 1440 min | TTL not needed | yes |
| `redisTranslations` (raw key `translations:{namespace}:{langCode}`) | none | No TTL by design; confirms precedent | no — already no TTL |

---

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
- Part 6 (translations keys): N/A — no translation keys touched. (The translations *cache* was an investigation subject; that's not a Part 6 concern.)
- Part 7 (error contract): N/A this session.
- Part 11 (trust boundaries): N/A this session — no DTOs touched.

## For Mastermind

### Recap for the implementation brief

The four `yes` rows in the summary table all live in `RedisConfig.java` and all flow through the same helper (`addTypeConfig` or `addListTypeConfig`), so they can be removed in one pass. Combined with the `redisBaseSites` removal queued from the prior session, the implementation brief touches:
- `redisBaseSites` (prior session)
- `redisBaseSiteOverviews` (this session)
- `redisLanguage` (this session)
- `redisLanguages` (this session)
- `redisBaseCurrency` (this session)

Five caches, five call-site edits in one file. The helper-signature footgun (prior session "For Mastermind" item 2: `long ttl` implicit minutes) is the right moment to fix in the same brief — moving to `Duration` or `Duration | null` makes "no expiry" explicit and prevents the next 24-vs-1440 transcription accident.

For the two `no-deliberate` rows, the brief's "For Mastermind on completion" guidance says to add a documenting comment if one isn't already in `RedisConfig.java`. `redisUserAuth` already has one (lines 41-46). `redisUserInfo` does not — the documentation lives on `DefaultUserService.saveUser` instead. Recommend a brief comment at `RedisConfig.java:38-39` pointing at `DefaultUserService.saveUser` / `evictUserInfoCache` as the freshness mechanisms and noting the `generateUserTranslations` ordering-dependent path.

### Adjacent observations (out of scope for this brief)

1. **`DefaultUserFacade.updateCurrentUserData` ordering-dependent shortBio freshness.**
   - **File:** `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java:108-122` (the rationale comment) and `:154-156` (the call-site coupling).
   - **Detail:** `userService.saveUser(user)` evicts `redisUserInfo` at line 154; `userTranslationsService.generateUserTranslations(user, updateData.getShortBio())` mutates `UserTranslation` rows at line 156 without paired eviction. Correct today only because of ordering — if anyone reorders or decouples, cached `UserInfoDTO.shortBio` goes stale for up to 30 minutes (or, if the `redisUserInfo` TTL is later removed, indefinitely). The existing Javadoc already documents this as a deferred follow-up; flagging here in case Mastermind wants to promote it to a focused fix before the user-deletion / account-disabling features touch the same area. The robust fix per the existing Javadoc is either `userService.evictUserInfoCache(user)` after `generateUserTranslations`, or `@CacheEvict` on `UserTranslationsService`'s generation method directly.
   - **Severity guess:** medium (correct-by-ordering is fragile; the existing Javadoc warns about exactly this).
   - **Out of scope:** brief is a TTL investigation, not a fix-the-known-deferred work brief.

2. **`DefaultTranslationService.updateTranslation` does not refresh the in-memory `backendTranslations` map.**
   - **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultTranslationService.java:206-222` (`updateTranslation`); `:42` (the in-memory map field); `:55-58` (`@PostConstruct` populates the map once at boot); `:198-204` (`getBackendTranslation` reads from it).
   - **Detail:** `updateTranslation` ends by calling `indexNamespaceLang(namespace, language)` — which only re-indexes the Redis key for that namespace+language. It does **not** call `indexBackendTranslations` (line 261-272), the method that rebuilds the in-memory `backendTranslations` map. So if an admin edits a key in the `BACKEND_TRANSLATIONS` namespace (push-notification bodies, email content, any backend-emitted translated text), the change is in the DB and in Redis but the in-memory map remains the pre-edit version until the next app restart. Every backend-translated string emitted from `getBackendTranslation` will be stale. The current `decisions.md` 2026-05-14 "translations precedent" framing assumes Redis is the cache layer — this in-memory layer wasn't considered.
   - **Severity guess:** medium (real user-visible staleness on a path admins genuinely use to edit notification/email content; the fix is a one-line addition `if (namespace == BACKEND_TRANSLATIONS) indexBackendTranslations();` or unconditional `indexBackendTranslations()` since it's cheap).
   - **Out of scope:** brief is scoped to the seven named caches; the in-memory map is a separate layer.

3. **`redisBaseCurrency` caches a raw JPA entity, not a DTO.**
   - **File:** `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java:64-65`; `src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseCurrencyService.java:51-55`.
   - **Detail:** Every other configured cache stores a DTO or a `record` (`UserInfoDTO`, `AuthenticatedUserDTO`, `BaseSiteDTO`, `BaseSiteOverviewDTO`, `LanguageDTO`). `redisBaseCurrency` alone stores the `Currency` JPA entity directly. The `AuthenticatedUserDTO` comment at `RedisConfig.java:39-46` explicitly notes "Deliberately a record (no JPA entity, no proxies) so it can be safely serialised into Redis without lazy-init or session attachment surprises" — that rationale applies to `Currency` too. `Currency` is a simple entity with no lazy associations referenced from this cache path, so today it serialises fine via the Jackson serializer; but it's a footgun if anyone adds a lazy `@ManyToOne` / `@OneToMany` to `Currency` later. A small `CurrencyDTO` (which already exists in the codebase, used elsewhere for BaseSite allowed-currency lists) would close that footgun.
   - **Severity guess:** low (today's `Currency` shape is serialisable; the risk is future schema drift).
   - **Out of scope:** not a TTL question.

### Process / branch

- Stayed on `dev` per hard rule. Working tree's pre-existing modifications to `pom.xml` and `OglasinoApplication.java` (carried over from the prior session) were not touched and are unrelated to this investigation.
