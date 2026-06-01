# Backend Calls Reduction (Mobile)

**Slug:** `backend-calls-reduction-mobile`
**Status:** shipped (code) / verifying (mobile on-device deferred to Ψ)
**Repos:** oglasino-expo (new-expo-dev), oglasino-backend (dev)

## 1. Scope

Residual backend-call optimizations after the foundation chats, boot-redesign, and version-checksums absorbed most of the original §I scope. Four fixes:

- **C1** (mobile) — stop sending dead `idToken` in the `/auth/firebase-sync` body; drop the redundant explicit `getIdToken()`.
- **C2** (mobile) — cache `getUserForId` results.
- **C4** (mobile) — skip the per-login Firestore read in `ensureUserInFirestore` for returning users.
- **C5** (backend) — cache `findActiveBaseSiteCodes` so it stops hitting Postgres on every `/api` request.

Dropped after re-audit: per-pathname version check (gone — boot-redesign deleted `AppVersionConfigInit.tsx`; version check now fires once per boot, Gate 2).

## 2. Background

Re-audited on `new-expo-dev` (mobile) and `dev` (backend), 2026-05-31. Audits at `sessions/audit-backend-calls-reduction-mobile.md` and `sessions/audit-backend-calls-reduction-backend.md`.

X5 answered: `/auth/firebase-sync` does not read `idToken` from the body — `LoginRequest` has no such field; the token comes only from the `Authorization` header.

The "active base sites fetched per request" issue (issues.md 2026-05-31) was two things. Mobile: not reproduced on `new-expo-dev` — the interceptor reads a cached bootStore slot and only sets the `X-Base-Site` header, no network call. Backend: the real per-request cost was an uncached `findActiveBaseSiteCodes` Postgres query in `BaseSiteFilter`, paid by all clients. That is C5.

## 3. C1 — Drop dead idToken (mobile)

`syncUserToBackend` (`authService.ts`) no longer sends `idToken` in the `/auth/firebase-sync` body or calls `getIdToken()` explicitly. Body is now `{ profileImageKey, providerId }`. The token is attached by the axios request interceptor. Shipped in session `oglasino-expo-backend-calls-reduction-2`.

## 4. C2 — Cache getUserForId (mobile)

`getUserForId` (`userService.ts`) was uncached, re-fetched per screen revisit (product detail, user profile). Now an id-keyed read-through in the existing `userCache.ts` (alongside the firebaseUid-keyed chat cache), cleared on logout via the existing `clearUserCache` call. Shipped in session `-3`.

## 5. C4 — Skip per-login Firestore read (mobile)

`ensureUserInFirestore` (`authService.ts`) already skipped the doc-creation write for returning users; now also skips the `getDoc` read via a persisted per-uid AsyncStorage flag (`firestoreEnsuredStorage.ts`). The flag is set only on a confirmed create (`created !== null`), so a failed create does not permanently skip. First-ever sign-in unchanged. Shipped in session `-4`.

## 6. C5 — Cache active base site codes (backend)

`findActiveBaseSiteCodes` ran uncached on every `/api` request via `BaseSiteFilter` → `getAllBaseSites`, for all clients. Now a no-TTL Redis cache `redisActiveBaseSiteCodes` (new `getActiveBaseSiteCodes()` `@Cacheable` via `@Lazy self`), warmed by `CacheWarmupService`, evictable via `CacheAdminController`. Per-request Postgres hit gone. Shipped in session `oglasino-backend-backend-calls-reduction-2`.

Deliberately left uncached: `VersionController` and `VersionChecksumService` read the codes directly — they build version/translation checksums and must read fresh DB state. Correct, not a gap.

Out of scope (stays with `connection-pool-hardening`): removing `@Transactional` from `getAllBaseSites`, per-key single-flight.

## 7. Definition of done

- C1: body `idToken` and explicit `getIdToken()` removed; auth works via interceptor header.
- C2: `getUserForId` cached id-keyed; cleared on logout; revisits don't re-fetch.
- C4: returning users skip the per-login Firestore read; first sign-in unchanged.
- C5: `findActiveBaseSiteCodes` cached, no TTL, warmup-populated, evictable.
- Mobile `tsc --noEmit` + lint clean; backend `spotless` + tests green (710).
- Mobile on-device verification deferred to Ψ.

## Session log

- 2026-05-31 — `oglasino-backend-backend-calls-reduction-2` (C5 cache).
- 2026-05-31 — `oglasino-expo-backend-calls-reduction-2` (C1), `-3` (C2), `-4` (C4).
- Phase-2 audits: `oglasino-backend-backend-calls-reduction-1`, `oglasino-expo-backend-calls-reduction-1`.
