# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Task:** Audit mobile's backend call patterns — redundant calls, unbatched calls, missing caches, patterns web has already eliminated.

## Implemented

- Read-only audit of all backend call patterns in the mobile app
- Produced `.agent/audit-expo-readiness-backend-calls-reduction.md` covering all 7 sections required by the brief plus a severity-ranked summary
- Confirmed all 4 findings from other audits: (a) i18n 21-call burst with caching commented out, (b) double idToken in firebase-sync, (c) language-change multi-page parallel refetch, (d) favorites toggle → list refresh
- Mapped the full app-init call sequence (25–28 calls depending on auth state)
- Audited 5 high-traffic routes for per-route call patterns
- Inventoried all caching surfaces (catalogStorage unused, useCatalogStore unused, userCache limited to chat scope, viewTokenStore well-implemented)
- Identified the 5-second maintenance poll as the highest-volume ongoing waste (720 calls/hour)
- Identified the auth double-sync on login as the mobile equivalent of web's fixed `UseTokenRefresh` bug

## Files touched

- `.agent/audit-expo-readiness-backend-calls-reduction.md` (new, +~280 lines)

## Tests

- N/A — read-only audit, no code changes

## Cleanup performed

- None needed (read-only audit)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing

## Conventions check

- Part 4 (cleanliness): confirmed (no code changes)
- Part 4a (simplicity): N/A (read-only audit)
- Part 4b (adjacent observations): two cross-audit findings flagged in "For Mastermind" (dead catalogStorage/useCatalogStore code; stale `allowPreferenceCookies` in user settings)
- Part 11 (trust boundaries): confirmed — no trust-boundary concerns in the call patterns audited; the `idToken` body duplication is redundant but not a violation

## Known gaps / TODOs

- Cannot verify whether backend reads `idToken` from the body, the header, or both — mobile can only confirm what it sends
- The exact translation count is 21 (not 20 as the brief states) — `TranslationNamespace` enum has 21 members

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Top 5 findings by impact (ranked):**
  1. **H1 — Translation caching disabled (21 calls/init).** The caching infrastructure is half-built (AsyncStorage helpers exist, commented code shows the design). Missing piece: a backend version endpoint. This is the single biggest win.
  2. **H2 — 5-second maintenance poll (720 calls/hour).** Recommend 30–60s, or push-based notification for maintenance entry.
  3. **H3 — App version check on every pathname change.** `AppVersionConfigInit.tsx` checks version on every navigation. Once per foreground resume is sufficient.
  4. **H4 — Auth double-sync on login.** `buildUserSession` + `initAuthListener` both call `syncUserToBackend`. Web fixed the equivalent bug in the 2026-05-21 `UseTokenRefresh` single-flight session.
  5. **H5 — Double `getIdToken()` in `syncUserToBackend`.** The explicit call + explicit header is redundant with the interceptor.

- **Cross-audit observations (not investigated, per brief scope):**
  - `catalogStorage.ts` and `useCatalogStore.ts` are dead code — zero callers. Relevant to the general audit.
  - User settings screen still references `allowPreferenceCookies` as a server-persisted field. Relevant to the consent-mode audit.

- **No config-file drafts.** This audit produces input for a future spec, not config-file changes.
