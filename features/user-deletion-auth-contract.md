# User Deletion — Auth Lifecycle Contract

**Status:** finalized (Mastermind, 2026-05-18 — revised against backend + web audits)
**Scope:** the auth boundary as it relates to user deletion, restoration, and ban enforcement
**Sibling document:** [`user-deletion.md`](user-deletion.md) — the feature spec. This contract supersedes specific clauses of §10 (the auth filter and firebase-sync) per the amendments listed in §10 of this document.
**Authoritative on conflict:** this contract wins over `user-deletion.md` §10. Other spec sections are unaffected.
**Source audits:** `oglasino-backend/.agent/audit-user-deletion-auth-lifecycle.md`, `oglasino-web/.agent/audit-user-deletion-auth-lifecycle.md`. Both confirm the contract's claims against on-disk code; no clauses required restructuring.

---

## 1. Why this document exists

The user-deletion feature shipped a backend half and a web half that passed 598 tests and two PR reviews. Manual testing then surfaced a race in which a server-side Next.js SSR call, fired during the post-deletion navigation, carried a still-valid `firebase_token` cookie to the backend. The backend's `FirebaseAuthFilter` verified the token, saw `deletion_status = PENDING_DELETION`, and ran `cancelDeletionOnLogin` — restoring the just-deleted user without the user ever signing back in.

The root cause is not in any single file. It's a **gap in the contract** between six surfaces that each carry some version of "is this user signed in?" — and each updates on a different schedule. The spec described the pieces (§10 filter, §10.1 firebase-sync, §10.2 delete endpoint, §14.3 fresh-token submit) but never the **interaction surface across all six on the timeline of a deletion**.

This document closes that gap. It is the canonical reference for any future engineer who needs to know whether a particular code path participates in the deletion lifecycle, and what's required of it.

---

## 2. The six surfaces

Each of the following carries auth state and updates with a different latency. Anywhere a future bug appears in this area, the diagnosis starts by mapping the symptom to one of these surfaces:

| #      | Surface                                                                 | What it stores                                                    | Who writes it                                                                              | Who reads it                                                 | Update latency                                                                                 |
| ------ | ----------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| **S1** | Firebase Auth client state                                              | `auth.currentUser`, ID tokens (~1hr TTL), refresh tokens          | Firebase SDK                                                                               | Frontend code, `UseTokenRefresh` listener                    | Immediate (synchronous in-process)                                                             |
| **S2** | `firebase_token` httpOnly cookie                                        | The current ID token                                              | `/api/auth/token` route handler (POST)                                                     | Server-side `fetchApi.ts` during SSR                         | One round-trip (the POST must complete + Set-Cookie must land before next browser request)     |
| **S3** | Axios interceptor cached token                                          | `cachedToken` module variable in `api.ts`                         | The interceptor itself, on first call after expiry                                         | Every `BACKEND_API` request                                  | Refreshes ~1min before token expiry; **not cleared on signout** (see §6 D-3)                   |
| **S4** | Next.js SSR request context                                             | Inherits cookies from the browser request                         | The browser sends them on every navigation                                                 | Server components, server actions, route handlers            | Per-request; cannot be "invalidated" — only future requests see the new cookie state           |
| **S5** | Backend Redis auth cache (`redisUserAuth`)                              | `AuthenticatedUserDTO` keyed by `firebaseUid`                     | `DefaultUserService.saveUser` evicts; filter populates on miss                             | `FirebaseAuthFilter` on every authenticated request          | 30-minute TTL; eviction via `@CacheEvict` is the freshness mechanism (per 2026-05-14 decision) |
| **S6** | Backend Postgres `users.deletion_status` + `user_deletion_requests` row | `'ACTIVE'` / `'PENDING_DELETION'` enum plus the dated request row | `UserDeletionService.requestDeletion`, `cancelDeletionOnLogin`, `executeScheduledDeletion` | Filter (via S5 cache), firebase-sync handler, scheduled jobs | Transactional commit                                                                           |

**The authoritative surface** for "is this user being deleted" is **S6 (Postgres)**. All other surfaces are derived caches or projections of S6. This is the foundational decision the contract turns on.

