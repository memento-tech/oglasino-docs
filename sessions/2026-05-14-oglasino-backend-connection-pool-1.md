# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-14
**Task:** Backend investigation — connection-pool exhaustion under concurrent load (read-only). New slug `connection-pool`, session `-1`. Output: `.agent/investigation-connection-pool.md`.

## Implemented

- Read-only investigation answering the brief's five questions on HikariCP connection-pool exhaustion. No code changes.
- Established the actual mechanism — a thundering-herd cache miss on `DefaultBaseSiteService.getAllBaseSites()` during cold-cache windows — and **refuted** the brief's working hypothesis that a cache HIT acquires a JDBC connection. Spring's `CacheInterceptor` is the outer advisor (confirmed by the prod stack trace's call direction); on a HIT, `TransactionInterceptor` is never entered and no connection is acquired.
- Identified the prod-vs-stage profile discrepancy: `application-prod.yaml:15` declares `maximum-pool-size: 18` but the log evidence (`total=8`) matches `application-stage.yaml:18` exactly. Flagged for Mastermind — the running profile must be confirmed before any fix is sized.
- Classified `/api/public/translations` as a **victim** of upstream filter-chain pool contention, not the culprit. Its own controller path is DB-free on warm caches. `/api/public/baseSite/overviews` is both a victim (filter chain) and a partial culprit (its own `@Cacheable` method runs JPA repo calls on miss).
- Inventoried connection-holding hot spots, noting in particular that `DefaultProductService.createProduct/updateProduct` holds a JDBC connection across the synchronous `openAIFacade.translateTextWithPrompt` call — a latent issue, not what's exhausting the pool right now.
- Stated explicitly that the burst trigger is not determinable from backend code (no backend fan-out, no retry mechanism, no SSE/WS broadcaster touches translations) — origin is client-side or external.

## Files touched

- `.agent/investigation-connection-pool.md` (created, ~340 lines)
- `.agent/2026-05-14-oglasino-backend-connection-pool-1.md` (this file)
- `.agent/last-session.md` (overwritten — exact copy of this file)

## Tests

- N/A — read-only investigation. No build, no test run.
- `./mvnw spotless:check` and `./mvnw test` would be no-ops for this session since no code changed.

## Cleanup performed

- None needed — read-only investigation.

## Obsoleted by this session

- Nothing. The previous Redis-caching investigation (`.agent/investigation-redis-caching.md` sessions 1 and 2) remains accurate within its own scope; this investigation cites it as orthogonal evidence rather than superseding it.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no `System.out.println`, no `TODO`/`FIXME` added. No source code touched.
- Part 4a (simplicity): N/A — no code changes.
- Part 4b (adjacent observations): confirmed. Three adjacent observations flagged in "For Mastermind" of the investigation file: (a) `CurrentLanguageFilter` silently swallows all exceptions including pool timeouts; (b) the gap between `RedisCacheEvictConfig.evictCachesOnStartup` and `CacheWarmupService.warmup` completion is the cold-cache window the incident exploits; (c) no `@Transactional` in the codebase declares a timeout. Severity flagged inline; none fixed (out of scope, brief is read-only).
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): N/A this session.
- Other parts touched: Part 1 (file naming) — investigation file uses kebab-case `investigation-connection-pool.md`, session files use the Part 5 naming scheme.

## Known gaps / TODOs

- The stage-vs-prod profile question (production logs showing `total=8` but `application-prod.yaml` declaring `maximum-pool-size: 18`) cannot be resolved from inside this repo. Confirmation requires reading `/opt/oglasino/.env` or the running container's effective profile. Flagged in the investigation file's "Brief vs reality" and "For Mastermind" sections.
- Deploy-side reality (whether the readiness probe waits for warmup, whether the running profile is `prod` or `stage`) was inferred from code defaults; not verified against the live container.
- The exact OpenAI client timeout (relevant to the `DefaultProductService` write-path connection-hold) was not chased — out of scope for this investigation, noted as a separate concern.

## For Mastermind

- **Recommended fix scoping**: the load-bearing fix (warmup-gates-readiness + per-key in-flight deduplication on `getAllBaseSites` / `getLanguageByCode` cache misses) is **bug-sized**, single-session. The structural items (rate-limit categorisation of public reference-data endpoints, removal of `@Transactional` from `getAllBaseSites`, separation of `DefaultProductService` write-tx from OpenAI calls, in-memory long-lived reference-data cache) are each bug-sized on their own but make a coherent **feature-sized** package if you want to harden the pattern in one push. My recommendation is to ship the warmup-gating + miss-dedup as the urgent fix and queue the rest as separate briefs.
- **Stage vs prod profile**: this is the single highest-value question to answer before any fix is sized. Two minutes on the box (`docker exec ... env | grep PROFILE`) settles it.
- **Bot mitigation note**: `/api/public/translations` and `/api/public/baseSite/overviews` are unprotected by `RateLimitFilter` (categorisation returns null). This is consistent with the project's "rate-limiting scope: narrow on purpose; no default catch-all" stance — but this investigation surfaces a *concrete abuse cost* (pool exhaustion) that meets the documented threshold for adding a category. Worth a `decisions.md` entry either way (decide to add, decide to refuse, or decide that warmup-gating + miss-dedup makes the rate-limit moot).
- **The 14:35:49 stack trace was diagnostic gold.** The trace showed `CacheInterceptor → TransactionInterceptor → doBegin`, which let me confirm the interceptor order from runtime evidence rather than speculate on Spring's auto-config order. Without it, this investigation would have ended at "Spring usually puts Cache outer when both have LOWEST_PRECEDENCE, but it's technically undefined" — which is much less actionable. If similar future investigations are surfaced, please preserve the relevant stack frame ordering in the brief; that single piece of evidence collapses a wide hypothesis space.
- **Read-only constraint observed**: no code changes, no build, no `psql`. The brief's "no fix" instruction was followed strictly — Section 4 of the investigation file lists candidate fix directions briefly (per the brief's "*can* list candidate fix directions briefly" clause) without proposing implementations.
