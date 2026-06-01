# Expo Auth Lifecycle

**Status:** in-progress
**Owner:** Igor (operator), Mastermind (planning), Mobile Engineer (execution)
**Last updated:** 2026-05-24
**Branch:** `dev`
**Affects:** `oglasino-expo` only. No backend, web, router, or Firestore Rules changes.

**Parent program:** [`expo-structural-foundation.md`](expo-structural-foundation.md) — Φ1 of four foundation chats.
**Authoritative auth contract:** [`user-deletion-auth-contract.md`](user-deletion-auth-contract.md) — the single source of truth for the auth lifecycle on the platform. Mobile inherits this contract. On any disagreement between this spec and the contract, the contract wins.
**Audit source:** [`sessions/audit-expo-structural.md`](../sessions/audit-expo-structural.md) — Findings F1, F2, F3, F4, F5, F13, F22 are the in-scope items.

---

## 1. Overview

Φ1 closes seven structural gaps in `oglasino-expo`'s auth lifecycle. The mobile app today relies on stale AsyncStorage persistence to "know" who is signed in. Banned and deleted users continue using the app with stale credentials. There is no foreground re-validation, no global 401/403 handling, no auth guards on secured routes, and store cleanup on logout is fragile.

The fixes are mobile-only. They adopt the contract that web already implements. No new behavior is invented; mobile mirrors web shapes where web has them, and adds mobile-specific lifecycle hooks (AppState, AsyncStorage hydration) only where the platform requires them.

### What this feature is

- Wires `initAuthListener` from `AppInit` with a single-flight guard and hydration gate.
- Adds `auth.signOut()` to the logout sequence so Firebase sessions tear down properly.
- Hardens store cleanup on logout — direct calls to user-scoped store clear methods.
- Adds an `AppState` listener that re-validates auth on foreground resume after >5 minutes of backgrounding.
- Adds a global axios 401/403 response interceptor that handles banned-user enforcement and token-refresh retry.
- Adds layout-level auth guards on secured routes.
- Resolves the hydration race between Zustand persistence and init components.

### What this feature is not

- It is **not** a backend change. The backend already supports everything mobile needs; mobile is conforming to existing contracts.
- It is **not** the deletion UI surface. Chat E (mobile user-deletion adoption) handles the danger zone, deletion-in-flight flag display, and post-deletion dialog wiring. Φ1 delivers the auth plumbing E depends on.
- It is **not** a navigation rearchitecture. Φ2 handles native navigators. Φ1's auth guards work inside whatever navigation shape exists.
- It is **not** a performance pass. Φ3 handles re-render math and `expo-image` adoption.
- It is **not** a service-layer rewrite. Φ4 handles the Part 7 error contract across all services. Φ1's 401/403 interceptor sits at the axios layer, not the service layer.
- It is **not** deep-link handler wiring. Φ1's auth guards transparently cover deep-link route entry (expo-router routes deep links through layout trees), but the deep-link entry-point parsing and target-route resolution are deferred.

---

## 2. Backend constraint

**No backend code changes happen in Φ1.** Mobile conforms to existing backend contracts.

The backend ships the response shapes mobile needs:

- `403 + {errors: [{field: null, code: 'USER_BANNED', translationKey: 'user.banned'}]}` — banned user response from `FirebaseAuthFilter` and `firebase-sync` (per `user-deletion.md` §8.8).
- `403 + {errors: [{field: null, code: 'EMAIL_BANNED', ...}]}` — re-registration block from `firebase-sync` (same source).
- `X-Account-Restored: true` response header — set by `firebase-sync` on PENDING_DELETION restore (per `user-deletion-auth-contract.md` C-9).

If the parallel backend audit chat surfaces a response-shape mismatch against these expectations, the resolution is to **change mobile to match the existing backend shape**, not to change the backend. Web is already in production consuming these contracts; backend changes would risk breaking web.

---

## 3. Authoritative contract

Mobile inherits the full `user-deletion-auth-contract.md` (sibling document). The clauses that apply to Φ1's mobile implementation:

- **C-1** — Postgres is the sole authority on deletion lifecycle. Mobile reads state via `syncUserToBackend` (which reads from S6 via the backend). Mobile never invents lifecycle state.
- **C-2** — Restoration is a sign-in event only. Mobile's `initAuthListener` calls `syncUserToBackend`, which calls `firebase-sync`, which performs the restoration. The listener handles the `X-Account-Restored: true` response header.
- **C-3** — The backend's auth filter on PENDING_DELETION drops the auth context and lets the request continue anonymously. Mobile observes this as the secure endpoint returning 401 for that user's stale-token request. Mobile's 401 interceptor handles it by force-refreshing the token (which will then either restore the user via `firebase-sync` or return null on hard delete).
- **C-4** — Filter returns 403 USER_BANNED for `disabled = true` users. Mobile's 403 interceptor catches this code and signs the user out with the ban dialog.
- **C-5** — Frontend awaits cookie-clear before navigation. **Does not apply directly to mobile** — mobile has no `firebase_token` cookie (no SSR layer). Mobile uses Firebase's in-memory `auth.currentUser` and the axios interceptor's `getIdToken()` call; sign-out is synchronous in-process. No cookie-clear race exists on mobile. Recorded here for completeness.
- **C-6** — `UseTokenRefresh` does not fire `firebase-sync` while deletion is in flight. **Does not apply to Φ1** — the deletion UI surface lives in chat E. When E ships, E adds the `deletionInFlight` flag and gates the listener accordingly. Φ1 ensures the listener exists and is callable.
- **C-7** — Axios interceptor's cached token invalidated on signout. **Applies to mobile.** Mobile's axios request interceptor reads `auth.currentUser` directly (no cached token), so this is already structurally correct. Recorded for completeness; verification step in F1's brief confirms no caching layer was added without notice.
- **C-8** — Cookie handling distinguishes missing from null/empty. **Does not apply to mobile** (no cookie).
- **C-9** — `firebase-sync` retains its full responsibility. Mobile calls `firebase-sync` via `syncUserToBackend`; the handler does the work as documented in `user-deletion.md` §10.1.
- **C-10** — Refresh-token revocation is a courtesy, not a defense. Mobile's 401 interceptor is the in-window defense. Reads from C-3 above.

The mobile-specific surface map (analogous to the six surfaces in the contract §2):

| #     | Surface                                       | What it stores                                           | Update latency                                   |
| ----- | --------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| **M1** | Firebase Auth client state                   | `auth.currentUser`, ID tokens                            | Immediate (in-process)                           |
| **M2** | Axios interceptor token retrieval             | `auth.currentUser.getIdToken()` called per request       | Immediate (Firebase SDK handles refresh)         |
| **M3** | AsyncStorage-persisted Zustand `authStore`    | `{ user }` (the `AuthUserDTO`), `_hasHydrated` flag       | Rehydrates at app start (~tens of ms)            |
| **M4** | Backend Redis auth cache (`redisUserAuth`)   | `AuthenticatedUserDTO` keyed by `firebaseUid`            | 30-min TTL, evicted on `saveUser`                |
| **M5** | Backend Postgres `users.deletion_status`     | The source of truth                                      | Transactional                                    |

Mobile is closer to web than it looks: M1=S1, M2 maps to S3 (token cache, but mobile reads fresh per request, so latency is near-zero), M3 has no web equivalent (this is the new race surface for mobile), M4=S5, M5=S6. Mobile has **no equivalent of S2** (no httpOnly cookie), which is why C-5 doesn't apply.

---

## 4. Findings and fix shapes

### F1 — Wire `initAuthListener`

**File:** `src/lib/store/authStore.ts:150-162` (the method, defined but never called) + `src/components/init/AppInit.tsx` (the call site).

**Today:** the method exists; zero call sites. Cold-start auth state comes only from AsyncStorage persistence. Banned/deleted users see the app as if still signed in.