---

## 3. The race we hit, mapped to surfaces

Reproducing the 2026-05-18 manual test against this surface map:

| Time   | Event                                                                           | S1                          | S2                                                           | S3                | S4                 | S5                     | S6                            |
| ------ | ------------------------------------------------------------------------------- | --------------------------- | ------------------------------------------------------------ | ----------------- | ------------------ | ---------------------- | ----------------------------- |
| T+0    | User clicks Delete                                                              | logged-in (auth_time=T-45s) | old token                                                    | old token         | (idle)             | ACTIVE                 | ACTIVE                        |
| T+0.05 | `reauthenticateWithCredential`                                                  | fresh auth_time             | (unchanged)                                                  | (unchanged)       | —                  | —                      | —                             |
| T+0.1  | `getIdToken(true)` mints fresh token                                            | fresh token + auth_time     | (unchanged)                                                  | (unchanged)       | —                  | —                      | —                             |
| T+0.15 | `userService.deleteCurrentUser(freshToken)` submits                             | —                           | (unchanged)                                                  | (unchanged)       | —                  | (verifies fresh token) | ACTIVE → flipping...          |
| T+1.5  | `me/delete` commits                                                             | —                           | old token                                                    | old token         | —                  | evicted                | **PENDING_DELETION**          |
| T+1.6  | set `accountJustDeleted` on `useAuthStore`                                       | —                           | —                                                            | —                 | —                  | —                      | —                             |
| T+1.7  | `auth.signOut()`                                                                | **null**                    | (still old token; cookie clear is in flight)                 | (still old token) | —                  | —                      | —                             |
| T+1.7  | `onIdTokenChanged(null)` fires                                                  | —                           | clear in flight (Step 1: POST to /api/auth/token)            | (unchanged)       | —                  | —                      | —                             |
| T+1.7  | `onClose()` + `router.replace('/')`                                             | —                           | (clear still in flight)                                      | (unchanged)       | —                  | —                      | —                             |
| T+1.7  | Browser navigates to `/rs-sr`                                                   | —                           | **still carries old token** because Set-Cookie hasn't landed | —                 | (request inbound)  | —                      | —                             |
| T+1.8  | Next.js SSR for `/rs-sr` reads cookie                                           | —                           | **old token reaches fetchApi.ts**                            | —                 | request lifecycle  | —                      | —                             |
| T+2.0  | SSR calls `/public/product/search` with `Authorization: Bearer <old>`           | —                           | —                                                            | —                 | request to backend | filter loads S6        | —                             |
| T+5.7  | Backend filter sees PENDING_DELETION → fires `cancelDeletionOnLogin`            | —                           | —                                                            | —                 | —                  | re-populates as ACTIVE | **PENDING_DELETION → ACTIVE** |
| T+5.8  | Audit log marked CANCELLED, request row status=CANCELLED, products re-activated | —                           | —                                                            | —                 | —                  | —                      | (Rolled back)                 |

**The cookie-clear race (S2 lag) is the trigger.** **The filter's restoration behavior (S6 read coupled to a state mutation) is the amplifier.** Either fix alone is incomplete; the architectural fix is to break the amplifier so future trigger variants cannot cause restoration outside of an explicit sign-in.

---

## 4. The contract — clauses

The numbered clauses below are the durable contract. Engineers refer to them by number in code comments and session summaries.

### C-1. The Postgres state machine is the sole authority on deletion lifecycle

`users.deletion_status` (combined with `user_deletion_requests` rows) is the source of truth for "where is this user in the deletion lifecycle." Specifically:

- `deletion_status = 'ACTIVE'` AND no PENDING `user_deletion_requests` row → **ACTIVE**.
- `deletion_status = 'PENDING_DELETION'` AND PENDING `user_deletion_requests` row → **PENDING_DELETION**.
- (user row absent) → **DELETED** (post-hard-delete; the row no longer exists).

No other surface invents, infers, or short-circuits this state. Surfaces S1–S5 are derived caches; they may lag but they may not override.

### C-2. Restoration is a sign-in event only

