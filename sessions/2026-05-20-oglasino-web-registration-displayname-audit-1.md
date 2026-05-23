# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit brief — Registration displayName persistence (web side). Read-only audit of how `displayName` flows through the registration form, store, and `firebase-sync` service; propose a fix shape to extend the `firebase-sync` POST body to carry `displayName` (symmetric with the edit-profile flow).

## Implemented

- Read-only audit. No code changes on disk.

## Files touched

- (none — read-only)

## Tests

- Not run. Read-only audit.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (the existing 2026-05-19 entry "Registration `displayName` never persists to backend" already tracks this work — this audit is its read-only prep)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind" (Firestore `users/<uid>.displayName` is empty for email+password registrations; see observation)
- Part 6 (translations): confirmed — no new keys proposed in this audit; one observation about reusing existing `username.empty` (VALIDATION, frozen) vs introducing a new `ERRORS` key noted under "For Mastermind"
- Part 7 (error contract): N/A this session
- Part 11 (trust boundaries): explicitly checked — see "Trust-boundary check" below
- Other parts touched: none

## Known gaps / TODOs

- None.

---

# Audit findings

## 1 — Registration form

### Form component

`src/components/popups/dialogs/RegisterDialog.tsx`

- **`displayName` state binding.** `src/components/popups/dialogs/RegisterDialog.tsx:45-49` declares `userData: RegisterUserRequestDTO` via `useState`, initialised to `{ displayName: '', email: '', password: '' }`. `handleChange('displayName', val)` (line 61-64) spread-merges into `userData` on every input change. Matches the project's prop-drilled `onChange(partial)` form pattern per CLAUDE.md.
- **Input rendering.** `RegisterDialog.tsx:137-144` renders an `Input` with `label={tInputs('username.label')}`, bound to `userData.displayName`, with `errors.displayName` as the error surface and `handleChange('displayName', val)` as the change handler.
- **Submit flow.** `registerInternal` (line 99-111) calls `validateForm()` → runs reCAPTCHA → on success calls `register(userData)` (the store action). The full `userData` object — including the populated `displayName` — is passed in.
- **Client-side validation on `displayName`.** `RegisterDialog.tsx:69-74`:
  ```ts
  if (!userData.displayName) {
    setError('displayName', tValidation('username.empty'));
    formValid = false;
  } else {
    setError('displayName', '');
  }
  ```
  Required (empty string fails). No length check. No character-set check. No `trim()`. A single space ` ` passes the truthy check today.

### `register` action in `useAuthStore`

`src/lib/store/useAuthStore.ts:197-209`:

```ts
register: async (userData: RegisterUserRequestDTO) => {
  try {
    set({ loading: true, error: null });
    await registerUserFirebase(userData.email, userData.password);
  } catch (err) {
    console.error('[auth.store] auth action failed:', err);
    const friendly = mapAuthError(err);
    if (friendly !== null) set({ error: friendly });
  } finally {
    set({ loading: false });
  }
},
```

- **What happens to `displayName` today.** It enters the action via `userData: RegisterUserRequestDTO` (`.../types/user/RegisterUserRequestDTO.ts:1-5` — `{ displayName, email, password }`). It is destructured into nothing — only `userData.email` and `userData.password` are forwarded into `registerUserFirebase`. The `displayName` field is silently dropped.
- **No subsequent reference to `userData.displayName` anywhere in the action.** Grep on `displayName` in `useAuthStore.ts` returns zero hits.

### `registerUserFirebase` service call

`src/lib/service/reactCalls/authService.ts:172-175`:

```ts
export const registerUserFirebase = async (email: string, password: string): Promise<void> => {
  const res = await createUserWithEmailAndPassword(auth, email, password);
  await ensureUserInFirestore(res.user);
};
```

- Signature is `(email, password)`. No `displayName` parameter.
- Calls `createUserWithEmailAndPassword(auth, email, password)` — Firebase Auth's primitive that does **not** accept a `displayName`. The resulting `FirebaseUser` has `displayName: null`.
- Calls `ensureUserInFirestore(res.user)` — see Firestore note below.
- Returns `void`. Per the comment at `authService.ts:162-166`, the action-layer functions return void by design — the `UseTokenRefresh` listener (`onIdTokenChanged`) is the sole hydrator of `useAuthStore.user`. The listener fires `syncUserToBackend(firebaseUser)`, which POSTs `/auth/firebase-sync`.

### `/auth/firebase-sync` POST body — today's exact shape

