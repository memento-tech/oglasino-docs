# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Task:** Backend investigation brief — Redis caching bypass. Read-only investigation. No code changes.

## Implemented

- Read-only investigation of all Redis-backed caches in the repo. Inventoried the seven caches, traced every caller of every `@Cacheable` method, verified key alignment, audited eviction triggers, and confirmed Redis wiring.
- Wrote findings to `.agent/investigation-redis-caching.md` with per-cache root cause statements as the brief requested.
- Headline finding: the brief's prime suspect — Spring proxy bypass on the `BaseSite` caches — is **not what's happening**. The `BaseSite` caches go through the proxy correctly via the deliberate `BaseSiteCacheService → BaseSiteService` cross-bean delegation. The real bypass is on `redisBaseCurrency` (`DefaultBaseCurrencyService.convertToBaseCurrency()` self-invokes `getBaseCurrency()`, hitting DB on every price-filtered product search). The likely dominant cause of observed DB load is the `redisUserAuth` 1-minute TTL: every active user re-fetches `findAuthDataByFirebaseUid` at least once per minute by design.

## Files touched

- `.agent/investigation-redis-caching.md` (+201 / -0) — investigation report (the brief's required deliverable)
- `.agent/2026-05-14-oglasino-backend-redis-caching-1.md` (+85 / -0) — this session summary
- `.agent/last-session.md` (+85 / -0) — duplicate of the summary, per Part 5

No source code modified. No tests modified. Read-only investigation per the brief.

## Tests

- Not run. Brief is read-only investigation, no code changes.

## Cleanup performed

- None needed — read-only investigation.

## Obsoleted by this session

- Nothing.

## Known gaps / TODOs

- Step 6 (concrete signal via local Hibernate SQL logging) was **deliberately skipped**. The brief flagged it as optional and only if cleanly doable read-only; doing it would have required a config edit I did not want to commit, or a runtime override I cannot safely apply against Igor's local stack. Step 2's call-tracing was sufficient to issue verdicts. Recipe for Igor to run himself is included at the end of Step 6 in the report.

## For Mastermind

1. **Brief vs reality discrepancies (3, all in the report's "Brief vs reality" section):**
   - 7 caches exist, brief said 5–6.
   - `BaseSite` cache names are `redisBaseSites` / `redisBaseSiteOverviews` (brief tentatively named them `BaseSiteData` / `BaseSiteOverview`).
   - The proxy-bypass framing for `BaseSite` is wrong — those caches work. There IS a real bypass, but it's on `redisBaseCurrency`.
2. **Strongest hypothesis for the observed DB load:** the `redisUserAuth` 1-minute TTL. This is a design choice (per the comment in `RedisConfig.java:41-46`), not a bug, but it means every active authenticated user re-fetches `findAuthDataByFirebaseUid` from Postgres at least once per minute regardless of how many requests they make. If steady-state DB load on the auth path is the smoke Igor saw, this is the lever. Bumping TTL to 30 min looks safe given how comprehensively `saveUser` evicts.
3. **Real proxy bypass (smaller impact, real):** `DefaultBaseCurrencyService.convertToBaseCurrency()` (`:91`) self-invokes `getBaseCurrency()`. Triggered on every price-range-filtered product search via `PriceQueryGenerator` — 2 redundant `currencyRepository.findByBaseCurrencyTrue()` queries per filtered search. Three reasonable fix shapes proposed in the report.
4. **Cold caches after every deploy.** `RedisCacheEvictConfig.evictCachesOnStartup` clears all 7 caches on app boot. Warmup repopulates the global ones; per-user caches (`redisUserInfo`, `redisUserAuth`) stay cold and pay one DB hit per user on first request after each deploy. Worth knowing about, not necessarily worth fixing.
5. **Adjacent observation:** `redisUserInfo` mixes Long-id keys and String-firebaseUid keys in one cache namespace. No collision risk in practice, but mildly surprising — a future engineer might miss the dual-key invalidation in `evictUserInfoCache`. The existing `@Caching(evict = {...})` block does it correctly. Low severity, no fix recommended.
6. **Adjacent observation:** the `application-dev.yaml` block for `spring.cache.redis.time-to-live: 0` (line 50) is misleading: the inline comment says "1 hour (ms)" but the value is 0 (Spring Boot interprets as no expiration). Doesn't matter because the manual `RedisCacheManager` bean overrides this, but the comment is wrong. Low severity, cosmetic — noting in case Mastermind wants to either remove the unused config or fix the comment in a future chore.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, nothing to clean.
- Part 4a (simplicity): N/A — no code authored.
- Part 4b (adjacent observations): two adjacent observations flagged in "For Mastermind" (#5, #6), as required.
- Part 5 (session summary template): confirmed — wrote to both the named file and `.agent/last-session.md`. `<n>` = 1 (no prior `*-redis-caching-*.md` files).
- Part 6 (translations): N/A this session — no translation work.
- Part 7 (error contract): N/A.
- Part 11 (trust boundaries): N/A — investigation, not a request-handling change.