Restoration of a `PENDING_DELETION` user to `ACTIVE` happens **only** at the explicit sign-in handshake — `POST /api/auth/firebase-sync`. No other endpoint, filter, or background job restores deletion state.

This is the architectural change at the heart of the contract. Today, `FirebaseAuthFilter` restores on any authenticated request to any endpoint, which created the race in §3. Under this contract, the filter inspects state but does not mutate it.

The `firebase-sync` handler restores per spec §10.1: load user, see PENDING_DELETION, call `cancelDeletionOnLogin`, set `X-Account-Restored: true` response header, return the now-ACTIVE user.

### C-3. The auth filter on PENDING_DELETION drops the auth context and lets the request continue anonymously

When `FirebaseAuthFilter` verifies a token and finds the user in `PENDING_DELETION`:

- **Do not** call `cancelDeletionOnLogin`. (Removed from filter.)
- **Do not** set `SecurityContextHolder` with the user's authentication.
- **Do not** return 403 or 401. The request continues.
- **Do** continue the filter chain as if no `Authorization` header had been sent.

The effect: public endpoints serve public data. Secure endpoints get an anonymous request and their existing auth machinery returns 401. The user observes the same behavior as if they had no session at all — which is correct, because functionally they don't: they've requested deletion.

The user can still restore the account by signing in via `firebase-sync` (C-2). The filter does not block this path; it just refuses to do it side-channel.

**Why "drop" and not "401":** a 401 from the filter would propagate as an error to whatever the SSR call was doing, potentially crashing renders for pages that should serve fine to anonymous users (the home page, public catalog, etc.). Dropping the auth context lets the existing endpoint-level authorization decide — which is the right boundary.

### C-4. The auth filter on `disabled = true` returns 403 with `USER_BANNED`

Unchanged from spec §10 step 3. Banned users get a hard rejection at the filter, not a drop. This is per existing design: a banned user must not be able to act as anonymous either (their `User.disabled = true` is a hard veto, separate from deletion).

### C-5. The frontend awaits cookie-clear before navigation on deletion success

`DeleteAccountConfirmationDialog`'s success path:

1. Submit deletion (`userService.deleteCurrentUser(freshToken)`).
2. On 200: set `accountJustDeleted` on `useAuthStore` (carrying `scheduledDeletionAt`). Web and mobile both use a Zustand store flag; see spec §14.4. This replaces the retired sessionStorage mechanism.
3. **`await auth.signOut()`** — completes Firebase client-side sign-out.
4. **`await clearFirebaseTokenCookie()`** — explicitly POST `/api/auth/token` with `{token: null}` and wait for the response. This is the new step. Today the `UseTokenRefresh` listener handles cookie clearing asynchronously via `onIdTokenChanged(null)`; the success path does not wait for it.
5. `onClose()` — dismiss the dialog.
6. `router.replace('/<locale>')` — navigate. By this point S2 is provably cleared.

`clearFirebaseTokenCookie()` is a one-line alias exported from `authTokenCookie.ts` over the existing `writeFirebaseTokenCookie(null)`. Per the web audit Q-1.5, no new network surface is needed; the existing helper already does the work. The alias exists for call-site readability — the dialog body reads "clear the cookie," not "write null."

**The cookie-clear call must be wrapped in a local try/catch that swallows and logs.** If `clearFirebaseTokenCookie()` rejects (network blip after a successful deletion commit), the dialog must still navigate. Under C-3, the stale cookie cannot cause restoration, so swallowing is safe. If the outer catch were to handle the rejection, the user would see "system error" on a successful deletion and the navigation would never happen. Engineering brief instructs this explicitly.

The await on step 4 is the frontend half of the fix. Combined with C-3 (the backend half), the system defends against the race from both sides: even if the cookie-clear await fails or is bypassed, the backend will not restore on the resulting SSR call.

### C-6. `UseTokenRefresh` does not fire `firebase-sync` while deletion is in flight

