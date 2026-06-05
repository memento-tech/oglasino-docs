# Audit — Dead / unused code inventory (oglasino-backend)

**Repo:** oglasino-backend · **Branch:** dev · **Mode:** READ-ONLY (inventory, no fixes)
**Date:** 2026-06-05
**Scope:** `src/main/java` (610 files) + `src/main/resources/data/configuration/*.sql`

This is an inventory. Every candidate carries `file:line`, what it is, and the grep/tool
evidence that establishes it is unused. Items are bucketed into exactly one tier:

- **TIER 1** — safe to delete: declared, zero readers/callers anywhere in `src`, not a
  published/wire API, not reflectively/dynamically referenced.
- **TIER 2** — looks dead, has a catch (catch named per item).
- **TIER 3** — intentionally kept; recognized and EXCLUDED from removal.

Per the brief, the **B7 `emailVerifiedExternal` column** and the **email-edit gate**
(`DefaultUserFacade.java:170`) are out of scope — both are open tracked issues, not dead code.
They were not touched.

---

## How this was established (method + mechanical coverage)

Two mechanical passes gave authoritative negatives for whole categories; the rest was a
grep fan-out, each candidate re-verified by hand before listing.

| Category | Tool / method | Result |
|---|---|---|
| **Unused imports** | `./mvnw spotless:check` (spotless config has `<removeUnusedImports/>`, `pom.xml:268`); exit 0 | **ZERO repo-wide.** A non-clean import would fail the check. |
| **Uncalled `private` methods, unused `private` fields, dead local stores** | `./mvnw compile spotbugs:spotbugs` (effort `Max`, 665 classes analyzed); report `target/spotbugsXml.xml` | **ZERO.** Only findings were 12× `VA_FORMAT_STRING_USES_NEWLINE` + 2× `CT_CONSTRUCTOR_THROW` (style/bad-practice, not dead code). No `UPM_UNCALLED_PRIVATE_METHOD`, `UUF_UNUSED_FIELD`, `URF_UNREAD_FIELD`, or `DLS_DEAD_LOCAL_STORE`. |
| Repository `@Query`/derived methods | name-grep across `src`, every declared (non-inherited) method | see TIER 1 |
| Unreferenced beans / whole classes / enums + enum constants | name-grep, excluding own file; entry-points filtered | see tiers below |
| `public`/package methods + `public` constants (SpotBugs only sees `private`) | name-grep, overrides/handlers/callbacks filtered | one dead method, no dead constants |
| Config keys (both directions) | seed-SQL keys ↔ `getConfig`/`get*Config` call sites, incl. dynamic key composition | see tiers + §Inverse |

Grep convention below: a Spring-Data repo method and a config key are both invoked **by name**
(method name / key string), so a name-grep returning only the declaration is a valid zero-reader proof.

---

## TIER 1 — Safe to delete (zero readers, not a contract, not dynamic)

### Repository methods (declared, never called)

1. **`ConfigurationRepository.getAll()`** — `repository/ConfigurationRepository.java:17`
   `List<ConfigurationDTO> getAll();` (`@Query` JPQL constructor projection, lines 14–17).
   Evidence: `grep -rn "getAll(" src --include='*.java'` → **1 match, the declaration only**.
   (Other `getAll*` names — `getAllValues`, `getAllBaseSites`, `getAllLabelKeys`, … — are distinct methods.)

2. **`UserRepository.findAllNonNullProfileImageKeys()`** — `repository/UserRepository.java:37` (`@Query` 34–36).
   `List<String> findAllNonNullProfileImageKeys();`
   Evidence: `grep -rn "findAllNonNullProfileImageKeys" src` → **1 match, the declaration only**.
   Note: its Javadoc claims "Used by `ProductImagesRemovalJob`" — **false** (drift; see §Adjacent).

3. **`UserRepository.getUserBaseSiteId(Long id)`** — `repository/UserRepository.java:68` (`@Query` :67).
   `Optional<Long> getUserBaseSiteId(@Param("id") Long id);`
   Evidence: `grep -rn "getUserBaseSiteId" src` → **1 match, the declaration only**.

