# Investigation — Translations Redis TTL

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Type:** Read-only investigation, no code changes.

---

## TL;DR

**The translations Redis entries have no TTL anywhere.** The brief's working premise — that there is a 24 h TTL on the translations keys, inherited by analogy from the prior `redisBaseSites` finding — is **wrong**. The translations are written via `StringRedisTemplate.opsForValue().set(key, value)` with no `Duration` argument, no surrounding `expire(...)` call, and they are not registered in `RedisCacheManager`'s per-cache TTL table (because they don't go through `@Cacheable`). The Redis SET issued is a plain `SET key value` — the key persists indefinitely until explicitly overwritten or evicted.

The follow-up fix brief Igor was planning ("remove the translations TTL") is a **no-op** — there is nothing to remove. The current state is already the state Igor wants.

The eviction reasoning Igor sketched does hold: every DB-write path to `oglasino_translation` does have a corresponding Redis reindex (boot-time `@PostConstruct` after `spring.sql.init`, and admin `updateTranslation` via explicit `indexNamespaceLang`). No path mutates the DB rows without also pushing to Redis. So even though there's no TTL, the cache cannot go stale relative to the DB.

The only theoretical gap is at the `(namespace, language)` *bucket* level (not the individual key level): if a `(namespace, lang)` pair ever had zero rows in the DB, `indexNamespaceLang` logs a warning and `return`s without touching Redis — so a previously-populated bucket would linger. In practice this cannot occur because translations are seeded from idempotent SQL files on every boot and no code path deletes rows. Documented for completeness, not load-bearing.

**Verdict: there is no TTL to remove. The investigation refutes the premise; Igor's underlying reasoning is otherwise sound; the trigger Igor was worried about for translations specifically does not exist.** The cold-cache trigger established by the prior connection-pool investigation lives in `redisBaseSites` (`RedisConfig.java:49` — `1440` min), not in translations.

---

## Brief vs reality

This investigation surfaces one finding that contradicts the brief's premise. Per `CLAUDE.md` I surface it before stating the answers.

1. **The brief assumes the translations Redis entries have a 24 h TTL. They do not.**
   - Brief says: "One frequent cold-cache trigger is a **24h TTL expiry**. The translation cache is suspected of having a TTL that serves no purpose and actively causes harm — when it expires, the Next.js server refetches *all* translation namespaces..."
   - Code says (`service/impl/DefaultTranslationService.java:95`):
     ```java
     redisTemplate.opsForValue().set(redisKey, encoded);
     ```
     The two-arg overload (`SET key value`) with no `Duration`. There is no `expire(...)` call anywhere on translation keys, the translations entries are not registered in `RedisConfig`'s `cacheConfigs` map (because they don't use `@Cacheable`), and `application-*.yaml` declares no global default TTL on `StringRedisTemplate`.
   - Why this matters: the assumption is the load-bearing premise of the follow-up fix brief. "Remove the harmful TTL" is meaningless if no TTL is set. The harm Igor described (cold-cache window when the entry expires) cannot originate from translations — it originates from `redisBaseSites` (`RedisConfig.java:49`, `1440` min), which the prior connection-pool investigation already identified.
   - Recommended resolution: do not draft the planned "remove translations TTL" fix brief — there is nothing to change. The translations are already configured exactly as Igor's reasoning prescribes.

The remaining four questions are answered below.

---

## Question 1 — Where is the translations TTL actually set?

### Every translations-key write site

Only one site writes the `translations:<NAMESPACE>:<lang>` keys:

- **`service/impl/DefaultTranslationService.java:95`** — inside `indexNamespaceLang(...)`:
  ```java
  String redisKey = redisKey(namespace, language.code());
  redisTemplate.opsForValue().set(redisKey, encoded);
  ```
  The two-arg `set(key, value)` overload. **No TTL applied.**

The Redis key format is `translations:%s:%s` formatted with `namespace.name()` and `language.code()` — `DefaultTranslationService.java:39` (`REDIS_KEY_PATTERN`) + `:50-52` (`redisKey(...)`).

