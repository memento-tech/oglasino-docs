# Audit — version-checksums-per-language — oglasino-backend

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-29
**Mode:** READ-ONLY. No code changed. This documents current reality so the per-(namespace, language) change can be specified against it.

Every fact is marked `[read]` (verified directly in source by me this session) or `[inferred]` (deduced from read code, not observed running). Line numbers are from the files as they stand on `dev`.

---

## 1. The endpoint

- **Route & method** `[read]`: `VersionController` (`controller/VersionController.java:16-18`) is `@RestController` at `@RequestMapping("/api/public/versions")`. The single handler `getVersions()` (`:32-33`) is annotated `@GetMapping` with **no** path suffix → exact route is `GET /api/public/versions`.
- **Security matcher — genuinely public** `[read]`: `security/config/SecurityConfig.java` authorizes `.requestMatchers("/api/public/**", "/api/auth/**", "/internal/**").permitAll()`. `/api/public/versions` falls under `/api/public/**` → `permitAll()`. No auth, no role. (`FirebaseAuthFilter` may still populate a principal if a token happens to be present, but the matcher does not require one.)
- **Response shape** `[read]`: returns `VersionsResponseDTO` (`dto/VersionsResponseDTO.java`), a record:
  ```java
  public record VersionsResponseDTO(Map<String, String> translations, Map<String, String> catalog) {}
  ```
  - `translations` is keyed by **namespace name only** (e.g. `"COMMON" → "abc123…"`). One checksum per namespace. **No language dimension.** Built at `VersionController.java:34-39` by iterating `ALL_NAMESPACES` and reading `configurationService.getConfig("translations.checksum." + ns.name())`.
  - `catalog` is keyed by base-site code (`"rs"`, `"rsmoto"`, `"me"`), one checksum per active base site (`:41-46`). Out of this feature's scope but shares the same DTO/cache machinery.
- **Cache-Control** `[read]`: set via `ReferenceDataCacheControl.INSTANCE` (`VersionController.java:48-49`; constant at `config/ReferenceDataCacheControl.java:8-11`) →
  `Cache-Control: max-age=300, public, stale-while-revalidate=86400` (5 min fresh, 1 day stale-while-revalidate).

**Verdict:** the endpoint returns exactly one checksum per namespace today.

---

## 2. The checksum computation

Owner: `VersionChecksumService.computeTranslationChecksum(TranslationNamespace)` (`service/impl/VersionChecksumService.java:217-229`).

- **Row set gathered per namespace** `[read]`: `translationRepository.findByTranslationNamespace(namespace)` (`:218`) — **all** rows for that namespace, **across all languages**, no language filter, no outer join to the language table (so only languages that actually have rows appear).
- **Deterministic ordering before hashing** `[read]` (`:221-224`): sorted by `language.getCode()` **then** `translationKey`:
  ```java
  .sorted(Comparator.comparing((Translation t) -> t.getLanguage().getCode())
                    .thenComparing(Translation::getTranslationKey))
  ```
- **Hashed material** `[read]` (`:225-226`): each row mapped to `translationKey + "|" + translationValue`, joined with `"\n"`. **The language code is used for ordering but is NOT part of the hashed string.**
- **Hash algorithm & 16-hex output** `[read]` (`sha256Hex16`, `:327-339`): `SHA-256` over the UTF-8 bytes of the input; full 32-byte digest rendered lowercase hex (`%02x`), then `hex.substring(0, 16)` → **first 16 hex chars = first 8 bytes** of the digest.
- **Language mixing — the central fact for this feature** `[read]`: the input mixes **all languages together** for a namespace. There is no per-language segmentation in the hashed material. A Russian-only edit changes one `key|value` line in the combined string, which changes the single namespace checksum — indistinguishable today from a Serbian or English edit. The language sort key keeps ordering stable but is invisible to the hash.

> Edge note: because `key|value` is concatenated without the language code, two languages whose `(key, value)` pairs were identical would collide into the same line — harmless for change-detection, but worth knowing: the hash identifies *content*, not *(content, language)*.