Even though C-3 makes the backend filter inert in this window, the frontend should not waste round-trips firing redundant `firebase-sync` POSTs during the few hundred milliseconds between deletion submit and full sign-out. Add a `deletionInFlight: boolean` flag on `useAuthStore`. Set to `true` immediately before `getIdToken(true)` in the dialog; clear in the `finally` block. `UseTokenRefresh`'s `onIdTokenChanged` handler reads the flag and skips the `firebase-sync` POST when true. The cookie-write step still happens (it must, to clear).

**The "immediately before `getIdToken(true)`" placement is load-bearing.** `getIdToken(true)` triggers `onIdTokenChanged` synchronously inside Firebase's SDK as part of the token mint. If `setDeletionInFlight(true)` is set after `getIdToken(true)`, the handler fires with the flag still false and the redundant `firebase-sync` POST fires anyway. The engineer must not move the flag-set call to before reauth either — reauth itself triggers `onIdTokenChanged` and the flag would be set too early.

This is not load-bearing for correctness under C-3 — it's a hygiene measure that removes a noisy code path from the deletion timeline and makes future debugging easier.

### C-7. The axios interceptor's cached token is invalidated on signout

The `cachedToken` module-scoped variable in `api.ts` (per audit §3 and Brief E "Seams") survives `auth.signOut()` today. It expires naturally only when its `expirationTime` arrives. Add an `onIdTokenChanged` subscription at module-init time that resets `cachedToken = null` and `tokenExpiry = null` when `firebaseUser === null`.

Under C-3 this is not strictly required for correctness — but it removes a confusing surface where in-memory client code can still attach a stale token to a request after `auth.signOut()`. The cost is a few lines; the benefit is observable cleanliness.

### C-8. `fetchApi.ts` SSR helper must distinguish missing cookie from null/empty cookie

If the `firebase_token` cookie has been explicitly cleared (value `null`, `""`, or absent), `fetchApi.ts` does not set the `Authorization` header. It must not send `Authorization: Bearer null` or `Authorization: Bearer `. This is presumed-correct today (no engineer flagged it as broken) but the contract makes it explicit.

### C-9. The `firebase-sync` handler retains its full responsibility

`POST /api/auth/firebase-sync` continues to do what spec §10.1 describes:

1. Verify token. Extract email.
2. Check `userAuditService.isEmailBanned(email)` — on hit, delete Firebase user (best-effort), return 403 `EMAIL_BANNED`.
3. `getOrCreateUser`.
4. If `user.disabled` → 403 `USER_BANNED`.
5. If `user.deletionStatus = 'PENDING_DELETION'` → `cancelDeletionOnLogin`, set `X-Account-Restored: true` header.
6. Return `AuthUserDTO`.

The change vs spec §10.1 is the surrounding context, not this endpoint's logic: under this contract, this is the **only** endpoint that ever restores. Spec §10.1 stays as-is; spec §10 (the filter) changes per C-3.

### C-10. Refresh-token revocation remains a courtesy, not a defense

Spec §4.2 calls `FirebaseAuth.revokeRefreshTokens(uid)` at `requestDeletion`. This forces existing sessions to fail on next token _refresh_ (typically within 1 hour), but does **not** invalidate already-issued ID tokens. The contract acknowledges this: the 1-hour window is real, and C-3 is the defense within that window (anyone presenting a still-valid ID token during the window gets anonymous treatment, not restoration).

Revocation is kept because it shortens the window — refresh fails when the token rotates, which signs the user out on every device — but no part of the deletion correctness relies on it firing immediately.

---

## 5. What happens on each surface during a clean deletion + restoration

This is the reference timeline once the contract is in force. Every surface's behavior is determined by the contract clause cited.

### Successful deletion (no restoration)