`src/lib/service/reactCalls/authService.ts:136-138`:

```ts
const response = await BACKEND_API.post<AuthUserDTO>('/auth/firebase-sync', {
  allowPreferenceCookies: globalCookie && globalCookie.cookieConsent?.preference,
});
```

Body fields, today: **`allowPreferenceCookies` only**. No `displayName`. Grepped both `authService.ts` and the whole `app/` + `src/` tree for `firebase-sync`: only this one call site.

### Confirming the in-memory DTO patch was genuinely removed

The 2026-05-19 frontend Observation-1 patch (commit `d7ce6f9` — "Lets role this web user deletion feature....") replaced the four sign-in actions with the listener-is-sole-hydrator design. The `register` block specifically lost these three lines:

```ts
const backendUser = await registerUserFirebase(userData.email, userData.password);
backendUser.displayName = userData.displayName;
set({ user: backendUser });
```

- The current `register` action (`useAuthStore.ts:197-209`) contains no remaining patch — no `setUser({…displayName…})`, no shadow assignment.
- Grep `displayName` in `useAuthStore.ts` returns zero hits in the current file. Confirmed clean.
- No leftover comment referencing the removed patch.

### Client-side validation on `displayName` at register (summary)

| Constraint | Behaviour today |
| --- | --- |
| Required | Yes — `!userData.displayName` → `tValidation('username.empty')` |
| Length min | None |
| Length max | None |
| Character set | None |
| Trim before check | No — `' '` (single space) currently passes |

---

## 2 — Edit-profile flow (symmetry reference)

### Component and submit handler

`app/[locale]/owner/user/page.tsx` (component `UserInfo`, default export).

- **State binding.** Local `displayName` state via `useState('')` at line 46, hydrated from `getUserDetails(user)` at line 66 (`setDisplayName(details.displayName)`).
- **Input rendering.** Line 221 — `<Input label={tInput('username.label')} value={displayName} onChange={setDisplayName} />`. Same `username.label` translation key as register.
- **Submit handler.** `saveChanges` (lines 98-205) computes `userChanged`, validates non-empty `displayName` (line 136-140: `if (!displayName) { setErrorMessage(tError('username.required')); ... }`), uploads avatar if present, then POSTs an `UpdateUserDTO` body via `updateUser(...)` at line 177-190. The DTO includes `displayName` alongside many other profile fields (email, shortBio, phone, etc.).

### Backend endpoint, method, body shape

`src/lib/service/reactCalls/userService.ts:36-50`:

```ts
export const updateUser = async (user: UpdateUserDTO): Promise<boolean> => {
  try {
    const res = await BACKEND_API.post('/secure/user/update', user);
    ...
```

- **URL:** `POST /secure/user/update`
- **Body shape:** `UpdateUserDTO` (`.../types/user/UpdateUserDTO.ts`):
  ```ts
  { id, firebaseUid, displayName, email, profileImageKey, shortBio,
    phoneNumber?, regionAndCity?, allowPreferenceCookies?, allowNotifications?,
    allowEmails?, allowPromoEmails?, allowPhoneCalling, providerId? }
  ```

### Client-side validation on `displayName` (edit path)

`app/[locale]/owner/user/page.tsx:136-140`:

```ts
if (!displayName) {
  setErrorMessage(tError('username.required'));
  setLoading(false);
  return;
}
```

- Required only. No length, charset, or trim.
- Different translation key from register: edit uses `tError('username.required')` (namespace `ERRORS`); register uses `tValidation('username.empty')` (namespace `VALIDATION` — frozen per conventions Part 6).
- Same semantic constraint (required) but inconsistent key surface. Flagged under "For Mastermind."

### Does edit-profile hit Firebase Auth?

No. `updateUser` only POSTs to the backend at `/secure/user/update`. There is **no** call to `updateProfile(auth.currentUser, { displayName })` or any other Firebase Auth client primitive in `saveChanges`. The backend owns the canonical store; Firebase Auth's own `displayName` field is not maintained on the edit path. This is exactly the pattern the chosen register-fix design extends to the register path.

---

## 3 — `firebase-sync` service / endpoint reference

- **Service function:** `syncUserToBackend` in `src/lib/service/reactCalls/authService.ts:127-157`.
- **Sole caller:** `UseTokenRefresh.tsx:38` inside `onIdTokenChanged`.
- **Method + URL:** `POST /auth/firebase-sync`.
- **Request body, today:**
  ```ts
  { allowPreferenceCookies: globalCookie && globalCookie.cookieConsent?.preference }
  ```
  One field. `displayName` does **not** appear anywhere in the request body.