4. **`ReviewTranslationRepository.findByReviewAndLanguage(Review, Language)`** — `repository/ReviewTranslationRepository.java:14` (derived query).
   Evidence: `grep -rn "findByReviewAndLanguage" src` → 4 matches: the declaration + **three comment/Javadoc-only** mentions (`ReviewTranslationRepository.java:18` `{@link #...}`, `converter/ReviewTranslationMappingContext.java:11`, `admin/facade/impl/DefaultAdminReviewFacade.java:41`). No invocation. The batched all-rows-in-one-query path (decisions.md 2026-…, "single-query-per-page") superseded this per-row lookup.

### Unreferenced Spring beans (injected nowhere)

5. **`ProductAuditService`** (`service/ProductAuditService.java:5`) + impl **`DefaultProductAuditService`** (`service/impl/DefaultProductAuditService.java:20`).
   Evidence: `grep -rn "\bProductAuditService\b" src` → only the interface decl + the impl's own `import`/`implements`. `grep -rn "DefaultProductAuditService" src` → only its own file (decl + logger). No injector, no test.

6. **`ProductFilterService`** (`service/ProductFilterService.java:8`) + impl **`DefaultProductFilterService`** (`service/impl/DefaultProductFilterService.java:15`).
   Evidence: `grep -rn "\bProductFilterService\b" src` → only interface decl + impl `import`/`implements`. No injector. **Distinct** from the live `ReferenceProductFilterService`/`DefaultReferenceProductFilterService` — do not confuse.

7. **`ProductsService`** (admin) (`admin/service/ProductsService.java:6`) + impl **`DefaultProductsService`** (`admin/service/impl/DefaultProductsService.java:11`).
   Evidence: `grep -rn "\bProductsService\b" src` → only interface decl + impl `import`/`implements`. No injector, no test.

### Unreferenced whole classes

8. **`ProductValidationUtil`** — `util/ProductValidationUtil.java:8` (147 lines; `BANNED_WORDS`, `containsBannedWords`, `isKeywordStuffing`, …).
   Evidence: `grep -rln "\bProductValidationUtil\b" src` → **only its own file**. Refactor leftover — this logic now lives in `moderation/analyzer/` (`BannedWordsAnalyzer` and siblings). Its `public` `BANNED_WORDS` constant dies with it (no external reader).

9. **`SnowflakeUtil`** — `util/SnowflakeUtil.java:6` (71 lines; `nextId()` ID generator).
   Evidence: `grep -rln "\bSnowflakeUtil\b" src` → **only its own file**. No `nextId(` caller elsewhere.

### Unreferenced method on a live class

10. **`OpenAIResponseMapper.fromJson(String)`** — `openai/service/impl/OpenAIResponseMapper.java:11`.
    `public static OpenAIResponseDTO fromJson(String json)`.
    Evidence: `grep -rn "fromJson" src` → **1 match, the declaration only** (also nil in `.yaml/.yml/.properties/.xml`).
    The class is **live** — its sibling `extractFirstText` is called at `DefaultOpenAIFacade.java:106`; only `fromJson` is dead. The facade parses the raw JSON itself rather than via this mapper.

---

## TIER 2 — Looks dead, has a catch

1. **`IDFileProcessor`** — `util/IDFileProcessor.java:10` (134 lines).
   **Catch:** standalone **dev-only `static void main(String[])`** runner — a one-off SQL label-key
   fixup tool, no Spring annotations, wired into nothing. `grep -rln "\bIDFileProcessor\b" src`
   (excl. own file) → empty. Safe to delete for the *running app*; deletion loses a runnable
   maintenance tool. Lives in `src/main` rather than a scripts dir.

2. **`LabelKeyConflictDetector`** — `catalog/util/LabelKeyConflictDetector.java:13` (88 lines).
   **Catch:** same as above — dev-only `main()` label-conflict checker over the seed SQL, no
   annotations, zero refs. Tool, not app code.

