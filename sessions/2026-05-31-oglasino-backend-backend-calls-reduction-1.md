# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-31
**Task:** Read-only audit: (Q1) does `/auth/firebase-sync` use `idToken` from the request body, and (Q2) how is "active base sites" resolved per request, and is it cached?

## Implemented

- Read-only audit only — no source changes. Produced `.agent/audit-backend-calls-reduction-backend.md` with two `file:line`-evidenced answers.
- **Q1:** The body `idToken` is dead. `/api/auth/firebase-sync` (`AuthController.java:52-54`) binds the body to `LoginRequest`, which has only `displayName` and **no `idToken` field** (`LoginRequest.java:6-18`). The token comes exclusively from the `Authorization: Bearer` header via `resolveFirebaseToken` (`DefaultFirebaseAuthService.java:152-165`). Safe for mobile to drop.
- **Q2:** Active-base-site resolution runs in `BaseSiteFilter` for every `/api/**` request, all clients (`BaseSiteFilter.java:24-32`). The per-code `BaseSiteDTO` is Redis-cached with **no TTL** (evict-driven, warmup-populated — `RedisConfig.java:69-77`), but `findActiveBaseSiteCodes()` is **uncached** and hits Postgres on every resolution (`BaseSiteRepository.java:32-33` via `DefaultBaseSiteService.java:58-63`). No mobile-only endpoint — the `/api/public/baseSite/*` fetch endpoints are shared (`BaseSiteController.java`) and carry a 5-min HTTP `Cache-Control` (`ReferenceDataCacheControl.java:8-11`).

## Files touched

- `.agent/audit-backend-calls-reduction-backend.md` (new, audit output)
- `.agent/2026-05-31-oglasino-backend-backend-calls-reduction-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten, copy of this summary)
- No source files touched.

## Tests

- Ran: none. Read-only audit; no code changed, so no build/test gate applies.
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed (read-only audit, no code written).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change. (The 2026-05-31 "Mobile: active base sites fetched on every request, uncached" entry already exists and anticipates this Phase-2 audit; this audit supplies the backend-side `file:line` confirmation it asked for. No new entry authored by an engineer agent — Docs/QA is the sole writer; findings live in the audit file for Mastermind to route.)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code written, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundaries) — Q1 confirms the firebase-sync token is taken from the verified `Authorization` header / server-verified Firebase token, never from a client body field, which is consistent with Part 11. No violation.

## Known gaps / TODOs

- none. Both audit questions answered. No recommendations or fixes per the brief's "just the answers" instruction.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no abstractions, config, or patterns introduced.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Brief vs reality (minor, non-blocking — does not change the answer):** The brief's Q1 framing says "The request body has an `idToken` field." On the bound DTO (`LoginRequest`, `LoginRequest.java:6-18`) there is **no `idToken` field at all** — only `displayName`. So the body `idToken` mobile sends is an unbound/ignored JSON property, not a declared-but-unused field. This strengthens the answer (it is dead, and was never wired), so I did not stop — I recorded it plainly in the audit. Flagging here only so the framing is accurate when Mastermind reaches the seam-analysis phase.
- **Adjacent observation (Part 4b), low/medium severity — out of scope, not fixed:** On the per-request resolution path, `BaseSiteFilter` → `getBaseSiteForCode` → `getAllBaseSites()` rebuilds the **entire** active-base-site list on every `/api/**` request (one uncached `findActiveBaseSiteCodes` Postgres query + one Redis read per active code), then `filter().findFirst()`s down to the single header-matched code (`DefaultBaseSiteService.java:58-63`, `DefaultBaseSiteCacheService.java:35-40`). For a single-code lookup this is more work than needed — the codes list is uncached and the full list is reconstructed to discard all but one entry. Severity medium: it is a fixed per-request cost on every API call for every client (web + mobile), not just the mobile over-fetch the issues.md entry targets. I did not change it — out of audit scope; logging for the eventual chat I (backend-calls optimization) to weigh against the existing `issues.md` 2026-05-31 entry.
- No config-file edits required or drafted this session.