- **Does `displayName` appear elsewhere in `authService.ts`?** Only at line 80, inside `ensureUserInFirestore`, where the Firestore `users/<uid>` doc is created with `displayName: firebaseUser.displayName || ''`. For OAuth signups (Google/Facebook) this is the provider-supplied name. For email+password signups, `firebaseUser.displayName` is `null` (Firebase Auth has no displayName for password-only accounts), so the Firestore doc lands with `displayName: ''`. This Firestore write is **independent** of the firebase-sync POST. Not echoed back into the request body.

---

## 4 — In-context places `displayName` is read after register

- **Store field.** `useAuthStore.user.displayName` — typed via `AuthUserDTO.displayName: string` (`.../types/user/AuthUserDTO.ts:7`).
- **Hydration path post-register.** `createUserWithEmailAndPassword` triggers `onIdTokenChanged` → `UseTokenRefresh.tsx:22-43` calls `syncUserToBackend(firebaseUser)` → backend returns an `AuthUserDTO` → listener calls `useAuthStore.getState().setUser(backendUser)`. Whatever the backend persisted as `displayName` at that moment becomes the visible value; today that is whatever the backend defaults to (empty / placeholder / Firebase-derived — depends on backend behaviour, out of scope here).
- **Example consumers within a few seconds of register completing:**
  - `src/components/client/buttons/AuthUserProfileButton.tsx:93,100` — header avatar + label. `<OglasinoAvatar … displayName={user.displayName} />` and `{user?.displayName}` rendered next to it. The avatar fallback uses `displayName` to compute initials.
  - `src/components/owner/client/NavUser.tsx:18,23` — owner sidebar `NavUser` element. Same `user.displayName` source.