3. **`NotificationCategoryId.PRODUCT_EXPIRATION`** — `notifications/enums/NotificationCategoryId.java:6`.
   **Catch:** never produced backend-side (the only categories ever `set` are `NAVIGATION`, `INFO`,
   `SAVED_PRODUCT`, `MESSAGE` — see `setNotificationCategoryId` call sites), but the value is part of
   the **cross-repo notification wire contract**: `oglasino-web` already implements client handlers
   for `PRODUCT_EXPIRATION`/`PRODUCT_EXPIRED` (per `issues.md` 2026-06-02, `notificationActions.test.ts`).
   This reads as a **planned-but-unwired** product-expiration notification (there is a `ProductRemovalJob`),
   not an accident. Do not delete on backend-grep evidence alone.

4. **`NotificationCategoryId.PRODUCT_EXPIRED`** — `notifications/enums/NotificationCategoryId.java:7`.
   **Catch:** identical to #3.

5. **`NotificationType.DANGER`** — `notifications/enums/NotificationType.java:6`.
   **Catch:** never `set` by any backend producer (only `NORMAL`/`SUCCESS`/`WARNING` are emitted —
   see `NotificationType.` call sites). `grep -rn "\bDANGER\b" src` → enum decl only. But
   `NotificationType` is an outbound **wire enum** clients may style on; this is an unused value in a
   published value-space, not local dead code.

6. **`SuggestionType.FEATURE_BUG_SUGGESTION`** — `admin/entity/SuggestionType.java:5`.
   **Catch:** the enum is a **client-deserializable** wire field (`@NotNull SuggestionType` on
   `SuggestionRequestDTO:13`; also a filter field on `SuggestionsFilterRequestDTO`). A client can send
   `FEATURE_BUG_SUGGESTION`. However `DefaultSuggestionService.saveSuggestion` **ignores the incoming
   value and hardcodes** `SuggestionType.CATEGORY_SUGGESTION` (`:25`), so it can never be persisted via
   this path. Reachable on the wire; functionally never stored. Product decision, not a silent delete
   (see the related latent bug in §Adjacent).

7. **Config key `redis.rate.limiter.ttl`** (value `600000`) — seed `data/configuration/data-configuration.sql:6` (id 4).
   **Catch:** config keys live in **seed SQL**, not Java. No code reader:
   `grep -rn "redis.rate.limiter.ttl" src` → only the seed row. Distinct from the live
   `redis.product.view.{delta.ttl,owner.ttl,dedup.window.ms}` keys (all have readers). No "generic
   redis rate limiter" reader exists. Removing it is a seed-data edit; no runtime reader depends on it.

---

## TIER 3 — Intentionally kept (recognized, NOT proposed for removal)

1. **Config key `user.deletion.report.window.days`** (value `7`) — seed `data/configuration/data-configuration.sql:87` (id 50).
   No code reader (`grep -rn "user.deletion.report.window.days" src` → seed row only; the code reads
   `user.deletion.grace.period.days` instead). **Kept by an explicit in-seed comment:** *"Today
   identical to user.deletion.grace.period.days; kept as a separate knob so the two timers can diverge
   later."* Deliberate staging, not dead. **Excluded.**

2. **`NotificationCategoryId.MESSAGE`** — `notifications/enums/NotificationCategoryId.java:2`.
   `decisions.md` (2026-…, line ~1686) records this as *"confirmed zero callers and intentionally left
   as a stub for future push-notification work."* **That note is now STALE — `MESSAGE` is LIVE:** it is
   set at `notifications/service/impl/DefaultMessageNotificationService.java:132`. **Not dead, not a
   stub anymore.** Listed only to record that the old TIER-3 marker was seen and is now superseded
   (see §Config-file impact / For Mastermind — `decisions.md` should be reconciled).

---

## Inverse check — code readers for keys not in seed (would be a bug, not dead code)

**None.** The only literal that wasn't in `data/configuration/` is `admin.info.enabled`
(`listeners/AdminInfoNotificationListener.java:47`), and it **is** seeded — per-environment — in
`data/admin/data-admin-{prod,stage,dev}.sql` (id 182). Not a missing-seed bug.

