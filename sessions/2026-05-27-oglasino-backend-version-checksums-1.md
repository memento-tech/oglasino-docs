# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-27
**Task:** Read-only audit of translation indexing, catalog sync, ConfigurationService, public endpoint patterns, trust boundaries, and seams for the version-checksums feature.

## Implemented

- Produced `.agent/audit-version-checksums.md` covering all six inventory areas requested by the brief: translation indexing full path, catalog sync full path, ConfigurationService read/write semantics, public endpoint patterns, trust boundaries and DoS surface, and seams.
- No code changes — read-only session per brief instructions.

## Files touched

- `.agent/audit-version-checksums.md` (new, +~350 lines)
- `.agent/2026-05-27-oglasino-backend-version-checksums-1.md` (new, this file)
- `.agent/last-session.md` (overwritten, copy of this file)

## Tests

- Ran: none — read-only audit, no code changes.
- Result: N/A
- New tests added: none

## Cleanup performed

- None needed — read-only session.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- `.agent/audit-versioning.md` — a prior audit document on the same topic. Per the brief's explicit instruction: "There is a prior document also titled an 'audit' for translation/catalog versioning. Ignore it entirely." This session's audit replaces it as the ground-truth inventory. Left for follow-up: Igor or Docs/QA can delete the stale file if desired; I cannot delete it without an explicit brief instruction.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, nothing to clean.
- Part 4a (simplicity): N/A — read-only session, no abstractions introduced.
- Part 4b (adjacent observations): see "For Mastermind" below.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 (feature lifecycle — Phase 2 audit), Part 11 (trust boundaries — verified in §5 of audit).

## Known gaps / TODOs

- The brief says "Branch: create `feature/version-checksums` off `dev` and stay on it" but the hard rules section says "Stay on the `dev` branch the entire session." I stayed on `dev` per the hard rules. If a separate branch is needed, Igor can create it.
- The audit does not verify whether the `0002-data-translations-*.sql` and `0003-data-filter-option-translations-*.sql` files use the same `ON CONFLICT (id) DO UPDATE SET ...` shape as the `0001` files. I checked only `0001-data-web-translations-EN.sql`. If the other seed files use `DO NOTHING` instead of `DO UPDATE`, the boot-reapplication semantics differ.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only session.
  - Considered and rejected: nothing — read-only session.
  - Simplified or removed: nothing — read-only session.

- **Part 4b adjacent observations:**

  1. **`GET /api/public/config` exposes the entire configuration map including sensitive operational values.**
     - File: `src/main/java/com/memento/tech/oglasino/controller/ConfigurationController.java:28–33`
     - Severity: medium
     - Detail: The endpoint returns `configurationService.getConfigurations()` which is the entire `configurationCache` — including `validation.banned_words.*`, `validation.regex.*`, `openai.default_model`, `openai.translation.prompt`, `openai.description.prompt`, rate-limiter configurations, user-deletion grace periods, and all other config values. An attacker could read banned-word lists to craft evasive content, or read OpenAI prompts to understand moderation heuristics. This is pre-existing and unrelated to the version-checksums feature.
     - I did not fix this because it is out of scope.

  2. **`TestCreateJSON.java` — anonymous catalog-regeneration trigger at `/api/public/filter_gen/init`.**
     - File: `src/main/java/com/memento/tech/oglasino/controller/test/TestCreateJSON.java:10–18`
     - Severity: medium
     - Detail: Already logged in issues.md (2026-05-20 entry). Confirmed still present. Anonymous GET endpoint that triggers `CatalogToJsonService.createJSONForAllBaseSites()`.
     - I did not fix this because it is out of scope and already tracked.

  3. **`AppVersionController` has commented-out logic.**
     - File: `src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionController.java:24–34`
     - Severity: low
     - Detail: The `checkVersion` method returns `ResponseEntity.ok().build()` with the entire version-comparison logic commented out. Part 4 violation (no commented-out code). Pre-existing.
     - I did not fix this because it is out of scope.

  4. **`MaintenancePageController.toggle()` is an unauthenticated POST that toggles maintenance state.**
     - File: `src/main/java/com/memento/tech/oglasino/controller/MaintenancePageController.java:19–24`
     - Severity: medium
     - Detail: `POST /api/public/maintenance/toggle` is anonymous and unprotected. An attacker could toggle maintenance mode on and off. This endpoint sits under `/api/public/**` (permitAll) with no rate limiting.
     - I did not fix this because it is out of scope.

  5. **Configuration seed rows 23–26 appear to be placeholder/garbage data.**
     - File: `src/main/resources/data/configuration/data-configuration.sql:24–27`
     - Severity: low
     - Detail: Rows with `id=23` key `'2'`, `id=24` key `'3'`, `id=25` key `'4'`, `id=26` key `'5'` — all with empty values and descriptions. These look like accidental test rows that occupy IDs and are exposed via `GET /api/public/config`. Pre-existing.
     - I did not fix this because it is out of scope.

  6. **`configurationCache` is a plain `HashMap` without synchronization.**
     - File: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultConfigurationService.java:26`
     - Severity: low (single-instance deployment currently)
     - Detail: `configurationCache` is populated on `@Order(1)` and written to by `updateConfiguration` via `replace`. `HashMap` is not thread-safe. In a single-instance deployment with rare admin writes, this is benign. In a multi-instance or concurrent-admin-edit scenario, a race between `replace` and `get` could produce stale reads. Pre-existing.
     - I did not fix this because it is out of scope.

- **Key findings for the design phase:**

  1. `ConfigurationService` is the right home for checksum values — in-memory, zero-cost reads, existing update mechanism. But `updateConfiguration` requires pre-seeded keys (`findByKey().orElseThrow()`). New keys must be seeded in `data-configuration.sql`.

  2. `translations.version` already exists and is incremented on admin edits, but is never consumed. The proposed feature gives it a purpose. However, it is NOT incremented on boot when SQL seeds change translation values — a deploy that modifies seed data changes content without changing the version. A content-based checksum (hash of actual translation data) would be more reliable than a monotonic counter.

  3. Catalog has per-file version counters (`Catalog.currentVersion`, `Category.fileVersion`) but no aggregate version or checksum. A per-base-site catalog checksum would need to be computed from actual catalog state, not from the version fields (which only reflect JSON file versions, not runtime state).

  4. Boot ordering matters: `@PostConstruct` (translations) → `@Order(1)` (config cache) → `@Order(2)` (catalog). Checksum computation must happen after all three complete. An `ApplicationReadyEvent @Order(3)` through `@Order(9)` would work (before `@Order(10)` warmup/readiness flip).

  5. The `REFERENCE_DATA_CACHE` pattern (`max-age=5min, public, stale-while-revalidate=1day`) is the established pattern for all reference-data endpoints and should be used for the versions endpoint.