| Step                                                                                                     | Surface activity                                                 | Clause                                            |
| -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------- |
| 1. User clicks Delete in `/owner/user` Danger Zone                                                       | (dialog opens)                                                   | —                                                 |
| 2. Reauth completes                                                                                      | S1: `auth.currentUser` carries fresh `auth_time`                 | —                                                 |
| 3. `getIdToken(true)`                                                                                    | S1 mints fresh ID token                                          | —                                                 |
| 4. `useAuthStore.setDeletionInFlight(true)`                                                              | (flag set)                                                       | C-6                                               |
| 5. POST `/me/delete` with fresh token                                                                    | S6: PENDING_DELETION committed, request row inserted, S5 evicted | C-1                                               |
| 6. set `accountJustDeleted` on `useAuthStore`                                                            | (UI handoff)                                                     | —                                                 |
| 7. `await auth.signOut()`                                                                                | S1: `auth.currentUser = null`                                    | —                                                 |
| 8. `await clearFirebaseTokenCookie()`                                                                    | S2: cookie cleared on the server before response returns         | C-5                                               |
| 9. `useAuthStore.setDeletionInFlight(false)`                                                             | (flag cleared in `finally`)                                      | C-6                                               |
| 10. `onClose()` + `router.replace('/')`                                                                  | (navigation)                                                     | —                                                 |
| 11. Browser navigates with cleared cookie                                                                | S2: absent → SSR sees no token                                   | C-5 + C-8                                         |
| 12. Next.js SSR for `/<locale>` calls backend with no Authorization                                      | S4: anonymous                                                    | C-8                                               |
| 13. Backend processes as anonymous request                                                               | S5: no lookup; S6: untouched                                     | C-3 (would also kick in if a stale token arrived) |
| 14. `AccountStateDialogsInit` reacts to the `accountJustDeleted` flag → post-deletion dialog opens, flag cleared in the same effect | (UI)                                                             | —                                                 |
| 15. Subsequent days: cron eventually finds the PENDING request, runs `executeScheduledDeletion` at Day 7 | S6: user row deleted                                             | spec §4.5 unchanged                               |

### Restoration (user signs back in within 7 days)

| Step                                                                                   | Surface activity                                                                                              | Clause               |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | -------------------- |
| 1. User opens app, sees login form                                                     | S1: `auth.currentUser = null`                                                                                 | —                    |
| 2. User signs in with email/password (or Google/OAuth)                                 | S1: `auth.currentUser` populated with fresh token                                                             | —                    |
| 3. Frontend calls `POST /api/auth/firebase-sync`                                       | S4: token reaches backend                                                                                     | —                    |
| 4. Backend handler verifies token, finds `user.deletionStatus = 'PENDING_DELETION'`    | S6: read                                                                                                      | C-9                  |
| 5. Handler calls `cancelDeletionOnLogin`                                               | S6: status flips ACTIVE, request row CANCELLED, audit log marked cancelled, products re-activated, S5 evicted | C-2 + C-9            |
| 6. Handler returns AuthUserDTO with `X-Account-Restored: true` header                  | —                                                                                                             | C-9                  |
| 7. Frontend axios response interceptor reads header → `useAuthStore.setRestored(true)` | —                                                                                                             | spec §14.5 unchanged |
| 8. `AccountStateDialogsInit` opens restoration dialog                                  | (UI)                                                                                                          | spec §14.5 unchanged |

### Stale-token attack window (what C-3 protects against)

| Step                                                                                           | Surface activity                       | Clause  |
| ---------------------------------------------------------------------------------------------- | -------------------------------------- | ------- |
| 1. User has clicked Delete; their ID token is still valid for up to 1hr                        | S1: stale token in memory or somewhere | —       |
| 2. An authenticated request reaches the backend                                                | S2/S3/S4: token present                | —       |
| 3. Filter verifies token, loads S5 cache → sees PENDING_DELETION                               | S5: PENDING_DELETION                   | C-1     |
| 4. Filter drops the auth context, lets request continue                                        | request becomes anonymous downstream   | **C-3** |
| 5. Endpoint serves response based on anonymous identity (public data) or 401 (secure endpoint) | —                                      | C-3     |
| 6. **No restoration occurs.** S6 untouched.                                                    | —                                      | C-2     |

This is the architectural fix. The race in §3 cannot happen under this clause regardless of which surface lags or which request leaks through.

---

## 6. Decisions explicitly made and rejected

Surfaced for the record so future engineers don't re-litigate them.

### D-1. "Why not return 403 USER_PENDING_DELETION from the filter?"