---

## 3. Language model in the data — **most important finding**

- **Distinct languages with stored rows: FOUR** `[read]`. `data/core/data-languages.sql` seeds:
  | id | code  | label       |
  |----|-------|-------------|
  | 1  | `sr`  | Serbian     |
  | 2  | `cnr` | Montenegrin |
  | 3  | `en`  | English     |
  | 4  | `ru`  | Russian     |
- **CNR has its OWN stored rows, distinct from SR** `[read]`. This is the load-bearing finding. There is a complete, separate set of Montenegrin seed files, all using `language_id = 2`:
  - `data/translations/0001-data-web-translations-CNR.sql`
  - `data/translations/0002-data-translations-CNR.sql`
  - `data/translations/0003-data-filter-option-translations-CNR.sql`
  - e.g. `0001-...-CNR.sql:2` → `(613, 2, 'DIALOG', 'portal.config.language.label', 'Promijeni jezik', …)` — a genuinely different value from the SR row (`'Promeni jezik'`, `language_id 1`).
  CNR is a real fourth language in the translation data, **not** a runtime alias of SR.
- **No aliasing on the translation read/hash path** `[read]`. The only `cnr → SR` collapse in the codebase is `moderation/SupportedLanguage.fromCode()` (`case "sr", "me", "cnr" -> SR;`), and its own class comment scopes it to "the content moderation layer." That code is **not** on the checksum or translation-serving path. The checksum path (`computeTranslationChecksum`) reads `t.getLanguage().getCode()` verbatim; the serving path (`DefaultTranslationService.loadTranslationsFromDb` → `languageService.getLanguageByCode` → `LanguageRepository.findLangByCode`, exact `WHERE l.code = :langCode`) and `CurrentLanguageFilter` pass the code through unchanged. No normalization, no collapse.
- **Literal language-code values in stored rows** `[read]`: exactly `'sr'`, `'cnr'`, `'en'`, `'ru'`. No `'me'`, no locale-region variants (`'sr_RS'`, `'en_US'`, …) anywhere in the language table or translation seeds.

> Implication for the feature (surfaced, not designed): conventions Part 9 says "Montenegrin (me/cnr) aliases to SR," but that aliasing is **routing/moderation only**. In the translation data CNR is independent, so a per-(namespace, language) checksum set will legitimately contain a distinct `cnr` checksum per namespace. The per-language space is **4 languages**, not 3.

---

## 4. The namespace set

- **`TranslationNamespace` enum** (`entity/TranslationNamespace.java:3-40`): **22 constants** `[read]` —
  `COMMON, COMMON_SYSTEM, ERRORS, VALIDATION, PAGING, BUTTONS, INPUT, DIALOG, HEADER, FOOTER, NAVIGATION, INTRO, EXTRA_PRODUCTS, COOKIES, MESSAGES_PAGE, DASHBOARD_PAGES, ADMIN_PAGES, ABOUT_PAGE, FREE_ZONE_PAGE, PRICING_PAGE, METADATA, BACKEND_TRANSLATIONS`.
- **`/versions` returns ALL 22 — no filter** `[read]`. `VersionController.ALL_NAMESPACES = Arrays.asList(TranslationNamespace.values())` (`:20-21`) and the rebuild loop `for (TranslationNamespace namespace : TranslationNamespace.values())` (`VersionChecksumService.java:145`) both iterate the full enum with no exclusion.
- **`ADMIN_PAGES` and `BACKEND_TRANSLATIONS` are included** `[read]` — confirmed both in the enum iteration and in the seed rows (`data-configuration.sql:116, 121`). The brief's context note (these two are "expected backend-side but not consumed by mobile") is correct on intent, but the backend `/versions` response does **not** filter them out; any filtering happens mobile-side (the mobile enum is 20, per `oglasino-docs/decisions.md` 2026-05-29 amendment #3 — backend is not aware of that distinction).

---

## 5. Caching

Two distinct concerns — the **checksum values** and the **translation payloads** — cached differently.