- **Cache-revalidation pattern that flows the persisted backend value into the store on first SSR refresh.**
  - There is **no** SSR-tag specifically for "current user's displayName during the register window." The `/auth/firebase-sync` call is a non-SSR `BACKEND_API.post` invoked client-side; its response is the direct source of truth for `useAuthStore.user`.
  - Backend → web `revalidateTag('user:<id>')` invalidation (decisions.md 2026-05-19; `app/api/revalidate/route.ts:85`) covers SSR fetches that consume `UserInfoDTO` via `fetch().next.tags` — e.g., public profile pages. That tag does **not** rehydrate `useAuthStore.user` (the actor's own session state); the listener is. So even if the backend were to revalidate-tag after persisting the displayName, the actor's header avatar would still see the value the listener received from the firebase-sync response.
  - **Implication for the fix:** as long as the firebase-sync response carries the just-persisted `displayName` back to the client (which it already does, via `AuthUserDTO.displayName`), the post-register UI surfaces the correct name immediately. No additional revalidate-tag plumbing needed on the web side for the actor's own surface.

---

## Trust-boundary check (conventions Part 11)

- `displayName` is client-supplied input. Pure pass-through from form → store → service → backend POST body. The web makes no trust decisions on it (no moderation gate, no authorization gate, no state-transition gate is conditioned on `displayName` on the web side).
- Confirmation that the backend treats it as client-supplied input is **out of scope for this audit** — the backend audit verifies independently.
- No "previous value" / "before-state" fields are introduced on the web side. The proposed extension carries only the new value. Conforms to Part 11 ("Client-supplied 'before' or 'previous' values for change detection are forbidden").

---

# Recommended fix (web side)

## Fix shape

1. **Widen `registerUserFirebase` to accept `displayName`, and forward it to `syncUserToBackend` on the first POST.**

   Two web-side touchpoints in the action layer:

   - **Option A (preferred — surgical, matches existing seam):** widen `syncUserToBackend(firebaseUser)` to `syncUserToBackend(firebaseUser, { displayName? })`, where `displayName` is included in the POST body when present. The default omits the field, so the listener's own calls (token rotation, refresh) stay byte-identical. Only the register path passes it. Path:
     - `useAuthStore.register` passes `userData.displayName` into `registerUserFirebase(email, password, displayName)`.
     - `registerUserFirebase` calls `createUserWithEmailAndPassword(auth, email, password)`, then immediately calls `syncUserToBackend(res.user, { displayName })` — replacing today's listener-driven path **only for the register branch**. The `UseTokenRefresh` listener will then see a user whose backend record already carries the displayName, so its own subsequent firebase-sync POST (the one without the `displayName` field) is harmless (backend would no-op the field).

     This requires the listener-is-sole-hydrator contract to remain intact for token rotation / refresh / OAuth sign-in. The register branch becomes a documented exception: a single "create-with-displayName" POST during `createUserWithEmailAndPassword`'s onIdTokenChanged firing, riding through the existing listener path. Effectively this still keeps **one** firebase-sync POST per register (Firebase fires `onIdTokenChanged` once on user creation); we just need the firebase-sync POST in that single firing to include `displayName`.

   - **Option B (simpler signature):** widen `syncUserToBackend` permanently to accept an optional `displayName` and source it from a transient store field set by `register` immediately before `createUserWithEmailAndPassword`. The listener reads the field on its single firing post-register, clears it, and includes it in the POST. Symmetric with how `deletionInFlight` is used today (transient flag the listener reads). Adds a store field; pays a small recurring cost for an obviously single-shot signal — leans heavier than Option A.

   - **Option C (compose at call site):** keep `syncUserToBackend` unchanged. Have the register-specific path bypass the listener for the very first firebase-sync POST by calling a new `syncUserOnRegister(firebaseUser, displayName)` that POSTs `{ allowPreferenceCookies, displayName }` directly, and gate the listener's subsequent firing on a transient flag (à la `deletionInFlight`). More moving parts. Strictly worse than Option A.

   **Recommendation: Option A.** It widens one signature (`syncUserToBackend`), keeps the listener as the sole hydrator, and avoids store state for a single-shot register value. The implementation change is roughly:

   ```ts
   // authService.ts (proposed)
   export const syncUserToBackend = async (
     firebaseUser: FirebaseUser,
     extras?: { displayName?: string }
   ): Promise<AuthUserDTO | null> => {
     ...
     const response = await BACKEND_API.post<AuthUserDTO>('/auth/firebase-sync', {
       allowPreferenceCookies: globalCookie && globalCookie.cookieConsent?.preference,
       ...(extras?.displayName !== undefined ? { displayName: extras.displayName } : {}),
     });
     ...
   };

   // useAuthStore.register (proposed)
   register: async (userData: RegisterUserRequestDTO) => {
     try {
       set({ loading: true, error: null });
       await registerUserFirebase(userData.email, userData.password, userData.displayName);
     } ...
   },

   // authService.ts registerUserFirebase (proposed)
   export const registerUserFirebase = async (
     email: string,
     password: string,
     displayName: string,
   ): Promise<void> => {
     const res = await createUserWithEmailAndPassword(auth, email, password);
     await ensureUserInFirestore(res.user);
     // Listener will run syncUserToBackend on the resulting onIdTokenChanged firing;
     // it needs `displayName` in that POST. Options for plumbing covered in audit.
   };
   ```

   The listener-side plumbing is the one judgment call. Two viable routes:
   - **A1.** Have `register` synchronously call `syncUserToBackend(res.user, { displayName })` itself (skipping the listener for this single firing), then let the listener fire its own POST after — the listener's POST will be `displayName`-less but the backend already has the value, so it's a harmless echo. Requires verifying the listener does not regress the displayName by re-reading and re-persisting (backend audit territory).
   - **A2.** Stash `userData.displayName` in a transient closure visible to the next listener firing (e.g., a `nextRegisterDisplayName` cell module-scoped in `authService.ts`, set by `registerUserFirebase` and consumed-and-cleared by `syncUserToBackend` on the next call). Single-use, then nulled.

   A2 is cleaner because it preserves the listener-is-sole-hydrator design without exception. A1 introduces a documented exception that breaks the recently-established design rule. **Sub-recommendation: A2** — module-scoped one-shot cell. Mastermind to confirm.

2. **Client-side validation on `displayName` at register — mirror the edit-profile path.** Today both paths only check non-empty. Recommend:
   - Trim before checking emptiness on both paths (a stray space currently passes).
   - Match length bounds with whatever the backend enforces on `users.display_name` (backend audit will name the column constraint). If the backend caps at e.g. 60 chars, the client should pre-check the same bound.
   - Keep the keys aligned with namespaces: `tValidation('username.empty')` on register and `tError('username.required')` on edit are inconsistent today. Pick one — per conventions Part 6, new error-like keys belong in `ERRORS`; `VALIDATION` is frozen. Suggest migrating the register path to `tError('username.required')` for consistency with edit, in the same brief that does the fix. (Or punt to a follow-up brief — flagged for Mastermind.)

3. **No SSR cache-revalidation plumbing needed on the web side.** The actor's `useAuthStore.user` rehydrates directly from the firebase-sync response, which already returns `AuthUserDTO.displayName`. The existing `revalidateTag('user:<id>')` channel from the backend handles other viewers' SSR-cached `UserInfoDTO`. Both paths converge without new web-side wiring.

## Trust-boundary check confirmation

- `displayName` is treated as **client-supplied input** on the web. Pure pass-through.
- The web makes no trust decisions on it. The backend audit confirms its side independently.
- No "previous" / "old" / "before" field is needed or proposed.

## Open questions for Igor / Mastermind

1. **Plumbing route A1 vs A2** (or another option). The listener-is-sole-hydrator design is two days old (2026-05-19); I prefer A2 to preserve it. Either is implementable in a single web session.
2. **Client-side validation bounds.** The audit deliberately did not invent length / charset rules — those should mirror the backend constraint that the backend audit names. Once the backend's `users.display_name` constraint is on paper, the web brief can target the same bound.
3. **Namespace alignment.** Register currently uses `tValidation('username.empty')` (frozen namespace per Part 6). Migrate to `tError('username.required')` in the fix brief, or punt to a follow-up sweep? The frozen-namespace rule explicitly allows existing keys to stay until post-launch, so this is a judgment call.
4. **Post-register window UX.** Before the firebase-sync POST resolves, the header `AuthUserProfileButton` reads `user?.displayName` from the store. During the register handshake, `useAuthStore.user` is `null` (the listener hasn't set it yet), so the avatar button is gated by `isAuthenticated()` anyway and the question may be moot. Worth a quick visual confirmation when implementing — does the header flicker through an unauthenticated state during the ~few hundred ms of `createUserWithEmailAndPassword + onIdTokenChanged + firebase-sync`? Not a blocker for the fix; flagging for QA awareness.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected:
    - A new "transient register state" slice on `useAuthStore` (Option B in the fix shape) — rejected in favour of a module-scoped one-shot cell (A2) because the value is single-use and never reads in React.
    - A new top-level `displayName` validator (length + charset) introduced in this audit — rejected pending the backend audit's canonical bound; better to define the constraint once, on both repos, than to invent a web-only bound that then drifts.
  - Simplified or removed: nothing — read-only.

- **Adjacent observations (Part 4b):**
  - **Inconsistent translation keys for the same required-displayName error.** Register uses `tValidation('username.empty')` (VALIDATION — frozen); edit uses `tError('username.required')` (ERRORS). File paths: `src/components/popups/dialogs/RegisterDialog.tsx:70`, `app/[locale]/owner/user/page.tsx:137`. Severity: low (cosmetic; both work, just inconsistent). Suggested resolution: migrate register to `tError('username.required')` as part of the implementation brief, or schedule as a separate sweep. I did not fix this because it is out of scope (read-only audit).
  - **Firestore `users/<uid>.displayName` is `''` for email+password signups.** `ensureUserInFirestore` in `src/lib/service/reactCalls/authService.ts:78-85` writes `displayName: firebaseUser.displayName || ''` to the Firestore doc. For OAuth, the provider populates it; for email+password, `firebaseUser.displayName` is always `null`, so the Firestore doc gets `''`. I traced the chat surface (`src/messages/store/useChatStore.ts:159,213` → `getUserForFirebaseUid` → backend `/auth/firebase/<firebaseUid>`) — chat reads `displayName` from the **backend**, not Firestore, so this is a dead-field condition for the displayName on the Firestore doc rather than a user-visible bug. Severity: low (correctness gap with no user-visible effect today; could mislead a future engineer who assumes Firestore is the source of truth for chat peer names). I did not fix this because it is out of scope. Worth flagging in `issues.md` if Mastermind wants it tracked.
  - **No `trim()` on `displayName` at submit on either path.** A single space passes the required check today on both register and edit. Severity: low. Mention in the fix brief; trivial to add alongside the length validation that mirrors the backend.

- **Drafted config-file text:** none. The existing 2026-05-19 `issues.md` entry already tracks this work; the audit's findings inform the fix brief, not a config-file edit.

- **No `Brief vs reality` section needed.** The brief's premise matched the code at every checkpoint I tested (form fields, removed patch, firebase-sync body shape, edit-profile symmetry, listener-as-sole-hydrator design, no Firebase Auth `updateProfile` on edit). No discrepancy worth challenging.