**Fix:**
- Add a single-flight guard inside the auth store: a ref-equivalent (e.g., a module-scoped `unsubscribeRef` and an `isInitialized` flag) that ensures only one Firebase `onAuthStateChanged` subscription is active per app session.
- Call `initAuthListener` from `AppInit.tsx` once, after `_hasHydrated === true`.
- The listener's callback handles three response shapes from `syncUserToBackend`:
  - **Success with valid user** — update `user` in the store.
  - **Success with `X-Account-Restored: true`** — set a Zustand flag `restored: true`; a downstream dialog initializer (added in chat E) reads the flag and opens the restoration dialog. **Φ1 wires the flag; the dialog rendering lives in E.** Φ1's job: ensure the response header is read and the flag is set.
  - **Failure with `USER_BANNED` or `EMAIL_BANNED`** — call `auth.signOut()` (which then drains via the listener's null path), set a Zustand flag `accountBanned: true`. Φ1's job: wire the flag. The ban dialog itself is rendered by a Φ1-scope component `AccountStateDialogsInit` (see §6 — mirrors web's component of the same name).
  - **Any other failure** — log via the existing `logServiceError` pattern; do not mutate auth state.

**Single-flight contract:** if the listener is somehow called twice (developer error, hot reload edge case), the second call is a no-op. The guard ensures one subscription per app lifecycle.

**Hydration gate:** the listener must not subscribe before `_hasHydrated === true`. AppInit's mount effect gates on `_hasHydrated`.

**Inherits from contract:** C-2, C-4, C-9.

### F2 — `logoutFirebase` calls `auth.signOut()`

**File:** `src/lib/services/authService.ts:233-235`.

**Today:** only `GoogleSignin.signOut()` is called. Firebase session persists; `auth.currentUser` remains populated; axios interceptor continues attaching the stale token.

**Fix:** call `auth.signOut()` first, then `GoogleSignin.signOut()`. The Firebase call drains the listener with `null`, which triggers the listener's null-path cleanup (delegates to the auth store's `logout` action — see F3).

Two-line change. Order matters: Firebase first, Google second, because Firebase's signOut triggers downstream cleanup via the listener.

### F3 — Direct store cleanup on logout

**File:** `src/lib/store/authStore.ts:121-145` (the `logout` action).

**Today:** `logout` clears `user: null` and `viewTokenStore`. Other stores rely on component effects that may not be mounted.

**Fix:** `authStore.logout()` directly calls clear methods on every user-scoped store:
- `useChatStore.getState().clearChatStore()` (method exists per audit, never called)
- `useFavoritesStore.getState().clear()` (or equivalent — engineer audits the actual API)
- `useNotificationStore.getState().reset()` (per audit Finding 3)
- `usePortalFilterStore.getState().clearAllFilters()`
- `useDashboardFilterStore.getState().clearAllFilters()`
- `useCatalogStore.getState().clearCatalog()` (method exists per audit, never called)

The engineer audits each store's actual clear API and calls it. If a store has no clear method, the engineer adds one minimally (same shape as web — reset state to initial). No store gets a redesign.

Component-driven cleanup (`ChatsInit.unsubscribeAll()` on user null, etc.) remains in place as a secondary safeguard. The direct calls are the primary mechanism.

### F4 — Foreground re-validation

**File:** new logic in `AppInit.tsx` (or a sibling init component — engineer picks the cleanest placement).

**Today:** no AppState listener for auth re-validation.

**Fix:**
- Track `backgroundedAt: number | null` in a ref or module-scoped variable.
- Subscribe to `AppState.addEventListener('change', ...)`.
- On `nextAppState === 'background'` or `'inactive'`: record `backgroundedAt = Date.now()`.
- On `nextAppState === 'active'`: if `backgroundedAt !== null && Date.now() - backgroundedAt > 5 * 60 * 1000`, call `syncUserToBackend`. Same response handling as F1's listener — `X-Account-Restored` sets the restored flag, `USER_BANNED`/`EMAIL_BANNED` signs out and sets the banned flag.
- Reset `backgroundedAt = null` after handling (or on `active` regardless, to avoid stale state if `syncUserToBackend` is skipped).

**5-minute threshold** is the chosen value (per Phase 1 decision). Lives as a constant in the listener file, not in config. Single setting, no foreseeable second setting per Part 4a.

**Gated on `user !== null`:** if no user is signed in when the app foregrounds, skip the sync call. No work to do.

### F5 — 401/403 response interceptor (BLOCKED on Q2)

**File:** `src/lib/config/api.ts:33-62` (the existing response interceptor — extend it).

**Today:** response interceptor handles `ERR_NETWORK`, `ECONNABORTED`, missing response, 404. No 401, no 403.

**Fix:**
- **On 403 with body `{errors: [{code: 'USER_BANNED'}]}` or `{errors: [{code: 'EMAIL_BANNED'}]}`:**
  - Call `auth.signOut()`.
  - Set `useAuthStore.getState().setAccountBanned(true)`.
  - Return a never-resolving promise (matches web's pattern at `user-deletion.md` §14.12 — prevents the caller from receiving the error and double-handling).
- **On 401:**
  - Force-refresh the token via `auth.currentUser?.getIdToken(true)`.
  - Retry the original request once.
  - On second 401: call `auth.signOut()`; set the banned flag is **not** the right action here — 401 means token problem, not ban. Set a generic "session expired" Zustand flag (or just sign out and let the listener's null-path handle it). Engineer picks the cleanest shape; web's equivalent is the standard pattern.
- **Other 403s** (non-USER_BANNED, non-EMAIL_BANNED): fall through to per-call handling, unchanged.

**Blocking question Q2:** the body shape `{errors: [{code: 'USER_BANNED'}]}` is what web assumes (per `user-deletion.md` §14.12). The parallel backend audit chat confirms this. If the actual shape differs, mobile matches the backend's actual shape — not the web spec's documented shape. (Per the §2 backend constraint: backend stays put.)

**F5's brief does not run until Q2 is answered.** All other Φ1 briefs run in parallel.

### F13 — Auth guards on secured routes

**Files:** `app/(portal)/(secured)/_layout.tsx`, `app/owner/_layout.tsx`, `app/owner/dashboard/_layout.tsx`.

**Today:** each is `<Slot />` with no auth check. Only the BottomBar's UI-level gate prevents normal entry.

**Fix:** each layout reads `useAuthStore(s => ({ user: s.user, hydrated: s._hasHydrated }))`. While `!hydrated`, render `null` or an existing loading component (engineer picks). After hydration: if `!user`, redirect via expo-router's `<Redirect href="/" />`.

**Deep-link coverage:** expo-router routes deep links through layout trees. A deep link to `/messages` enters via `(portal)/(secured)/_layout.tsx`, which now guards. No additional deep-link work is required for the guard itself. Deep-link handler wiring (parsing the URL, choosing the right target) is deferred to a later chat.

**One implementation detail:** `owner/dashboard/_layout.tsx` accesses `user.profileImageKey` directly (per audit Finding 13). After the guard fix, this access is safe because the guard redirects before render when `user === null`. The engineer verifies this is the case in the final layout shape.

### F22 — Hydration race

**Files:** `src/components/init/ChatsInit.tsx:7`, `src/components/init/InitFavoritesStore.ts:11`, `src/notifications/components/NotificationsInit.tsx:6`.

**Today:** each reads `user` without checking `_hasHydrated`. On cold start they see `user: null` pre-hydration, trigger cleanup, then re-trigger setup when hydration completes.

**Fix:** each init component checks `_hasHydrated` before reading `user`. Pattern: `const { user, hydrated } = useAuthStore(useShallow(s => ({ user: s.user, hydrated: s._hasHydrated })))`. While `!hydrated`, return early — no subscriptions, no cleanups. The `useShallow` is a Φ3 concern (selector refactoring); for Φ1, the simpler `useAuthStore(s => s._hasHydrated)` separate subscription is acceptable. **Φ1 does not optimize subscriptions.** Φ3 does.

**Design constraint check:** structural; no UX change.

---

## 5. Q2 verification gate

F5's interceptor brief does not run until the parallel backend audit chat answers Q2.

**Q2:** does the backend's USER_BANNED 403 response include `{errors: [{field: null, code: 'USER_BANNED', translationKey: 'user.banned'}]}` in the body?

**Expected answer:** yes. Web is already in production consuming this shape (per `user-deletion.md` §14.12).

**If answer is yes:** F5 brief drafts with this body shape. Mobile interceptor matches.

**If answer surprises us:** mobile conforms to the actual backend shape. Web is already consuming whatever the actual shape is; mobile matches web, not the spec.

Igor returns the backend audit answer to this chat. F5 brief drafts on receipt.

---

## 6. New components introduced by Φ1

One new file. The rest are modifications.

### `src/components/init/AccountStateDialogsInit.tsx` (new)

Mirrors web's component of the same name (per `user-deletion.md` §14.4, §14.5, §14.10).

Subscribes to three Zustand flags on `useAuthStore`:
- `restored: boolean` — opens restoration dialog, clears flag in same effect
- `accountBanned: boolean` — opens ban-notice dialog, clears flag in same effect
- `accountJustDeleted: string | null` — opens post-deletion dialog (carries scheduled-deletion date); **this third flag is wired by chat E**, not Φ1. Φ1's `AccountStateDialogsInit` includes the subscription scaffold so E can add the dialog without re-touching the file.

Mounted from `AppInit.tsx` (sibling to the existing init components).

The dialogs themselves: ban dialog and restoration dialog reuse existing translation keys (`banned.dialog.*` in `DIALOG` namespace, `common.system.account.restored.*` in `COMMON_SYSTEM`). No new translation keys. Dialog component is whatever mobile already uses for InfoDialog-equivalent rendering — engineer picks; **does not invent a new dialog shape**.

---

## 7. Translation keys

**No new translation keys are seeded in Φ1.**

The ban dialog and restoration dialog reuse existing keys already seeded for web. The keys exist in the backend translation seed SQL files (per `user-deletion.md` §15). Mobile's i18n system reads from the same translation tables.

Verification step in the F1/F5/`AccountStateDialogsInit` engineer brief: confirm the keys resolve in the mobile translation system before relying on them in the dialog copy. If any key is missing on the mobile side specifically (mobile i18n caches a different subset than web), surface the gap; do not seed new keys without a separate Mastermind decision.

---

## 8. Trust boundaries

Every value used in moderation, authorization, or state-transition decisions:

| Value                                          | Source                                                       | Verification                       |
| ---------------------------------------------- | ------------------------------------------------------------ | ---------------------------------- |
| Whether user is banned                         | Backend `User.disabled` → `syncUserToBackend` response       | Server-derived; client cannot lie. |
| Whether user's email is banned (registration)  | Backend `banned_user_audit` hash check                       | Server-derived.                    |
| Whether to restore PENDING_DELETION on sign-in | Backend `firebase-sync` handler reads `deletion_status`     | Server-derived per C-2, C-9.       |
| `X-Account-Restored` flag                      | Response header set by backend                               | Server-derived.                    |
| Whether request should retry on 401            | Token refresh outcome (Firebase SDK)                         | Server-side decision (Firebase).   |
| User identity in store                         | Persisted from `syncUserToBackend` response                  | Server-derived initially.          |

**Stale persisted user:** the audit's central concern — Zustand persistence can carry a stale `user` across cold starts. Φ1's mitigation: `initAuthListener` fires on every app start and reconciles with the backend via `syncUserToBackend`. If the persisted user no longer matches backend state, the listener updates the store. The persisted user is **never** used in trust decisions — every authenticated request goes through the axios interceptor which reads `auth.currentUser.getIdToken()` (Firebase's verified token), not the Zustand `user`. The persisted user is presentational state; trust decisions ride on the Firebase token.

Trust boundary intact.

---

## 9. Sequencing

Engineering briefs in this order. F5 holds for Q2.

1. **Brief 1 — F2 (logoutFirebase signOut).** Smallest, lowest-risk. Two-line fix. Lands first to de-risk subsequent work that relies on logout actually draining Firebase.
2. **Brief 2 — F3 (direct store cleanup).** Builds on F2. Adds explicit clear calls in `authStore.logout()`.
3. **Brief 3 — F22 (hydration race).** Init components gate on `_hasHydrated`. Standalone; no dependency on other findings.
4. **Brief 4 — F1 (initAuthListener wiring) + `AccountStateDialogsInit` scaffold.** Wires the listener, the restoration/ban flag plumbing, and the dialog initializer. Largest brief.
5. **Brief 5 — F13 (auth guards).** Layout-level guards on three layouts. Standalone.
6. **Brief 6 — F4 (foreground re-validation).** AppState listener. Depends on F1's flag plumbing.
7. **Brief 7 — F5 (401/403 interceptor).** Holds for Q2.

Each brief is one session. Engineer agent does the work, writes session summary, Igor brings summary to Mastermind, Mastermind verdicts.

---

## 10. Definition of done

- All seven findings closed per the fix shapes in §4.
- `AccountStateDialogsInit` mounted from `AppInit.tsx` with restoration and ban flag handling.
- `npx tsc --noEmit`, `npm run lint`, `npm test`, `npx expo-doctor` clean.
- Igor manually verifies on a real device:
  - Sign in → background app for >5 min → foreground → no force-logout (happy path).
  - Banned-user scenario (requires admin ban via backend) → next request triggers ban dialog → sign-out + dialog appears.
  - PENDING_DELETION user signs in → restoration dialog appears.
  - Cold start with stale persisted user who has been deleted → listener fires → user is null → app shows public state.
  - Deep link to `/messages` while signed out → redirects to home.
  - Logout → re-login as different user → no leaked state from previous user.
- The three Risk Watch entries in `state.md` for Φ1 lifecycle issues are flipped to closed.
- Spec, decisions, state, issues updates applied by Docs/QA.

---

## 11. Out of scope

- Deep-link handler implementation (parsing URLs, choosing target routes). Auth guards transparently cover deep-link entry; handler wiring is later.
- `useShallow` adoption and Zustand selector refactoring. Φ3 work.
- React.memo, expo-image, AppContext memoization. Φ3 work.
- Native navigators replacing `<Slot />`. Φ2 work.
- Part 7 error contract in the service layer. Φ4 work.
- The deletion UI surface (danger zone, post-deletion dialog wiring). Chat E.
- Backend changes of any kind. The §2 backend constraint is binding.
- Mobile admin removal. Chat α (shipped 2026-05-24).
- New translation keys. §7 says reuse only.

---

End of spec.