### 5a. Checksum values (what `/versions` serves)
- **Stored in the `configuration` table** `[read]`, one row per namespace, key pattern `translations.checksum.<NAMESPACE>` (`data/configuration/data-configuration.sql:100-121`, ids 58–79), seeded with empty-string values. Catalog: `catalog.checksum.<code>` (ids 80–82).
- **In-memory layer in addition to Redis** `[read]`: `DefaultConfigurationService.configurationCache` (`Map<String,String>`) is the authoritative read path. Populated at boot `@Order(1)` from the DB; `updateConfiguration()` writes DB **and** the in-memory map together. `VersionController.getVersions()` reads checksums from this in-memory map via `configurationService.getConfig(...)` — **Redis is not on the checksum delivery path.** `[read]`
- **Writer** `[read]`: `VersionChecksumService.persistChecksum()` (`:313-325`) is the sole writer; it throws if the config key is missing (enforces that all keys are pre-seeded before boot recompute).

### 5b. Translation payloads (what the mobile client fetches per namespace)
- **Redis cache `redisTranslations`** `[read]`. Crucially, this cache is **already keyed per (namespace, language)**: key shape `"<NAMESPACE>:<langCode>"` (e.g. `COMMON:en`), value type `Map<String,String>` (translationKey → value). Built at `VersionChecksumService.rebuildTranslationCache()` (`:201-206`, `cacheKey = namespace.name() + ":" + language.getCode()`) and read via `@Cacheable(key = "#namespace.name() + ':' + #langCode")` on `DefaultTranslationService.getTranslationsForNamespace()`. No TTL (reference data).
- So the payload layer is already per-language; **only the checksum layer is namespace-only.** That asymmetry is the heart of this feature.

### 5c. Invalidation / rebuild triggers
- **Boot** `[read]`: `VersionChecksumService.onAppReady()` `@Order(3)` → `rebuildTranslationCachesIfNeeded()` (`:143-183`). Per namespace: recompute the namespace checksum, compare to the persisted config value, and also check whether **any** language is missing from Redis (`:152-157`). Rebuild (all languages of that namespace) + persist checksum if `checksumChanged || anyLanguageMissingFromRedis`.
- **Admin translation edit** `[read]`: `DefaultTranslationService.updateTranslation()` calls `versionChecksumService.rebuildTranslationCacheAsync(namespace, language)` (`@Async`, `VersionChecksumService.java:185-199`). It rebuilds the single edited `(namespace, language)` Redis payload, then **recomputes the namespace-wide checksum** and persists it. Returns at transaction commit; Redis + checksum update on a background thread.
- Both boot and admin-edit therefore touch the same per-namespace checksum config key.

---

## 6. Per-language feasibility seams

Each place that today assumes "one checksum per namespace" and must become "one per (namespace, language)":

1. **Computation** — `VersionChecksumService.computeTranslationChecksum(namespace)` (`:217-229`). *Today:* gathers all languages, language code is sort-only, hashes one combined `key|value` string per namespace. *Needs:* compute per language — filter rows to one `langCode` (or group the fetched rows by language) and hash each group separately. The existing per-language Redis fetch `loadTranslationsFromDb(namespace, langCode)` is a ready row source.
2. **Configuration persistence keys** — `data-configuration.sql:100-121`, key `translations.checksum.<NS>`; written by `persistChecksum` (`:313-325`), which **throws if the key is not pre-seeded**. *Today:* 22 keys. *Needs:* `22 × N_languages` keys (`translations.checksum.<NS>.<lang>` or similar), pre-seeded — **or** `persistChecksum`/the boot path relaxed to create keys on first write (current design assumes pre-seeded existence; this is a real constraint, not optional).
3. **Response DTO** — `VersionsResponseDTO.translations: Map<String,String>` keyed by namespace. *Needs:* either a nested `Map<String, Map<String,String>>` (namespace → lang → checksum) or a flat composite key (`"<NS>.<lang>"`). DTO change is a wire-contract change consumed by `oglasino-expo` (`bootFreshness.ts`, per docs) — coordinate.
4. **Boot rebuild loop** — `rebuildTranslationCachesIfNeeded()` (`:143-183`). *Today:* one checksum per namespace; the `anyLanguageMissingFromRedis` short-circuit rebuilds all languages on any single-language miss. *Needs:* compute/compare/persist per `(namespace, language)`; rebuild logic can become per-pair rather than all-or-nothing.
5. **Admin-edit invalidation** — `rebuildTranslationCacheAsync(namespace, language)` (`:185-199`). *Today:* recomputes the **namespace-wide** checksum even though only one language changed (and would race if two languages of the same namespace were edited concurrently). *Needs:* recompute/persist only the edited `(namespace, language)` checksum — which also removes the concurrent-edit staleness.
6. **Read path in the controller** — `VersionController.getVersions()` (`:34-39`). *Needs:* iterate namespaces × languages (the live language set comes from `languageRepository.findAll()`, already used by the service) and assemble the new shape.

