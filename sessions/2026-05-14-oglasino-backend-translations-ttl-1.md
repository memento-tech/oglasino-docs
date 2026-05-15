# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Task:** Read-only investigation: verify Igor's position that the translations Redis entries should have no TTL — pin down where the TTL is set, confirm DB-write paths all trigger Redis reindex, classify reindex as full-replace vs additive, describe cold-miss cost.

## Implemented

- Investigation only; no code changes. Output: `.agent/investigation-translations-ttl.md`.
- Surfaced a brief-vs-reality finding: the brief assumes a 24 h TTL on translations keys; the code shows no TTL at all on any translation key. Investigation answers all four scoped questions with that finding made explicit.
- Mapped every Redis-write site to `translations:<NS>:<lang>` keys (one site: `DefaultTranslationService.java:95`, two-arg `set(key, value)`, no `Duration`).
- Mapped every DB-write site to `oglasino_translation` (two sites: `spring.sql.init` SQL seed at boot, `DefaultTranslationService.updateTranslation` admin path — both trigger Redis reindex).
- Classified reindex semantics: full-replace at the `(namespace, lang)` blob level for both admin and boot paths.
- Confirmed the per-pair `intern()`-synchronized double-checked-lock in `getOrRebuild` (`:231-249`) serializes refills per `(namespace, lang)`.

## Files touched

- .agent/investigation-translations-ttl.md (+285 / -0, new file)
- .agent/2026-05-14-oglasino-backend-translations-ttl-1.md (this file, new)
- .agent/last-session.md (overwritten with the content of this file)

No `src/` files modified.

## Tests

- Not applicable. Read-only investigation, no code changes, no test changes.

## Cleanup performed

- None needed.

## Obsoleted by this session

- The planned follow-up "remove translations TTL" fix brief is obsoleted before it was drafted — there is no TTL to remove. Cannot delete because the brief is a planning artifact in Mastermind's chat, not a committed file; flagging to Mastermind so the brief is not drafted.

## Known gaps / TODOs

- None. The four scoped questions are answered. Adjacent observations are listed in "For Mastermind" in the report.

## Conventions check

- Part 4 (cleanliness): confirmed — read-only session, no code touched, no debug logging added, no TODO/FIXME left, no new docs created in `<repo>/docs/`.
- Part 4a (simplicity) / Part 4b (adjacent observations): confirmed — flagged four adjacent low-severity observations in the report's "For Mastermind" section (no-`@Transactional`-on-updateTranslation, SQL-seed-clobbers-admin-edits, empty-bucket-stale-blob, two-parallel-Redis-patterns). All marked out-of-scope per Part 4b's instructions.
- Part 6 (translations): N/A this session — no translation keys added or modified; the investigation discusses the translation cache machinery but does not touch the `data/translations/*.sql` seeds.
- Other parts touched: Part 11 (trust boundaries) — N/A this session; investigation deals with caching, not request DTOs. Part 7 (error contract) — N/A this session.
- Brief-challenging rule per CLAUDE.md: surfaced the "no TTL exists" finding as a `Brief vs reality` section at the top of the report, in calibrated form (real factual discrepancy with the brief's premise; not cosmetic).

## For Mastermind

- **The planned follow-up "remove translations TTL" fix brief should not be drafted.** No TTL exists on the translations Redis keys today. The state Igor wanted is already the state on disk. The lever for the actual cold-cache-exhausts-pool failure mode is `redisBaseSites` (24 h TTL at `RedisConfig.java:49`) and the warmup-gating brief that's running in parallel — not translations.
- **Adjacent observations flagged (low-severity, out of scope, not fixed):**
  - `DefaultTranslationService.updateTranslation` (`:205-221`) has no `@Transactional` but relies on OSIV to flush the dirty entity at the next tx boundary (inside `updateTranslationsVersion → configurationService.updateConfiguration`). Works today; silently breaks if OSIV is disabled. Worth either an explicit `@Transactional` or a `// relies on OSIV` comment, but separate brief. Severity: low.
  - The SQL seed `ON CONFLICT (id) DO UPDATE SET translation_value = EXCLUDED.translation_value` clobbers admin edits on every boot. If admin-edited values are supposed to survive redeploys, this is a gap. If the seed is source of truth and the admin UI is "preview only," it's correct — but the admin UI then misleads. Severity: low–medium, depends on intent. Worth deciding.
  - `indexNamespaceLang` early-returns when `translations.isEmpty()` (`:80-83`) without clearing the existing Redis blob. Not reachable in practice today (seeds keep buckets populated, no delete-translations endpoint exists). If a delete path is ever added, stale blobs become possible. Severity: low.
  - Two parallel patterns for cached reference data: `StringRedisTemplate`-direct (translations) vs `@Cacheable` through `RedisCacheManager` (base sites, languages, etc.). Not wrong, but means any cross-cutting Redis-cache improvement (per-key in-flight dedup, observability, eviction) needs two implementations. Severity: low.
- **One deliberate omission:** I did not run a live `redis-cli TTL translations:ERRORS:en` against the prod Redis to double-check. The code-level evidence is unambiguous (`SET key value` with no `Duration`, no `expire(...)` call, no TTL config) — but if you want a belt-and-braces confirmation, that one-line `TTL` check should return `-1`.