Considered. Rejected because some surfaces that legitimately receive a token from a PENDING_DELETION user should still serve them content as anonymous — most importantly, the SSR call rendering the public home page they're being redirected to. A 403 from the filter would crash that render. Dropping the auth context (C-3) is the surgical move: secure endpoints still 401 via their own machinery; public endpoints serve public content.

### D-2. "Why not a Redis 'deletion in flight' flag (Option C from the architectural discussion)?"

Considered. Rejected because it introduces a second source of truth alongside `users.deletion_status`, which can drift. The `deletion_status` column already exists and is the natural authority (C-1). Adding a flag would buy nothing the column doesn't already provide.

### D-3. "Why is the axios interceptor cached token reset (C-7) only 'observable cleanliness' and not load-bearing?"

Under C-3, the backend defends itself regardless of which client-side cache leaked a stale token. So C-7 is hygiene, not correctness. We're including it because the surface is small, the value-of-cleanliness is high (one less mysterious surface during future bug hunts), and there's no good argument for _not_ clearing it on signout. If a future review wants to drop C-7 to reduce code surface, that's defensible — but currently the cost-benefit is in favor of including it.

### D-4. "Why not also revalidate the SSR cookie via a server middleware?"

Considered. Rejected for now: it adds Next.js middleware complexity for a problem C-3 + C-5 already fully solve. If we later find a class of leak that C-5's await doesn't catch, middleware becomes the next defense; today it's overkill.

### D-5. "Why is C-8 stated explicitly when it's presumed-correct?"

Because it's not on disk anywhere as a documented requirement. If a future refactor of `fetchApi.ts` changes how the cookie is read, the engineer doing the refactor needs to know that "Bearer null" or "Bearer " on the wire would re-open a variant of the original race (the backend's `verifyIdToken` would throw on "null", but the surrounding error path might short-circuit differently). Naming the requirement prevents that drift.

---

## 7. Audit resolution

The contract's open questions have been answered by two read-only audits:

- **Backend audit** at `oglasino-backend/.agent/audit-user-deletion-auth-lifecycle.md` (2026-05-18) — answered Q-1 through Q-6 of the backend brief.
- **Web audit** at `oglasino-web/.agent/audit-user-deletion-auth-lifecycle.md` (2026-05-18) — answered Q-1 through Q-6 of the web brief.

### Resolved findings (binding for engineering briefs)

- **Q-1 (Spring Security behavior on unset SecurityContextHolder):** confirmed. Spring's `AnonymousAuthenticationFilter` installs an `AnonymousAuthenticationToken` when our filter leaves the context unset. `/api/secure/**` `authenticated()` rule returns 401; `/api/public/**` and `/api/auth/**` `permitAll` rules serve through. The precedent for "drop context, let URL rules decide" already exists in `FirebaseAuthFilter.java:122-129` (exception branch). C-3 is implementable as a one-clause amendment.
- **Q-2 (frontend cookie-clear semantics):** confirmed. The Next.js `cookies().delete()` API emits a `Set-Cookie` clearing header (`Expires=Thu, 01 Jan 1970...`) that the browser applies before the fetch promise resolves. `await writeFirebaseTokenCookie(null)` resolving means the cookie is clear from the browser's perspective. No streaming or buffered-flush shape in the route handler that would break this.
- **Q-3 (axios interceptor lifecycle):** confirmed. `cachedToken` and `tokenExpiry` are the only two module-scoped auth variables in `api.ts`. The module is loaded once per browser session (client-side singleton). `auth` import is already present. C-7 is ~6 lines added at module scope.
- **Q-4 (useAuthStore shape):** confirmed. The store's `set()` pattern mirrors `restored` / `setRestored` exactly. No naming collision. `loading` is consumed by other auth actions (login, register, refreshUser) and reusing it would couple unrelated flows — dedicated `deletionInFlight` field is correct.
- **Q-5 (filter exemption list):** confirmed. Only `/api/auth/firebase-sync` is exempt (substring match at `FirebaseAuthFilter.java:79`). `/api/public/**` is **not** exempt — the filter verifies tokens when present, which is why the 2026-05-18 race ran on `/public/product/search`. OPTIONS preflight and Postman-bypass are additional skip paths (not relevant to C-3).
- **Q-6 (fetchApi.ts cookie handling):** confirmed via behavior matrix. `undefined` and `""` cookie values both fold into "no Authorization header sent" via the existing truthy-check. The literal string `"null"` would send `Bearer null`, but no code path writes that value to the cookie (the route handler uses `cookies().delete()`, never writes `"null"`).
- **Q-7 (sixth-surface model):** confirmed. No seventh surface in either repo. `useChatStore.userCache`, `useChatBlockStore`, `useViewTokenStore`, `notificationManager` are per-user state but do not carry the Firebase token and are not consulted by any backend-bound auth path.