### Every call site that reaches that write

`indexNamespaceLang(...)` is invoked from three places, all of which end at the same `set(...)` call:

| Caller | File:line | Trigger |
|---|---|---|
| `DefaultTranslationService.indexTranslations()` (boot + admin refresh) | `service/impl/DefaultTranslationService.java:67` (inside the per-namespace × per-language loop) | called from `@PostConstruct initTranslations()` at `:54-57`, AND from `AdminTranslationsController.refreshTranslationsCache()` at `admin/controller/AdminTranslationsController.java:29` |
| `DefaultTranslationService.updateTranslation(...)` (admin edit single key) | `service/impl/DefaultTranslationService.java:220` | called from `AdminTranslationsController.updateTranslation(...)` at `admin/controller/AdminTranslationsController.java:47` |
| `DefaultTranslationService.getOrRebuild(...)` (lazy on-demand refill) | `service/impl/DefaultTranslationService.java:242` | called from `getTranslationsForNamespace(...)` at `:122` and `getFilteredTranslations(...)` at `:159`, both inside the `useCache` branch |

All three paths converge on `:95`. There is no second write site, no `opsForValue().set(key, value, duration)` overload anywhere in the file (or in the codebase) targeting translation keys, and no `redisTemplate.expire(redisKey, ...)` call on translation keys.

### Verifying no TTL is applied anywhere

I checked five places a TTL could be hiding:

1. **The `set(...)` call itself** — `DefaultTranslationService.java:95`: two-arg overload, no `Duration`. Plain Redis `SET`.
2. **A separate `expire(...)` or `expireAt(...)` call** — `grep -rn "expire\b" src/main/java/`: only matches are `RateLimitConfig.java:48` (Bucket4j), `DefaultRedisViewCounterService.java:35, 76` (view counter), `RedisUploadOwnershipService.java:38` (R2 upload tokens). **Zero matches on translation keys.**
3. **`RedisConfig.java` per-cache `entryTtl`** — `RedisConfig.java:38-65` registers TTLs for `redisUserInfo`, `redisUserAuth`, `redisBaseSites` (1440 min), `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. **No `translations:*` entry.** This is the table the prior connection-pool investigation cited for the `redisBaseSites` 24 h TTL — translations aren't in it because they don't go through `@Cacheable` / `RedisCacheManager`; they use `StringRedisTemplate` directly.
4. **Global default TTL on `StringRedisTemplate`** — `application-prod.yaml`, `application-stage.yaml`, `application-dev.yaml` declare `spring.data.redis.{host,port,password,timeout}` and `spring.cache.redis.cache-null-values: false`. **No `spring.cache.redis.time-to-live` and no `StringRedisTemplate` `@Bean` overriding default behaviour.** `RedisConfig.java:27-30` explicitly notes that the framework-default `StringRedisTemplate` and `LettuceConnectionFactory` are used unmodified.
5. **A Redis server-side default-TTL policy** — out of repo scope (lives in the Redis server config, not source). Not consulted because Igor confirmed the deployment uses a Dockerised Redis with stock config (per the prod-deployment memory). No server-side policy is configured by anything in this repo.

### Answering the brief's sub-questions

- **The exact file:line of every place a TTL is applied to a translations key:** none. The surface area of the eventual fix is empty.
- **The single write site is** `service/impl/DefaultTranslationService.java:95`, and the write applies no TTL.

---

## Question 2 — Is admin reindex the ONLY write path to the translation *data*?

### Every code path that writes/updates rows in `oglasino_translation`

Two paths, both expected:

| # | Path | Triggers Redis reindex? |
|---|---|---|
| 1 | **`spring.sql.init` at boot** — runs `data/translations/*.sql` (12 files: 0001-data-web-*.sql + 0002-data-*.sql + 0003-data-filter-option-*.sql × 4 languages). Every file is one big `INSERT ... VALUES (...) ON CONFLICT (id) DO UPDATE SET language_id=..., translation_namespace=..., translation_key=..., translation_value=...;` (verified at `data/translations/0001-data-web-translations-EN.sql:1081-1085`). Configured in all three profiles at `application-{prod,stage,dev}.yaml` under `spring.sql.init.data-locations` with `mode: always`. | **Yes**, transitively. `spring.sql.init` runs as part of the same `DatabaseInitializer` wiring that fires before bean-initialization-complete (per the comments at `application-prod.yaml:36-40`). `@PostConstruct initTranslations()` at `DefaultTranslationService.java:54-57` then calls `indexTranslations()` which re-indexes every `(namespace, language)` from the freshly-seeded DB to Redis. |
| 2 | **`DefaultTranslationService.updateTranslation(...)`** (`service/impl/DefaultTranslationService.java:205-221`) — called from `AdminTranslationsController.updateTranslation(...)` at `admin/controller/AdminTranslationsController.java:44-49` (POST `/api/secure/admin/translations/update`, `@PreAuthorize("hasRole('ADMIN')")`). Mutates a single `Translation` row via `original.setTranslationValue(...)` at `:217`. | **Yes**, explicitly. Line `:220` calls `indexNamespaceLang(translationUpdateRequest.getNamespace(), language)` which re-builds the entire `translations:<NS>:<lang>` Redis blob for the affected bucket. |

### What I checked to be sure these are the only paths

- `grep -rn "TranslationRepository\b" src/main/java` — the `TranslationRepository` interface (the one for the `Translation`/`oglasino_translation` entity) is autowired only into `DefaultTranslationService` (`:44`). The three other repositories (`ProductTranslationRepository`, `UserTranslationRepository`, `ReviewTranslationRepository`) are for *different* entities and different tables (`product_translation`, `user_translation`, `review_translation`) — they are not the translation seed/UI text store and are not the source of `translations:<NS>:<lang>` Redis blobs.
- `grep -rn "oglasino_translation" src/main` — only matches:
  - The seed SQL files (path 1 above).
  - The Flyway schema migration `db/migration/V1__init_schema.sql` (CREATE TABLE, constraints, indexes — schema only, no data writes).
  - The entity `@Entity(name = "oglasino_translation")` and the repository JPQL `FROM oglasino_translation t`.
- `grep -rn "translationRepository\." src/main/java` — only finder calls (`findByNamespaceAndLanguage`, `findByTranslationNamespace`, `findByTranslationNamespaceAndTranslationKeyAndLanguage_Code`). **No `.save(...)`, `.saveAll(...)`, `.delete*(...)`, no `@Modifying` query, no native `INSERT`/`UPDATE`/`DELETE` issued from Java code.**
- Flyway: only `V1__init_schema.sql` exists; it creates the table but writes no rows.

The `updateTranslation` write deserves one technical note: the method itself has no `@Transactional` annotation. The mutation propagates because OSIV (`spring.jpa.open-in-view` is unset → Spring Boot default `true`) keeps the entity manager bound to the request thread; the entity returned from `findByTranslationNamespace...` (`:210-215`) is therefore still in the persistence context when `setTranslationValue(...)` is called at `:217`; the subsequent `configurationService.updateConfiguration(...)` call inside `updateTranslationsVersion()` (`:223-228`) opens its own write transaction, and Hibernate's dirty-checking flushes the modified `original` entity at that tx's commit. Then `indexNamespaceLang(...)` reads the now-persisted value back from the DB and rebuilds the Redis blob. The end-to-end sequence is correct in practice; flagged here only because the implicit-flush mechanism is non-obvious — if OSIV is ever disabled, the admin update silently stops persisting. (Adjacent observation, not in scope.)

### The load-bearing answer

**No, there is no way for the DB translation data to change without the Redis copy being updated.** Every Java-side write path triggers a Redis reindex (path 1 via `@PostConstruct`, path 2 via explicit `indexNamespaceLang`). The SQL seed itself runs before `@PostConstruct` so the indexing always operates on the post-seed DB state.

Igor's reasoning holds. There is no path that mutates translation rows in the DB while leaving Redis stale.

### Caveats worth flagging (low-severity, off-scope)

- The SQL seed's `ON CONFLICT (id) DO UPDATE SET ... translation_value = EXCLUDED.translation_value` means **admin edits to translation values are clobbered on every boot** — the seeded value wins. This is a separate concern (admin-edits-vs-seed-source-of-truth) and is out of scope for this brief, but it's worth recording: an admin who edits `ERRORS:product.name.banned_words` will see their edit revert on the next deploy unless the SQL seed file is also updated. Flagging for Mastermind as an adjacent observation.
- The SQL seed only INSERTs/UPDATEs rows declared in the seed files. It does not DELETE rows from `oglasino_translation`. So if a row was hand-deleted directly in the DB (no app endpoint deletes rows), the seed would re-insert it on next boot (because the row is in the seed file). This is well-behaved.

---

## Question 3 — Does "reindex" fully replace, or just overwrite?

### The unit of reindex

The Redis key for translations is one key per `(namespace, language)` pair, holding a single gzipped+base64 JSON map of **all** keys for that pair. So "replace at the (namespace, lang) blob level" is the natural granularity, not per-individual-translation-key.

### `indexNamespaceLang(...)` — `DefaultTranslationService.java:72-104`

```java
public void indexNamespaceLang(TranslationNamespace namespace, LanguageDTO language) {
  ...
  var translations = translationRepository.findByNamespaceAndLanguage(namespace, language.id());
  if (translations.isEmpty()) {
    log.warn("No translations found for {} - {}", namespace, language.code());
    return;                                    // <-- early return; existing Redis blob NOT touched
  }
  var map = translations.stream().collect(Collectors.toMap(...));
  ...
  redisTemplate.opsForValue().set(redisKey, encoded);   // full overwrite of the blob
}
```

The DB query selects all current rows for the `(namespace, lang)` pair, builds a fresh `Map<key, value>`, and overwrites the Redis blob with that map.

**Within a populated `(namespace, lang)` bucket** — full replace at the blob level. If a translation row was deleted from the DB, the rebuilt map doesn't include that key, so the next reindex makes it disappear from Redis. **No within-bucket orphans are possible.**

**For an entirely-empty `(namespace, lang)` bucket** — the `if (translations.isEmpty()) return;` branch *does* leave a stale blob in Redis if one was previously written. In practice this never fires because:

- The seed SQL files always populate at least the four languages × the namespaces that ship with the product.
- No code path deletes rows from `oglasino_translation`.

So the only way to land in this branch in production is a manual SQL delete that empties a bucket — which is not a supported operation. Flagging for completeness; not load-bearing for the TTL decision.

### `indexTranslations()` — `DefaultTranslationService.java:60-70`

The "full reindex" entry point:

```java
public void indexTranslations() {
  indexBackendTranslations();              // in-memory map for the BACKEND_TRANSLATIONS namespace
  var languages = languageRepository.findAllLanguages();
  for (TranslationNamespace namespace : TranslationNamespace.values()) {
    for (LanguageDTO lang : languages) {
      indexNamespaceLang(namespace, lang);
    }
  }
}
```

Iterates over `TranslationNamespace.values()` × `languageRepository.findAllLanguages()` and calls `indexNamespaceLang` for each pair. This is the same loop whether the trigger is the boot-time `@PostConstruct initTranslations()` (`:54-57`) or the admin `AdminTranslationsController.refreshTranslationsCache()` (`:27-31`).

**Verdict for both paths (admin reindex AND boot `@PostConstruct`):** they are **full replace at the (namespace, lang) blob level**. The blob is rebuilt from a fresh DB read, then overwritten. Within-bucket orphans cannot persist.

The only remaining orphan vectors are:

1. A `(namespace, lang)` that becomes entirely empty in the DB — Redis blob lingers (the `isEmpty() return` branch). Not reachable in practice.
2. A `TranslationNamespace` enum value that is removed from Java but whose Redis blob was previously written — the loop no longer iterates that namespace, so the stale blob would linger forever. Not reachable in practice without explicit Redis cleanup (or a fresh Redis like the Dockerised one used in prod, which is brought up empty).
3. A `Language` row removed from the `oglasino_language` table — `findAllLanguages()` would no longer return it, the loop skips it, the stale blob lingers. Not reachable in practice.

None of these is created or worsened by the absence of a TTL. A TTL would only mask them by eventually expiring everything; removing a TTL (if one existed) would expose them. **Since there is no TTL to remove, this is academic.**

---

## Question 4 — What exactly happens on a translations cache miss today?

### The read-path trace

`GET /api/public/translations?namespace=<NS>&lang=<lang>`:

1. **Filter chain** — same as for any `/api/public/*` request. Upstream of this controller. Per the prior connection-pool investigation, on cold caches `BaseSiteFilter` and `CurrentLanguageFilter` can each consume a connection before the request reaches the controller. **Out of scope for this brief; the translations cache-miss cost below excludes the filter-chain cost.**
2. **`TranslationsController.getTranslations(...)`** — `controller/TranslationsController.java:30-38`. Calls `translationService.getTranslationsForNamespace(TranslationNamespace.valueOf(namespace), lang)`.
3. **`DefaultTranslationService.getTranslationsForNamespace(...)`** — `service/impl/DefaultTranslationService.java:106-141`:
   1. `languageService.getLanguageByCode(langCode)` (`:114`) — `@Cacheable("redisLanguage")` with a 24 h TTL. **On hit:** no DB. **On miss:** 1 SELECT via Spring Data JPA's per-method tx (acquires one JDBC connection briefly). Same per the prior connection-pool investigation.
   2. `configurationService.getBooleanConfig("cache.translations.redis")` (`:118`) — **in-memory cache** populated at startup by `DefaultConfigurationService.onAppReady`. No DB, no Redis.
   3. `getOrRebuild(redisKey, namespace, language)` (`:122`) — the translations cache read, detailed below.
4. **`getOrRebuild(...)`** — `service/impl/DefaultTranslationService.java:231-249`. The double-checked-locked refill helper:
   ```java
   private String getOrRebuild(String redisKey, TranslationNamespace namespace, LanguageDTO language) {
     var encoded = redisTemplate.opsForValue().get(redisKey);                  // (A) first GET
     if (Objects.isNull(encoded)) {
       var lockKey = (namespace.name() + "_" + language.code()).intern();       // (B) per-pair intern lock
       synchronized (lockKey) {
         encoded = redisTemplate.opsForValue().get(redisKey);                   // (C) second GET (re-check)
         if (Objects.isNull(encoded)) {
           indexNamespaceLang(namespace, language);                             // (D) 1 SELECT + 1 SET
           encoded = redisTemplate.opsForValue().get(redisKey);                 // (E) third GET (read-back)
         }
       }
     }
     return encoded;
   }
   ```

### Confirming the synchronized-block claim from the brief

The brief says: "the prior investigation noted a `synchronized` block keyed on `(namespace + "_" + langCode).intern()` serializes refills per key. Confirm that's accurate."

**Confirmed.** Line `:236`:
```java
var lockKey = (namespace.name() + "_" + language.code()).intern();
```

`.intern()` canonicalises the string in the JVM string-intern pool, so two threads computing the same `<NAMESPACE>_<lang>` end up with the *same* object reference, and `synchronized(lockKey)` blocks on the same monitor. With 22 namespaces × 3 active languages = up to 66 distinct lock objects, the JVM intern table holds them indefinitely (one per pair); this is a fixed-size, negligible memory footprint.

**Per-key serialization shape:**

- Concurrent misses on the **same** `(namespace, lang)`: only the first thread crosses the inner `if`, runs `indexNamespaceLang(...)` (one DB query, one Redis SET, one Redis GET), and exits. All other threads block at `synchronized(...)`, then enter, find the value present on the re-check at `(C)`, and return without a DB query. **Per-pair DB cost: exactly 1 SELECT for the entire thundering herd on that key.**
- Concurrent misses on **different** `(namespace, lang)` pairs: each locks a different monitor, so all proceed in parallel. **Up to one DB query per distinct pair, all concurrent.** This is the worst case for connection-pool pressure on the translations cache specifically — bounded by the number of distinct `(namespace, lang)` pairs the Next.js bootstrap fetches in parallel.

### DB cost per cold-miss path (Question 4's narrow scope)

Inside the translations cache path *only* (excluding filter-chain costs covered by the prior investigation):

| Step | Operation | DB? | Connection? |
|---|---|---|---|
| (A) | Redis GET | no | no |
| (B) | `intern()` + `synchronized` enter | no | no |
| (C) | Redis GET (re-check) | no | no |
| (D-1) | `translationRepository.findByNamespaceAndLanguage(namespace, language.id())` | **1 SELECT** | **1 connection acquired, briefly** (Spring Data JPA's per-method `@Transactional(readOnly = true)` on `SimpleJpaRepository`) |
| (D-2) | JSON serialise + gzip + base64 (in-memory) | no | no |
| (D-3) | `redisTemplate.opsForValue().set(redisKey, encoded)` | no | no |
| (E) | Redis GET (read-back) | no | no |

**Cold-miss cost per `(namespace, lang)`: 1 SELECT, one short JDBC connection acquisition, plus 3 Redis ops (or 2 if the re-check at (C) wins). No outer transaction wraps `getOrRebuild` or `indexNamespaceLang` — the only tx is the implicit one Spring Data JPA opens for the `findByNamespaceAndLanguage` call itself.**

The `findByNamespaceAndLanguage` query returns a constructor projection (`@Query` JPQL constructing `TranslationDTO` directly — see `repository/TranslationRepository.java:14-26`), so there's no entity-level work, no associations loaded, no N+1 risk. One row-set, one round-trip.

### Why removing the (non-existent) TTL would have been "safe to remove"

If a TTL did exist (it doesn't), removing it would mean:

- The cache could only go cold via: (a) Redis flush/restart of the Redis container, (b) the `getOrRebuild` race where the key truly is missing because the boot indexer ran before Redis was reachable (unlikely; Spring Boot blocks on Redis connectivity at startup with `initialization-fail-timeout`), (c) admin-triggered evict — though there's no such admin endpoint targeting the translations keys specifically (the `CacheAdminController` operates on `RedisCacheManager`-registered caches, which translations are not in).
- A normal long-running deploy would just have warm entries forever, refilled only by an actual reindex.

In other words: removing a TTL would have eliminated the daily cold-miss window the brief described. Since there's no TTL today, that window never existed for translations in the first place.

---

## Verdict

**The TTL is safe to remove because the TTL does not exist.** The investigation refutes the brief's premise. There is no follow-up fix brief needed for translations TTL.

The full picture:

- Translations write site (`DefaultTranslationService.java:95`) issues a plain `SET key value` with no expiry.
- No `expire(...)` call exists anywhere on translation keys.
- Translations are not in `RedisCacheManager`'s per-cache TTL table.
- No global Redis-template TTL is configured.
- DB-write paths to `oglasino_translation` are exactly two — boot SQL seed (followed by `@PostConstruct` reindex) and admin `updateTranslation` (followed by explicit `indexNamespaceLang`) — both of which keep Redis in sync.
- The reindex (both admin and boot) is a full replace at the `(namespace, lang)` blob level; within-bucket orphans cannot occur.
- The cold-miss path is correctly per-pair serialized by `intern()`-synchronized; per-pair DB cost is exactly 1 SELECT, bounded.

If Igor's intent was to confirm "translations should have no TTL, and the eviction picture justifies that," the investigation confirms both:

1. **They already have no TTL.**
2. **The eviction picture justifies that.**

The cold-cache trigger Igor was worried about for translations specifically is illusory — the only TTL-driven cold-cache trigger that actually exists in this codebase is the `redisBaseSites` 24 h TTL (`RedisConfig.java:49`), which the prior connection-pool investigation already established. The pool exhaustion *visible at* `/api/public/translations` is upstream-filter-chain damage, not translation-cache damage.

---

## What this investigation rules out / confirms

- **No TTL is set on any `translations:<NS>:<lang>` Redis key.** Confirmed.
- **The single Redis-write site is `DefaultTranslationService.java:95`, two-arg `set(key, value)`, no `Duration`.** Confirmed.
- **The DB-write paths to `oglasino_translation` are exactly: (1) `spring.sql.init` at boot, (2) `DefaultTranslationService.updateTranslation` triggered by `AdminTranslationsController.updateTranslation`.** Confirmed.
- **Both DB-write paths trigger a Redis reindex on the affected scope** — full reindex via `@PostConstruct initTranslations` after the seed; per-`(namespace, lang)` reindex via `indexNamespaceLang` after admin edit. Confirmed.
- **Reindex is full-replace at the `(namespace, lang)` blob level for both admin and boot paths.** Confirmed.
- **Cold-miss DB cost is exactly 1 SELECT per `(namespace, lang)` pair, serialized per pair by `intern()`-synchronized double-checked-locking.** Confirmed.

---

## For Mastermind

- **The premise of the planned "remove translations TTL" follow-up fix brief is invalid; do not draft it.** The state Igor wants is already the state on disk. The "fix" would be a no-op edit. If the goal is to harden against the cold-cache-exhausts-pool failure mode, the lever is on `redisBaseSites` (and `CacheWarmupService` gating readiness) per the prior connection-pool investigation — not on translations.
- **The translation cache is structurally well-behaved.** No TTL, correct per-pair refill serialization, correct full-replace reindex, no orphan vectors that matter in practice. The `synchronized(...).intern()` pattern is mildly unconventional but does what the brief described and the file:line evidence confirms.
- **Adjacent observations (low-severity, off-scope, not fixed):**
  - `DefaultTranslationService.updateTranslation` (`:205-221`) has **no `@Transactional`** but relies on OSIV's request-scoped persistence context to flush the dirty entity at the next tx commit (which fires inside `updateTranslationsVersion()` → `configurationService.updateConfiguration(...)`). It works today; it silently breaks if OSIV is ever disabled. Worth either an explicit `@Transactional` on `updateTranslation` or a comment, but out of scope. Severity: low.
  - The SQL seed's `ON CONFLICT (id) DO UPDATE SET translation_value = EXCLUDED.translation_value` clobbers admin edits on every boot — the seed file value wins. This means admin-edited translations are *not* durable across deploys unless the seed file is also updated. Possibly intentional (seed is source of truth), possibly a gap (admin UI implies edits stick). Out of scope; flagging. Severity: low–medium depending on whether admin edits are supposed to persist past redeploy.
  - `indexNamespaceLang` early-returns on `translations.isEmpty()` without clearing the existing Redis blob (`:80-83`). Not reachable in practice (seeds keep every bucket populated, no delete-translations endpoint exists), but if the code ever grows a delete path, a stale blob could linger. Out of scope; flagging. Severity: low.
  - The translations Redis layer uses `StringRedisTemplate` directly while reference-data caches (`redisBaseSites` etc.) use `@Cacheable` through `RedisCacheManager`. Two parallel patterns for "cached reference data." Not wrong, just inconsistent. The translations pattern can't easily be ported to `@Cacheable` because the value is gzipped+base64 — but it's worth noting that the prior connection-pool fix work (e.g. per-key in-flight deduplication on misses) would need separate implementations for the two patterns. Out of scope; flagging. Severity: low.
- **One thing I deliberately did not do:** I did not connect to the deployed Redis to verify the absence of a TTL on a live key via `TTL translations:ERRORS:en` (which would return `-1` for "no expiry"). The brief is read-only and code-level; the source-of-truth for what the application sets is the code, which is unambiguous. If Igor wants a live confirmation, the one-line check is `redis-cli TTL translations:ERRORS:en` on the prod Redis (expected output: `-1`).