Checksum keys (`catalog.checksum.*`, `translations.checksum.<NS>.<lang>`) and per-language
moderation/threshold keys are **read via dynamically-composed key names** (e.g.
`"catalog.checksum." + code` at `VersionChecksumService.java:105`; `"translations.checksum." +
namespace.name() + …` at `:143`/`:181`; moderation `validation.*.{en,sr,ru}` built in
`ContentValidationConfig.java:87-94`). They are **not** dead despite not appearing as string literals.
`openai.description.prompt` / `openai.translation.prompt` likewise reach `getConfig` via
`getFormattedPrompt(...)` (`DefaultOpenAIFacade.java:55,94 → :120`) — not dead.

---

## Adjacent observations (NOT dead code — flagged, no fix per read-only brief)

- **Latent bug — `DefaultSuggestionService.saveSuggestion` discards its argument.** It takes
  `SuggestionType suggestionType` but always stores `CATEGORY_SUGGESTION` (`admin/service/impl/DefaultSuggestionService.java:21,25`).
  Consequence: the `suggestionType` parameter is effectively dead (always overwritten) and
  `FEATURE_BUG_SUGGESTION` (TIER 2 #6) can never be persisted. Either honor the param or drop the enum
  value + wire field. This is a correctness/contract issue, not dead code.
- **Javadoc drift.** `UserRepository.findAllNonNullProfileImageKeys()` Javadoc says "Used by
  `ProductImagesRemovalJob`" — it is not used anywhere (TIER 1 #2).
- **Over-broad visibility (live, not dead).** SpotBugs/grep surfaced members used only within their
  own file that could be tightened (no removal warranted): `DefaultScheduledRedisFlushService.flushAll`
  (`public`, only caller is the same class's `@Scheduled periodicFlush`), `CatalogToJsonService.updateFilterData`
  (`public`, internal-only), and ~16 in-file-only `static final` constants
  (`DefaultMessagingCleanupService`, `ContentValidationConfig`, `LanguageDetectorAnalyzer`,
  `DefaultOpenAIService`, the analyzer `CONFIG_KEY`s, `ALIAS_NAME`). Style, not dead code.
- **Already tracked, not re-filed:** the config default-`0` getter footgun and unmigrated callers are
  logged in `issues.md` (2026-06-04 ES-performance carry-forward); the `redis.product.view.*` keys read
  by those getters are live readers, so they are NOT dead keys.

---

## Coverage caveats

- Grep-based reference detection cannot see invocation purely by reflection/SpEL string name. The one
  dynamic-lookup surface in the repo (`applicationContext.getBean(...)`) does **not** exist
  (`grep -rn "getBean(" src/main/java` → empty), so name-based analysis is complete for beans. Config
  keys and Spring-Data methods are name-invoked and fully covered. Enum constants reachable via
  `values()`/`valueOf()` or inbound wire/DB fields were explicitly checked and excluded from the dead
  list (TranslationNamespace, AdminCacheDescriptor, ReportOption, OwnerReviewTab, ReasoningEffort — all
  runtime-reachable).
- Response-DTO fields are write-only to the wire by design (Jackson-serialized, SpotBugs-excluded per
  `spotbugs-exclude.xml`); no "emitted-but-also-never-written" backend field surfaced (the historical
  `shown` case was already confirmed never-written backend-side — `issues.md` 2026-06-02, closed).

---

## Tally

- **TIER 1 (safe to delete):** 10 items — 4 repo methods, 3 service-bean pairs (interface+impl), 2 whole
  util classes, 1 method (`fromJson`).
- **TIER 2 (dead, has a catch):** 7 items — 2 dev-`main()` runners, 3 unproduced wire-enum values, 1
  wire-deserializable enum value, 1 seed-only config key.
- **TIER 3 (intentionally kept, excluded):** 2 items — 1 staged config key, 1 stale-marker live enum value.
- **Mechanical negatives:** unused imports = 0 (spotless); uncalled private methods / unused private
  fields / dead locals = 0 (spotbugs Max, 665 classes).
- **Inverse (reader-without-seed) bugs:** 0.