### Findings folded into the contract from the audits

- **C-5 amendment (web audit Q-1.5):** the helper named `clearFirebaseTokenCookie()` is a one-line alias over the existing `writeFirebaseTokenCookie(null)`, not a new network surface. Contract C-5 above reflects this.
- **C-5 cookie-clear failure swallowing (web audit Q-6.2):** the cookie-clear call must be wrapped in a local try/catch that swallows. Contract C-5 above reflects this.
- **C-6 flag placement (web audit Q-2.5):** the "immediately before `getIdToken(true)`" wording is load-bearing because `getIdToken(true)` triggers `onIdTokenChanged` synchronously. Contract C-6 above reflects this.

### Adjacent observations from the audits

Both audits surfaced low-severity adjacent observations. Most are pre-existing, deletion-irrelevant, and queued for future cleanup briefs. One is folded into this work as adjacent fix-in-passing:

- **Backend audit observation A (filter exception-branch broadening):** today `FirebaseAuthFilter.java:125` suppresses the 401 on token-verification failure only for `/api/public/**`. But `/api/auth/**` and `/internal/**` are also `permitAll`. A malformed token to `/api/auth/firebase/{id}` returns 401 even though the URL rule would have served it as anonymous. **Folded into the backend engineering brief as adjacent cleanup** (per D-4 of the round-3 chat). Once C-3 reshapes the filter to "drop context, let URL rules decide," the consistency win is one extra line of code.

Other adjacent observations (backend B, web 1-6) are queued for a future cleanup brief — see backlog entries in the round-2 handoff.

---

## 8. What this contract does not cover

Listed so future readers don't expect it to.

- The post-deletion dialog lifecycle bug (handoff Task 2). That's a separate issue — the dialog not opening after `router.replace` is unrelated to the auth boundary.
- Admin ban-with-reason UX (handoff Task 3). Separate scope.
- Testing infrastructure (handoff Task 4). Separate.
- Mobile adoption. Separate Mastermind chat post-merge.
- The 7+ second `/public/product/search` SSR call. Performance issue surfaced incidentally during the test; not a deletion bug. Backlog as a perf brief.
- The hour-long ID token validity window itself. Inherent to Firebase's design; we live with it and C-3 is our defense.

---

## 9. Implementation impact summary

Both audits resolved the open questions and produced concrete diff sketches. The engineering briefs are mechanical from this point — no architectural decisions remain.

**Backend (single-file change in `FirebaseAuthFilter.java`):**

- Remove the PENDING_DELETION mutation branch (per C-3). Clear `SecurityContextHolder` and fall through, mirroring the existing exception-branch precedent at `FirebaseAuthFilter.java:122-129`.
- Broaden the exception-branch 401-suppression to all `permitAll` paths (`/api/public/**`, `/api/auth/**`, `/internal/**`), not just `/api/public/**`. Adjacent cleanup folded in per D-4 of the round-3 chat.
- Tests in `FirebaseAuthFilterTest`: delete two tests (the PENDING_DELETION restore test, the stale-cache race test), rewrite one (the fresh-cache-skip test), add one or two new tests (PENDING_DELETION drops context and falls through anonymous). `AuthControllerFirebaseSyncTest` and `DefaultUserDeletionServiceTest` are unaffected.
- Spec amendment to `user-deletion.md` §10 reflecting C-3.
- No DB schema changes. No new endpoints. No new error codes. No translation work.