Note: seam #2's "all keys pre-seeded or boot throws" invariant and the V1-fold schema convention (conventions Part 12: edit `V1__init_schema.sql` / seed SQL in place pre-launch) together mean the per-language config rows are a seed-file change, not a migration — but they must exist before `VersionChecksumService` runs.

---

## 7. Empty / absent language handling

- **Namespace with zero rows for some language** `[read/inferred]`: `findByTranslationNamespace` returns only existing rows (no language outer-join), so a language with no rows in that namespace **contributes nothing** to the combined hash — it is silently absent. The current all-languages hash cannot distinguish "language X has no strings here" from "language X simply wasn't edited."
- **Namespace with zero rows at all** `[read/inferred]`: the stream over an empty list yields the empty string; `sha256Hex16("")` is a fixed value — first 16 hex of SHA-256 of empty = **`e3b0c44298fc1c14`** (verified by hashing empty input). All empty namespaces share that checksum.
- **Implication for the new shape** (surfaced, not designed): per-(namespace, language) checksums must decide how to represent an absent pair — either emit `e3b0c44298fc1c14` (empty-content hash) for it, or omit the key and let the client treat "missing" as "no content / use fallback." The current code gives no signal either way; the new contract has to choose explicitly. The boot rebuild already iterates `languageRepository.findAll()`, so the full language set is available to enumerate absent pairs if explicit representation is wanted.

---

## 8. Trust boundary verdict (conventions Part 11)

**CLEAN.** `[read]`

- `getVersions()` (`VersionController.java:33`) is parameterless — no `@RequestParam`, `@RequestHeader`, `@RequestBody`, or path variable. Nothing from the request influences the response.
- Translation checksums: namespace list is the hard-coded `TranslationNamespace` enum; config keys are server-side string composition (`"translations.checksum." + ns.name()`); values come from the server-side `configurationCache` (loaded from the DB at boot).
- Catalog checksums: codes from `baseSiteRepository.findActiveBaseSiteCodes()` (fixed JPA query `WHERE b.active = true`), values from the same config store.
- Checksum computation (`VersionChecksumService`) reads only DB entities (`Translation`, `CatalogCategoryAssignment`, etc.); no client input reaches the hashed material, the cache/config key, or the response.

No value used in the computation, the cache/config key, or the response is derived from client-supplied input. Everything is server-side stored data or server config. No violation.

---

## Inferred vs read — summary

- **`[read]`/directly verified by me:** the route and security matcher, the DTO shape, the Cache-Control constant, the full checksum computation (row source, sort, `key|value` join, SHA-256 first-16-hex), the 22-namespace enum and its unfiltered iteration, the four seeded languages and CNR's own distinct seed rows, the moderation-only `cnr→SR` alias and its absence from the checksum path, the configuration-table checksum keys, the dual in-memory + Redis caching split, both invalidation paths, and the empty-string hash value.
- **`[inferred]`:** the exact empty-language / empty-namespace runtime behavior (deduced from reading the stream code + repository method, not observed at runtime); the design options in seams #2/#3/#7 are framings of what the change implies, not statements about current code.

No design proposed — this is the "what is," plus the seams.