**Frontend (changes in four files):**

- `authTokenCookie.ts`: add a one-line alias `clearFirebaseTokenCookie = () => writeFirebaseTokenCookie(null)`. Export alongside the existing helper.
- `DeleteAccountConfirmationDialog.tsx`: add `setDeletionInFlight(true)` immediately before `getIdToken(true)`; clear in the existing `finally`. After `auth.signOut()`, add `await clearFirebaseTokenCookie()` wrapped in a local try/catch that swallows and logs. Then `onClose()` and `router.replace`. Three new lines plus one new import.
- `useAuthStore.ts`: add `deletionInFlight: boolean` field + `setDeletionInFlight(value)` action. Pattern mirrors `restored` / `setRestored`.
- `UsetTokenRefresh.tsx`: read `useAuthStore.getState().deletionInFlight` inside the `onIdTokenChanged` handler; skip the `firebase-sync` POST when true. One new if-block.
- `api.ts`: subscribe to `onIdTokenChanged` at module-init; on null user, reset `cachedToken` and `tokenExpiry`. Cheap-fire-and-forget version (no HMR unsubscribe) per D-5 of the round-3 chat.
- No new dialogs. No new wire types. No new translation keys. No tests added (per the standing testing-infrastructure gap).

**Spec amendments (drafted in the briefs, applied by Docs/QA):**

- `user-deletion.md` §10 — rewrite the filter's PENDING_DELETION behavior per C-3.
- `user-deletion.md` §14.3 — note the cookie-clear await per C-5.
- Reference link from spec to this contract document.

---

## 10. Amendments to the feature spec

When this contract is finalized and the briefs land, the following sections of `user-deletion.md` are amended:

- **§10** (Auth filter changes) — step 4 "If authData.deletionStatus == 'PENDING_DELETION'" is replaced by "Drop the auth context and continue the filter chain anonymously (per auth contract C-3). Do not call `cancelDeletionOnLogin`." The `X-Account-Restored` header is no longer set by the filter; it's set only by `firebase-sync` (§10.1, unchanged).
- **§10.2** (Delete-account endpoint) — unchanged; the endpoint receives the fresh-reauth token from the dialog and proceeds as before. The endpoint sits behind a `@PreAuthorize`-equivalent check; per Q-1 confirmation, an unauthenticated request to it from a stale-token user receives 401.
- **§14.3** (Post-reauth deletion flow) — step 5 ("Sign the user out") becomes:

  > 5. Set `deletionInFlight = true` on `useAuthStore` (cleared in `finally`).
  > 6. Submit deletion request with the fresh token.
  > 7. On 200: set `accountJustDeleted` on `useAuthStore` (carrying `scheduledDeletionAt`).
  > 8. `await auth.signOut()`.
  > 9. `await clearFirebaseTokenCookie()`.
  > 10. `onClose()`; `router.replace(\`/${locale}\`)`.

  (The existing numbering shifts accordingly.)

- **§17.14** (Concurrent restore-on-login and admin-ban race) — the race acknowledged here is now structurally impossible per C-2 + C-3. The section can be deleted or rewritten as "historical; superseded by auth contract."

- New cross-reference to this document at the top of §10 and §14.

Docs/QA applies these amendments after the engineering briefs land and you've verified the implementation end-to-end.

---

## 11. Glossary

For readers who arrive here without context on the feature.

- **Deletion request** — user-initiated, in-app action. Backend records intent; user has 7 days to restore by signing in. After 7 days, hard delete.
- **Restoration** — when a `PENDING_DELETION` user signs in (via `firebase-sync`), the system reverses the deletion request and the user resumes as ACTIVE.
- **Hard delete** — irreversible removal of the user row plus cascades (Day 7 cron job, per spec §4.5).
- **Ban** — admin-initiated, separate code path. Banned users cannot sign in. Different from deletion: bans have no grace period and no self-restore.
- **Anonymous** — in the C-3 sense, a request that reaches the backend without an authenticated principal in `SecurityContextHolder`. Public endpoints serve it normally; secure endpoints reject with 401.

---

End of contract.
