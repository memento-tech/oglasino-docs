# User deletion — auth lifecycle audit (web)

**Date:** 2026-05-18
**Branch:** `feature/user-deletion`
**Repo:** `oglasino-web`
**Scope:** read-only audit answering Q-1 through Q-6 of the brief at `.agent/brief.md`, confirming the assumptions of `oglasino-docs/features/user-deletion-auth-contract.md` against the code on disk.

No code changes. No tests modified. The only new files this session are this audit deliverable and the paired session-summary files.

---

## Summary

All six contract clauses inspected (C-5 through C-8 directly; C-3/C-4/C-9/C-10 surface only as backend concerns and are out of scope here) are implementable against the code on disk with **small, surgical diffs**. The contract's read of the existing code is accurate on every claim the web repo can confirm. Three points need to be flagged:

1. **C-5 wording is slightly misaligned with reality.** The contract names a **new** `clearFirebaseTokenCookie()` helper. The existing `writeFirebaseTokenCookie(null)` already does exactly what C-5 needs — it awaits the `/api/auth/token` POST response, which sends a `Set-Cookie` delete header. Calling `await writeFirebaseTokenCookie(null)` from the dialog satisfies C-5 without introducing a new helper. The route handler is `cookies().delete(COOKIE_NAME)` (Next.js cookie API), not a max-age=0 / expires-in-past header authored by us — but the effect on the browser is the same (Next.js sets `Set-Cookie: firebase_token=; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT`).
2. **C-7's axios interceptor reset is mechanically clean and adds ~6 lines.** `auth` is already imported in `api.ts`; `cachedToken` and `tokenExpiry` are the only two module-scoped state. No SSR ambiguity — the file is the client-side instance and is loaded once per browser session. The subscription belongs outside `createApiInstance` (so it's not re-registered if the factory were ever called again).
3. **C-8 holds today by luck-of-construction, not by explicit guard.** `fetchApi.ts` reads the cookie via `cookieStore.get('firebase_token')?.value` and conditionally spreads `...(firebaseToken ? { Authorization: 'Bearer ' + firebaseToken } : {})`. The guard is truthy-check, which folds `undefined`, `null`, and `''` into the same "skip" branch — so the wire is safe. But the type leaks `string | undefined` only; a future refactor that swaps to `?? ''` or to a non-optional read would silently re-introduce the bug. Worth keeping C-8 explicit in the contract as the contract already does. The literal string `"null"` is the one edge that **would** send `Authorization: Bearer null` — confirmed by reading the code — but this is not a value the route handler ever writes (the handler `cookies.delete()`s on null; it does not write a literal `"null"`). So the path is closed in practice. C-5 + C-8 together hold.

Implementation impact summary at the end of §9 of the contract draft remains accurate. No surprise blockers. The audit is "green" for the contract.

---

## Q-1 — `/api/auth/token` route handler cookie-clear semantics

### 1.1 Route handler response shape on `{token: null}`

`app/api/auth/token/route.ts:6-29`:

```ts
export async function POST(request: NextRequest) {
  let body: { token?: string | null } | null = null;
  try {
    body = await request.json();
  } catch {
    return NextResponse.json({ ok: false, error: 'invalid-body' }, { status: 400 });
  }

  const token = body?.token ?? null;
  const cookie = await cookies();

  if (token === null || token === undefined || token === '') {
    cookie.delete(COOKIE_NAME);
  } else {
    cookie.set(COOKIE_NAME, token, {
      httpOnly: true,
      secure: true,
      sameSite: 'lax',
      path: '/',
    });
  }

  return NextResponse.json({ ok: true });
}
```

For `{token: null}`:

- Body parses (line 9).
- `body?.token ?? null` → `null` (line 14).
- `token === null` matches the if-branch (line 17).
- `cookie.delete(COOKIE_NAME)` runs (line 18).
- Response is `200 OK` with JSON body `{ok: true}` (line 28).

The body shape matters less than the `Set-Cookie` header. See 1.2.

### 1.2 Effective cookie-clear `Set-Cookie` header

Next.js's `cookies().delete(name)` is implemented as a `Set-Cookie` header with the same name, an empty value, `Path=/`, and `Expires=Thu, 01 Jan 1970 00:00:00 GMT` (per Next.js `cookies()` API contract — `cookies.delete()` emits a delete-cookie directive on the response). This matches the "expires in the past" form C-1 calls out.

There is no `Max-Age=0` form here; Next.js uses `Expires` in the past. Both are spec-compliant ways to clear a cookie and the browser treats them identically. The contract's "either `expires` in the past, or `max-age=0`, or both" wording covers this.

The route handler does not set additional cookies on the `null` branch (no CSRF cookie, no session marker). The only cookie touched is `firebase_token`. The `cookie.set` branch (line 20-25) sets the cookie with `httpOnly`, `secure`, `sameSite: 'lax'`, `path: '/'`. Same name on both branches, so the `Set-Cookie` directives on the clear branch are sufficient to fully evict the cookie regardless of which path the prior value was set on (Path matches `/` on both write and delete).

### 1.3 Browser timing — await vs cookie applied

The audit cannot observe the browser's behavior at runtime, only the request-response shape. The relevant constraint:

- The fetch's `await` resolves when the response **headers are parsed** by the JS runtime (per the Fetch spec, the `Promise<Response>` resolves once the response head has been received, then `response.json()` reads the body separately).
- `Set-Cookie` is processed by the browser's network stack **before** the response head is delivered to JS — the browser's cookie jar is updated as the headers are parsed.
- Therefore, `await writeFirebaseTokenCookie(null)` resolving implies the cookie is already gone from the browser's perspective for any subsequent request originating in this tab.

The Next.js route handler returns `NextResponse.json({ok: true})` (line 28). `NextResponse.json` is the standard wrapper that flushes status + headers + body together in normal serverless / Node.js mode. There is **no streaming or chunked-flush shape** in this route handler that would let the headers arrive after the body — both arrive in the same response.

`fetch('/api/auth/token', {method: 'POST', ...})` in `writeFirebaseTokenCookie` (`src/lib/service/reactCalls/authTokenCookie.ts:8-14`):

```ts
export async function writeFirebaseTokenCookie(token: string | null): Promise<void> {
  await fetch('/api/auth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ token }),
  });
}
```

The `await` is on `fetch()`, which resolves when the response head is in. The browser's cookie jar is updated **before** that resolution. So `await writeFirebaseTokenCookie(null)` resolving means the `firebase_token` cookie is cleared from the browser's perspective for the next outbound request.

C-5's claim ("the route handler returns 200 with a `Set-Cookie` clearing header **before** the await resolves on the client") is correct. The cookie-clear is synchronous from the JS layer's perspective.

### 1.4 Additional cookies set on null branch

None. The handler does not set CSRF, session, or any other cookie. The clear branch only deletes `firebase_token`. The set branch only sets `firebase_token`.

The dialog only needs to clear `firebase_token`. No companion cookies need clearing in the same step.

### 1.5 Existing helper — is a new `clearFirebaseTokenCookie()` necessary?

**No new helper is needed.** `writeFirebaseTokenCookie(null)` already exists and is exported from `src/lib/service/reactCalls/authTokenCookie.ts:8`. It is the same network call (`POST /api/auth/token` with body `{token: null}`) the contract proposes for `clearFirebaseTokenCookie()`.

Current consumers of `writeFirebaseTokenCookie`:

- `src/components/client/initializers/UsetTokenRefresh.tsx:15` — clears the cookie on `onIdTokenChanged(null)`.
- `src/components/client/initializers/UsetTokenRefresh.tsx:22` — writes the cookie on token rotation.
- `src/lib/service/reactCalls/authService.ts:130` — writes the cookie on `syncUserToBackend`.

The deletion dialog can call `await writeFirebaseTokenCookie(null)` between `await auth.signOut()` and `onClose()`/`router.replace`. No new helper, no new file, no rename.

This is a small contract-vs-reality wording note, not a blocker. The contract draft introduces `clearFirebaseTokenCookie()` as a separate helper name; the engineer should either alias it (`export const clearFirebaseTokenCookie = () => writeFirebaseTokenCookie(null)`) for readability, or call `writeFirebaseTokenCookie(null)` directly. Either resolves C-5; the alias is a one-line file that costs nothing but a name.

### Verdict on C-5

**C-5 is implementable as drafted.** The single new line in the dialog (`await writeFirebaseTokenCookie(null)` or its alias) does what the contract requires. Browser cookie-clear semantics are correct: `await` resolving means cookie cleared. The wording "new helper `clearFirebaseTokenCookie()`" is a contract-style choice (clarity at call site) rather than a need-driven new abstraction. Recommend the contract draft updates §4 C-5 to read "`await clearFirebaseTokenCookie()` — a small wrapper over the existing `writeFirebaseTokenCookie(null)`" so the next engineer doesn't reach for a new network surface.

---

## Q-2 — `UseTokenRefresh` exact lifecycle

### 2.1 The handler today

`src/components/client/initializers/UsetTokenRefresh.tsx:1-30` (note: filename has the typo "Uset" — file is real, the typo is in the filename and the import path at `AppInit.tsx:13`):

```tsx
export default function UseTokenRefresh() {
  useEffect(() => {
    const auth = getAuth();

    return onIdTokenChanged(auth, async (firebaseUser) => {
      if (!firebaseUser) {
        await writeFirebaseTokenCookie(null);
        return;
      }

      const token = await firebaseUser.getIdToken();
      const globalCookie = getGlobalCookie();

      await writeFirebaseTokenCookie(token);
      await BACKEND_API.post('/auth/firebase-sync', {
        allowPreferenceCookies: globalCookie && globalCookie.cookieConsent?.preference,
      });
    });
  }, []);

  return <></>;
}
```

Mounted from `AppInit.tsx:28` inside the `(portal)` tree; `AppInit` is mounted once from `app/[locale]/layout.tsx:43`. Singleton per route tree per browser session.

### 2.2 Sequence on `firebaseUser` non-null (token rotation or initial mount with signed-in user)

1. `await firebaseUser.getIdToken()` — cached form (no force-refresh). Returns a fresh token if Firebase rotated; otherwise returns the cached ID token (line 19).
2. `getGlobalCookie()` — synchronous read of the consent preference cookie (line 20).
3. `await writeFirebaseTokenCookie(token)` — POST `/api/auth/token` with the new token; awaits the cookie write (line 22).
4. `await BACKEND_API.post('/auth/firebase-sync', {allowPreferenceCookies})` — awaits the backend sync (line 23-25).

Both side-effects are awaited. The handler is async but `onIdTokenChanged` does not await its return.

### 2.3 Sequence on `firebaseUser` null (sign-out)

1. `await writeFirebaseTokenCookie(null)` — POST `/api/auth/token` with `{token: null}`; awaits the cookie delete (line 15).
2. `return` — no `firebase-sync` POST on signout (line 16).

Confirmed: **no `firebase-sync` POST fires on signout.** The early `return` ensures only the cookie clear runs.

### 2.4 `firebase-sync` consumption of response

Line 23-25 does `await BACKEND_API.post(...)` but discards the returned promise's resolved value. No reading of the response body in this handler. The store does not get a fresh `AuthUserDTO` written from `UseTokenRefresh` — that responsibility lives in `useAuthStore.initAuthListener` (`useAuthStore.ts:244-257`, via `syncUserToBackend`).

However, the **response interceptor** in `api.ts:25-31` reads `x-account-restored` on every response — so a `firebase-sync` POST that triggers backend restoration **does** flip `useAuthStore.setRestored(true)`. That's the restoration trigger today (per spec §14.5).

### 2.5 C-6 implementation sketch

Per C-6: add `deletionInFlight: boolean` + `setDeletionInFlight(value)` to `useAuthStore`. Set true in the dialog before `getIdToken(true)`, clear in `finally`. `UseTokenRefresh` reads the flag and skips the `firebase-sync` POST when true.

In the current handler shape, the conditional skip slots cleanly between the cookie write and the sync POST:

```tsx
return onIdTokenChanged(auth, async (firebaseUser) => {
  if (!firebaseUser) {
    await writeFirebaseTokenCookie(null);
    return;
  }

  const token = await firebaseUser.getIdToken();
  await writeFirebaseTokenCookie(token);

  if (useAuthStore.getState().deletionInFlight) {
    return;
  }

  const globalCookie = getGlobalCookie();
  await BACKEND_API.post('/auth/firebase-sync', {
    allowPreferenceCookies: globalCookie && globalCookie.cookieConsent?.preference,
  });
});
```

- The cookie write still fires (must — it has to clear or update).
- The store read is `useAuthStore.getState().deletionInFlight` (no subscription needed; the read inside a handler runs at fire time).
- No refactor of try/catch or function signature; just one if-block.

The shape accepts the conditional skip cleanly. No restructuring of `UseTokenRefresh` is required.

**Caveat to flag:** the `getIdToken(true)` call inside `DeleteAccountConfirmationDialog` (line 92) fires `onIdTokenChanged` synchronously inside Firebase's SDK as part of the token mint. That means the handler may run **between** `setDeletionInFlight(true)` (proposed pre-`getIdToken`) and the dialog's `try`-wrapped sync POST. If `setDeletionInFlight(true)` is set before `getIdToken(true)`, the flag is true at handler-fire time and the skip works. If set after, the handler fires with the flag still false and we get a redundant `firebase-sync` POST anyway. **The contract's "set to true immediately before `getIdToken(true)`" wording is load-bearing for the hygiene C-6 promises.** Recommend the engineering brief restate this precisely so the engineer doesn't move the call.

### 2.6 Mount-time race vs `initAuthListener`

`AuthInit` (`src/components/client/initializers/AuthInit.tsx:7-20`) calls `useAuthStore.getState().initAuthListener()` once on mount. `initAuthListener` subscribes via `listenAuthState` → `onAuthStateChanged(auth, ...)` (per `authService.ts:199-200`) which fires for the initial auth state and on every change.

`UseTokenRefresh` subscribes via `onIdTokenChanged`. The two listeners fire on overlapping events:

- On initial mount with a signed-in user, both `onAuthStateChanged` and `onIdTokenChanged` fire (Firebase emits both for the cached user).
- On login, both fire (after `signInWithEmailAndPassword` completes).
- On token rotation (every ~hour), **only** `onIdTokenChanged` fires.
- On signout, both fire (Firebase emits both with `null`).

This produces the back-to-back POSTs the audit §1 noted (and Brief F's For Mastermind item 5 re-flagged):

- `initAuthListener` → `syncUserToBackend` → `writeFirebaseTokenCookie(token)` + `firebase-sync` POST.
- `UseTokenRefresh.onIdTokenChanged` → `writeFirebaseTokenCookie(token)` + `firebase-sync` POST.

Two `firebase-sync` POSTs in the first ~second of mount. This is **correctness-irrelevant** — both POSTs read the same Postgres state and produce the same `AuthUserDTO` (or both restore the same PENDING_DELETION user, which is idempotent since the second call sees ACTIVE). It is **wasted network**, not a bug.

Under C-3 + C-6, the noise is further reduced: during the deletion-in-flight window, `UseTokenRefresh` skips its sync POST. `initAuthListener`'s sync POST is the one that drives the store's `user` write, so it cannot be skipped without a deeper refactor of how the deletion flow signs the user out (which is out of scope for the contract).

Adjacent observation, not load-bearing for the contract.

---

## Q-3 — Axios interceptor `cachedToken` lifecycle

### 3.1 Module-scoped variables in `api.ts`

`src/lib/config/api.ts:73-74`:

```ts
let cachedToken: string | null = null;
let tokenExpiry = 0;
```

Also relevant:

- `BACKEND_API_URL` (line 6) — env var, read once at import.
- `storedLocale` (line 8) — set by `setLocale(locale)` from `AppInit.tsx:19`. Header-stamping only, not auth-related.
- `BACKEND_API` (line 71) — the exported axios instance; module-singleton.

Auth-related module state is **only** `cachedToken` and `tokenExpiry`. No other auth caches in this file.

### 3.2 Request interceptor flow

`src/lib/config/api.ts:76-107`:

```ts
BACKEND_API.interceptors.request.use(async (config) => {
  const user = auth.currentUser;

  const { tenant, oglasinoLocale: lang } = getTenantLocale(storedLocale);

  config.headers.set('X-Base-Site', tenant);
  config.headers.set('X-Lang', lang);

  if (!user) {
    return config;
  }

  if (config.headers.Authorization) {
    return config;
  }

  const now = Date.now();

  if (!cachedToken || now > tokenExpiry) {
    const result = await user.getIdTokenResult();
    cachedToken = result.token;
    tokenExpiry = new Date(result.expirationTime).getTime() - 60000;
  }

  config.headers.Authorization = `Bearer ${cachedToken}`;
  return config;
});
```

Step-through:

1. Read `auth.currentUser` (line 77).
2. Stamp `X-Base-Site` and `X-Lang` headers from `storedLocale` (line 79-82). These are unconditional and run even for anonymous requests.
3. If no current user, return config without Authorization (line 84-86). This is the clean short-circuit. **No crash, no stale**, because `auth.currentUser?.getIdToken(...)` is only attempted in the path where `user` is truthy.
4. If a caller already set Authorization (the user-deletion fresh-token path; see `userService.ts:118-128`), respect it (line 93-95).
5. If `cachedToken` is empty or expired, call `await user.getIdTokenResult()`, store the token and the expiry minus 60s (line 99-103). The `getIdTokenResult()` form is a Firebase SDK helper that returns the cached token unless it's actively expired, in which case Firebase auto-rotates.
6. Set `config.headers.Authorization = 'Bearer <cachedToken>'` (line 105).

### 3.3 What happens when `auth.signOut()` fires?

Nothing inside `api.ts` resets `cachedToken` or `tokenExpiry`. The variables remain set to the previous user's token until their natural expiry (≤1 hour from issue, typically refreshing 60s before expiry).

The request interceptor's `if (!user) return config` branch (line 84) **does** prevent a stale token from being attached **on requests fired after `auth.currentUser` becomes null**. So practically, after `auth.signOut()`:

- The next request hits `auth.currentUser === null`, short-circuits, sends no Authorization. ✓
- `cachedToken` stays stale in module scope. No one reads it because of the short-circuit. ✗ (memory hygiene only)
- If the user signs back in (different account or same account) within the previous token's expiry window, `auth.currentUser` becomes non-null and the interceptor's `cachedToken` check at line 99 (`!cachedToken || now > tokenExpiry`) sees a token that is still within its `tokenExpiry`. It would re-use the **previous user's token** for one request. This is the latent bug C-7 protects against.

Worst-case observable today: after `auth.signOut()`, a new user signs in within the prior token's TTL minus 60s; the first in-memory `BACKEND_API` request after the new sign-in attaches the prior user's `Authorization: Bearer <old_token>`. The backend's `FirebaseAuthFilter` verifies the old token, loads the **old user's** Redis cache entry, and treats the request as the prior user. This is a real cross-account misattribution risk on shared devices or quick session swaps. Not a deletion-specific bug; same surface that C-7 closes.

### 3.4 C-7 implementation sketch

Per C-7: subscribe to `onIdTokenChanged` at module-init time; on null user, reset `cachedToken` and `tokenExpiry`.

- The `auth` import is already at `api.ts:4` (used by the existing `error.response.status === 403` branch at line 54 and the request interceptor at line 77). ✓
- Adding `import { onIdTokenChanged } from 'firebase/auth'` and subscribing at module-init is straightforward:

```ts
import { auth } from './firebaseClient';
import { onIdTokenChanged } from 'firebase/auth';
// ... existing code ...

let cachedToken: string | null = null;
let tokenExpiry = 0;

onIdTokenChanged(auth, (user) => {
  if (!user) {
    cachedToken = null;
    tokenExpiry = 0;
  }
});
```

- Place the subscription outside `createApiInstance` so it only runs once at module load, not per-instance creation (currently there's only one instance, but the factory is exported pattern). Closure capture is clean — `cachedToken` and `tokenExpiry` are in the same module scope as the subscription.
- The subscription belongs to a singleton client-side module that lives for the browser session. **Module-load subscriptions never unmount.** This is fine for a singleton like `api.ts` (the file is imported once per browser session and never re-evaluated); the subscription's lifetime equals the page's lifetime. Same pattern as other Firebase setup that lives at module scope in `firebaseClient.ts`.
- HMR consideration in dev: Next.js's webpack HMR may re-evaluate the module on edit, which would register a duplicate listener. Firebase's `onIdTokenChanged` returns an unsubscribe function; storing it in a module variable so HMR can call it on `if (module.hot)` is the conservative move. Brief E's "Seams" did not surface duplicate listeners as observed dev pain, so a simple `onIdTokenChanged(auth, ...)` without unsubscribe handling is acceptable for v1 — the duplicate listener fires twice, both setting the same variables to the same values, so the behavior is correct just slightly wasteful in dev. If the engineer wants to be tidy, store the unsubscribe in a module variable.

### 3.5 Existing 403 USER_BANNED branch

`src/lib/config/api.ts:50-62` already handles `403 + USER_BANNED` globally per Brief A — `auth.signOut()`, `sessionStorage.setItem('account-banned', '1')`, never-resolving promise. Confirmed present and matches spec §14.12. The contract does not touch this branch; C-7 is additive.

The interceptor today does **not** read `cachedToken` on the error path — only on the request path. So the prior-user-token leak path is request-only (a stale token gets attached to a new request after signout), not response-driven. C-7's reset on signout closes the only path.

---

## Q-4 — SSR `fetchApi.ts` cookie handling

### 4.1 Cookie read path

`src/lib/config/fetchApi.ts:29-60`. The relevant slice:

```ts
let cookieHeader = '';
let firebaseToken: string | undefined;
if (!skipAuth && typeof window === 'undefined') {
  const cookieStore = await cookies();
  firebaseToken = cookieStore.get('firebase_token')?.value;
  const allCookies = cookieStore.getAll();
  if (allCookies.length > 0) {
    cookieHeader = allCookies.map((c) => `${c.name}=${c.value}`).join('; ');
  }
}

try {
  const response = await fetch(`${BASE_URL}${endpoint}`, {
    method,
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json',
      ...headers,
      ...(cookieHeader ? { Cookie: cookieHeader } : {}),
      ...(firebaseToken ? { Authorization: `Bearer ${firebaseToken}` } : {}),
    },
    ...
```

The cookie value is read at line 40: `cookieStore.get('firebase_token')?.value`. The `?.value` chain returns `string | undefined`:

- Cookie absent → `cookieStore.get('firebase_token')` returns `undefined`; the optional chain short-circuits to `undefined`.
- Cookie present, any value (including `""`) → `cookieStore.get('firebase_token')` returns `{name: 'firebase_token', value: <string>}`; the `.value` is the raw string.

`firebaseToken` is then either `undefined` or a `string`. The Authorization header is conditionally spread on **truthy `firebaseToken`** (line 55: `...(firebaseToken ? { Authorization: 'Bearer ' + firebaseToken } : {})`).

### 4.2 Behavior matrix

| Cookie state | `cookieStore.get(...)` returns | `?.value` | Truthy? | `Authorization` header sent? |
|---|---|---|---|---|
| Absent (no `firebase_token` in `Set-Cookie` jar) | `undefined` | `undefined` | no | **not sent** ✓ |
| Present, value `""` (empty string) | `{name, value: ''}` | `''` | no | **not sent** ✓ |
| Present, value `"null"` (literal string) | `{name, value: 'null'}` | `'null'` | **yes** | **sent: `Authorization: Bearer null`** ✗ |
| Present, value `<valid_jwt>` | `{name, value: <jwt>}` | `<jwt>` | yes | sent: `Authorization: Bearer <jwt>` ✓ |

**The literal string `"null"` is the one trap.** A request would send `Authorization: Bearer null` on the wire. The backend's `verifyIdToken("null")` would throw a `FirebaseAuthException`, which the filter would handle as an invalid token (likely 403 or filter drop, depending on backend behavior — not our scope to audit, but the symptom is "secure endpoint rejects with 401/403").

**However**, no code path in the web repo writes the literal string `"null"` into the cookie. The route handler at `app/api/auth/token/route.ts:14-26` handles the null/undefined/empty cases via `cookie.delete(COOKIE_NAME)` — it never calls `cookie.set(COOKIE_NAME, 'null', ...)`. The only `cookie.set` path takes the validated `token` string (which is a real JWT or the call wouldn't have happened — the dialog is the only post-Brief-C caller that passes a token, and that token comes from `getIdToken(true)`). So in practice, the literal-`"null"` cookie value is unreachable today.

C-8 holds today on the route-handler side. C-8 holds today on the `fetchApi.ts` side for `undefined` and `""`. The literal-`"null"` edge is **closed by construction**, not by an explicit guard in `fetchApi.ts`. A future refactor could re-open it. C-8's explicit statement in the contract is the right hedge.

### 4.3 Caching layers on the SSR call path

`fetchApi.ts` calls `fetch(`${BASE_URL}${endpoint}`, ...)` with `next: { revalidate, tags }` passed through from the caller. Caching is per-caller via `next.revalidate` and `next.tags`. The Next.js Data Cache key includes the URL, method, body, and headers.

Critical: **the Authorization and Cookie headers are part of the cache key.** So a request made with `Authorization: Bearer <token>` is cached separately from one with no Authorization. This means: after the cookie is cleared, the next SSR call sees "no cookie" and produces a fresh fetch, not a cache hit on a prior Authorized call. ✓

For `getPortalProducts` (called on the home page render at `page.tsx:34`): this is a `POST` to `/public/product/search` with **no `next.revalidate`** set in the call (see `productsSearchService.ts:94-106`) — so Next.js's Data Cache treats the POST as uncacheable (POSTs are not cached by Next.js's Data Cache by default; even with `revalidate`, POSTs are typically not deduped). Every render fires a fresh fetch. No risk of a stale-Authorization cache hit.

For `/public/baseSite/{tenant}` (called via `getBaseSiteServer`): the call passes `skipAuth: true` (line 21 in `getBaseSiteServer.ts`), so no Authorization is ever attached and the cookie is never read. Cache key carries no auth dimension. ✓

For `/public/translations` (called via `getNamespaceTranslations`): the call uses a raw `fetch()` and does not pass through `fetchApi.ts` (`translationsCache.ts:39-42`). It does not read cookies; only sends `X-Base-Site`. ✓

For `/public/config` (called via `getConfig` from `app/layout.tsx:45`): the call uses a raw `fetch()` and does not pass through `fetchApi.ts` (`getConfig.ts:7-12`). Does not read cookies; sends only the URL. ✓

**Conclusion:** the only SSR call that **could** carry a stale Authorization to the backend during post-deletion navigation is `getPortalProducts` → `/public/product/search`. This is precisely the path the 2026-05-18 backend logs showed firing during the race.

### 4.4 401 retry semantics

`fetchApi.ts`'s `request<T>` (lines 29-80) does **no retry** on 401 or any other status. It returns the response data with `success: false` and `errorCode` populated. The caller decides what to do with the failure (typically returns `null` or `EMPTY_OVERVIEWS`).

No closed-over Authorization value — each call is one network attempt, no retry loop. ✓

### 4.5 SSR callers fired during navigation to `/<locale>`

Reading the SSR fan-out from outer to inner for the home page `/<locale>`:

- **`app/layout.tsx:45`** — `getConfig()` → raw `fetch('/public/config')`, no cookies, no Authorization.
- **`app/[locale]/layout.tsx:29`** — `getBaseSiteServer()` → `FETCH_BACKEND_API.get('/public/baseSite/{tenant}', {skipAuth: true, ...})`. No Authorization, no Cookie header.
- **`app/[locale]/layout.tsx:30`** — `getAllBaseSitesOverviews()` → `FETCH_BACKEND_API.get('/public/baseSite/overviews', {skipAuth: true, ...})`. No Authorization, no Cookie header.
- **`app/[locale]/layout.tsx` → `setRequestLocale(locale)`** → triggers next-intl's `getRequestConfig` (`src/i18n/request.ts:7-23`) which calls `loadAllNamespaces` (`src/i18n/internalRequest.ts:11`) which fans out to every namespace via `getNamespaceTranslations`. Raw `fetch('/public/translations?namespace=X&lang=Y')`. No cookies, no Authorization. The first SSR call after deploy populates the `unstable_cache` for 24h; subsequent renders hit the cache.
- **`app/[locale]/(portal)/(public)/page.tsx:31`** — `getBaseSiteServer()` again (deduped by React `cache()` — same render-pass instance).
- **`app/[locale]/(portal)/(public)/page.tsx:33`** — `filterHydrationSSR(...)` → no backend call; pure parsing.
- **`app/[locale]/(portal)/(public)/page.tsx:34`** — **`getPortalProducts(filtersData, getInitialPage())`** → `FETCH_BACKEND_API.post('/public/product/search', ...)`, **no `skipAuth: true`**. This is the call that reads the `firebase_token` cookie and attaches `Authorization: Bearer <token>` if present.
- **`app/[locale]/(portal)/(public)/page.tsx:40`** — `cookies().get('globalCookie')` — local cookie read, no network call.

The `(portal)` layout (`app/[locale]/(portal)/layout.tsx`) and `(public)` layout (`app/[locale]/(portal)/(public)/layout.tsx`) do not fire any backend calls themselves (no async data fetching in their bodies).

**Single SSR call that participates in the race:** `getPortalProducts` → `/public/product/search`. Backend log evidence in §3 of the contract is consistent with this single call. Other "flurry of `/public/*` calls" in the trace are the translation namespaces (one per namespace) and base-site calls, all of which are `skipAuth` or raw `fetch` and do not carry the cookie.

### 4.6 Verdict on C-8

C-8 holds today by behavior. The cookie reading logic in `fetchApi.ts:38-45` correctly folds `undefined` and `""` into "no Authorization sent." The only edge that would break C-8 is a literal `"null"` cookie value, and no code path writes that today. C-8's explicit naming in the contract is the right hedge against a future refactor.

---

## Q-5 — `useAuthStore` shape and the `deletionInFlight` flag

### 5.1 Current fields on `useAuthStore`

`src/lib/store/useAuthStore.ts:61-95` (the `AuthState` interface):

State fields:

- `user: AuthUserDTO | null` (line 62)
- `loading: boolean` (line 63)
- `error: string | null` (line 64)
- `isAdmin: boolean` (line 70)
- `isAdminLoaded: boolean` (line 71)
- `isAdminLoading: boolean` (line 72)
- `restored: boolean` (line 77)

Action methods:

- `resetAuthError`, `setUser`, `setRestored`, `isAuthenticated`, `isLoading`, `loadIsAdmin`, `login`, `register`, `loginWithGoogle`, `loginWithFacebook`, `logout`, `initAuthListener`, `refreshUser` (lines 79-94)

The audit §1 listing of seven state fields matches what's on disk today.

### 5.2 The `set()` pattern

The store uses standard Zustand `set({...})` calls throughout — see `setUser: (user) => set({ user })` (line 110), `setRestored: (value) => set({ restored: value })` (line 112), and partial-update calls inside async actions like `set({ loading: true, error: null })` (line 136).

Adding a new field follows the same shape exactly:

```ts
interface AuthState {
  // ...existing fields...
  deletionInFlight: boolean;
  setDeletionInFlight: (value: boolean) => void;
  // ...existing methods...
}

export const useAuthStore = create<AuthState>()((set, get) => ({
  // ...
  deletionInFlight: false,
  setDeletionInFlight: (value) => set({ deletionInFlight: value }),
  // ...
}));
```

This is the same pattern as `restored` / `setRestored`. No special handling. No naming collision — grep for `deletionInFlight` across `src/` and `app/` returns zero matches.

### 5.3 Callers of `setDeletionInFlight(true)`

Per the contract:

- **`src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx`** — before `getIdToken(true)` (current line 92). File exists; the dialog has a clean async `handleDelete` (lines 51-114) with a single try block around the deletion logic (lines 88-113) and a `finally` block at lines 111-113 that already resets `loading`. Adding `setDeletionInFlight(true)` immediately before line 92 and `setDeletionInFlight(false)` inside the existing `finally` is one line each. The brief should call out **where** the `setDeletionInFlight(true)` lands — the contract says "immediately before `getIdToken(true)`" which is line 92; that's correct, but the engineer must not move it earlier (e.g., to before reauth at line 69) because reauth itself can trigger `onIdTokenChanged` and that handler would then check the flag too soon. Strictly: set the flag in the second try-block (the one starting line 88), one statement above line 92. Clear in the existing finally at line 111-113.

- **`src/components/client/initializers/UsetTokenRefresh.tsx`** — reads via `useAuthStore.getState().deletionInFlight` inside the `onIdTokenChanged` callback. See Q-2.5 sketch. Touchable without ceremony.

All three sites confirmed touchable. The contract's plan is sound.

### 5.4 Could `loading` double for `deletionInFlight`?

The store's `loading: boolean` represents **auth-store-action-in-progress** — it is set to `true` at the start of each action (login, register, refreshUser) and reset in `finally`. The `initAuthListener` flow also sets it (line 246) so it covers initial Firebase-listener bootstrap too.

Reuse problems:

- `loading` is reset by **every** auth action's `finally`. The deletion dialog does not go through `login`/`register`/`refreshUser` — it calls `getIdToken(true)` and `deleteCurrentUser` directly. If we repurposed `loading`, we'd need to set it from the dialog at the right moments — but then ` UseTokenRefresh.onIdTokenChanged` couldn't tell "the deletion flow is in flight" from "the user just clicked Login in another tab and the store is bootstrapping."
- `loading` is read by several consumers: `useAuthStore.getState().loading` is consulted in `refreshUser` (line 262) to dedupe concurrent calls; `SessionGuard` indirectly observes it; the auth selector pattern in components watches it.

Using a single boolean for two distinct concepts ("auth bootstrap" vs "deletion in flight") would couple unrelated flows. The contract's choice of a separate field is the cleaner shape. **Recommend keeping `deletionInFlight` as a dedicated field per the contract.**

### 5.5 Implementation impact

Two-line type addition, two-line implementation addition, two call sites (dialog + UseTokenRefresh). No new selectors, no migration shim, no consumer rewrite. The contract's "no ceremony" framing is accurate.

---

## Q-6 — `DeleteAccountConfirmationDialog` current success path

### 6.1 Current sequence after `deleteCurrentUser(freshToken)` returns 200

`src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:51-114`. The success path (after reauth succeeded; the second try-block at lines 88-113):

```tsx
try {
  const freshToken = await currentUser.getIdToken(true);                     // 92
  const { scheduledDeletionAt } = await deleteCurrentUser(freshToken);       // 94
  sessionStorage.setItem('account-just-deleted', scheduledDeletionAt);       // 98
  await auth.signOut();                                                       // 99
  onClose();                                                                  // 100
  router.replace(`/${locale}`);                                               // 101
} catch (error) {
  if (isErrorWithCode(error, 'REAUTH_REQUIRED')) {                            // 103
    setErrorMessage(tErrors('errors.reauth.required'));                       // 104
  } else if (isErrorWithCode(error, 'USER_LOCKED_FROM_DELETION')) {           // 105
    setErrorMessage(tErrors('errors.user.locked.from.deletion'));             // 106
    setLocked(true);                                                          // 107
  } else {
    setErrorMessage(tErrors('system.error'));                                 // 109
  }
} finally {
  setLoading(false);                                                          // 112
}
```

Order on the success branch: `getIdToken(true)` → `deleteCurrentUser` POST → `sessionStorage.setItem` → `auth.signOut()` → `onClose()` → `router.replace`. The `setLoading(false)` resets in `finally` regardless of success or error.

The Brief F fix from session 6 placed `onClose()` + `router.replace` after `auth.signOut()` so the dialog dismisses and navigates instead of relying on `SessionGuard`. That order is on disk and confirmed.

### 6.2 C-5 amendment — diff size

Per C-5 (and the spec §10 amendment in §14.3), the new sequence is:

1. `getIdToken(true)`
2. `deleteCurrentUser(freshToken)`
3. `sessionStorage.setItem`
4. `await auth.signOut()`
5. **`await clearFirebaseTokenCookie()` ← new (per Q-1.5, this is `await writeFirebaseTokenCookie(null)` or a thin alias)**
6. `onClose()`
7. `router.replace`

Diff:

- **One new line** between current line 99 and current line 100: `await writeFirebaseTokenCookie(null);` (or `await clearFirebaseTokenCookie();` if the alias is added).
- **One new import** at the top of the file (if not using an alias, importing `writeFirebaseTokenCookie` from `@/src/lib/service/reactCalls/authTokenCookie`; if using the alias, importing the alias instead).

No restructuring of the try/catch. The `catch` still discriminates `REAUTH_REQUIRED` / `USER_LOCKED_FROM_DELETION` / generic, and `finally` still resets `setLoading`. The new `await` lives inside the success-only path and would only run after the `deleteCurrentUser` POST has committed and `auth.signOut()` has resolved — so a failed delete doesn't accidentally clear the cookie and leave the user with a confusing logged-in-without-token state.

**One sub-question to surface:** if `writeFirebaseTokenCookie(null)` itself rejects (network blip), should the dialog still navigate? Today's catch would catch it as a generic error and show `tErrors('system.error')` — but at that point the deletion has already committed in Postgres. The user would see "system error" on a successful deletion. This is a low-probability edge but worth a guard: wrap the `await writeFirebaseTokenCookie(null)` in its own try/catch that swallows and logs, so navigation always proceeds after the deletion commits. C-3 (the backend half) makes a stale cookie non-load-bearing for correctness, so swallowing the cookie-clear failure is safe. Recommend the engineering brief call this out explicitly.

### 6.3 C-6 amendment — slot points

Per C-6, the new lines are:

- `setDeletionInFlight(true)` immediately before `getIdToken(true)` (current line 92).
- `setDeletionInFlight(false)` in the existing `finally` block (current line 111-113).

Exact insertions:

```tsx
try {
  useAuthStore.getState().setDeletionInFlight(true);     // NEW
  const freshToken = await currentUser.getIdToken(true);  // 92 (existing)
  // ...
} catch (error) {
  // ... (unchanged)
} finally {
  useAuthStore.getState().setDeletionInFlight(false);     // NEW
  setLoading(false);                                       // 112 (existing)
}
```

The `useAuthStore` import is not present in this file today (the dialog uses `auth.currentUser` from Firebase directly, not the auth store). Add `import { useAuthStore } from '@/src/lib/store/useAuthStore'`. The `getState()` form is the established pattern for non-reactive reads/writes (see `api.ts:28, 54` — same pattern).

### 6.4 `deleteCurrentUser` return shape

Per `src/lib/service/reactCalls/userService.ts:118-128`, `deleteCurrentUser` returns `Promise<{ scheduledDeletionAt: string }>` directly (the unwrapped `res.data`). The destructuring at dialog line 94 (`const { scheduledDeletionAt } = await deleteCurrentUser(freshToken)`) is correct. No wrapper.

If the request fails, the function rethrows (no defensive `try/catch` inside `deleteCurrentUser`). The dialog's outer catch at line 102 handles the error-code discrimination. Confirmed correct.

### 6.5 Brief F stuck-loading fix — error path behavior

Per Brief E Finding A (the stuck-loading bug) and Brief F's fix, the dialog's `setLoading(false)` lives in a `finally` block (line 111-113) so the spinner clears on every code path. Walking the error paths:

- **Reauth failure** (lines 79-86, catch on the reauth try): `setErrorMessage(...)` + `setLoading(false)` + `return` inline. The dialog stays open with error visible. No `finally` in this block because the function returns early.
- **`getIdToken(true)` failure**: caught by the outer try (line 88) `catch` at line 102. Falls through to the generic else (line 108) → `setErrorMessage(tErrors('system.error'))`. `finally` at 111-113 resets `setLoading`. Dialog stays open with system-error visible. ✓
- **`deleteCurrentUser` REAUTH_REQUIRED 403**: caught at line 102, `isErrorWithCode(error, 'REAUTH_REQUIRED')` matches at line 103, `setErrorMessage(tErrors('errors.reauth.required'))`. `finally` resets loading. Dialog stays open. ✓
- **`deleteCurrentUser` USER_LOCKED_FROM_DELETION 403**: caught at line 102, matches at line 105, sets error message + `setLocked(true)`. `finally` resets loading. Dialog stays open with error + delete-button disabled. ✓
- **`deleteCurrentUser` 5xx / network error**: caught at line 102, hits the generic else at line 108. `finally` resets loading. Dialog stays open with system-error. ✓
- **Success path**: lines 92-101 run, navigation happens, the dialog is unmounted by `onClose()` + route change. `finally` would still run before unmount but the dialog is no longer visible. No-op effectively. ✓

The Brief F fix holds for all error paths. The C-5 and C-6 amendments do not regress any of these — the new `setDeletionInFlight(false)` in `finally` and the new `await writeFirebaseTokenCookie(null)` in the success path are independent.

**Edge note:** if `writeFirebaseTokenCookie(null)` throws inside the success path (per 6.2's open question), the catch falls into the generic else and shows `system.error`. The deletion has already committed, the cookie may or may not be cleared, `auth.signOut()` already ran — the user sees a misleading error on a successful operation, with navigation never happening. The fix is to wrap the cookie-clear in a local try/catch that swallows. The engineer should be told this in the brief explicitly.

---

## Adjacent observations (Part 4b)

Items spotted during the audit that are auth-boundary-adjacent but outside the question set. None of these are deletion blockers; flagging for Mastermind's triage.

1. **`UseTokenRefresh` filename typo persists.** `UsetTokenRefresh.tsx` (note "Uset"). The component name is `UseTokenRefresh`. The brief Q-2 explicitly warned the audit to check the typo; confirming for the record that the file is `UsetTokenRefresh.tsx` and the import in `AppInit.tsx:13` matches that typoed name. Renaming the file would touch one import and one filename. Low severity; cosmetic but the kind of thing a future engineer searches for as `UseToken*` and misses. Not deletion-relevant.

2. **`api.ts` request interceptor stamps `X-Base-Site` / `X-Lang` even on anonymous requests.** Lines 79-82 always set the tenant/lang headers before the `!user` short-circuit at line 84. This is correct behavior (the backend's `CurrentLanguageFilter` requires the lang on non-allowlisted public routes), but worth noting that the interceptor's flow is "stamp identity-derived headers always, attach Authorization conditionally." A future refactor that moves the header stamps below the user-check would silently break public requests. Low severity. Already audited correctly.

3. **`fetchApi.ts` forwards **all** cookies to the backend via the `Cookie` header** (lines 41-44). This is wider than just `firebase_token` — `globalCookie`, locale cookies, Vercel session cookies, and anything else the browser drops on the request would be forwarded. The backend only consumes `firebase_token` for auth; the rest are inert. But it's a wider-than-needed information surface. Lifting `Cookie:` to only forward `firebase_token` (when not `skipAuth`) would be a tightening; not deletion-relevant. Low severity.

4. **`deleteCurrentUser` rethrows; the dialog's catch is the only handler.** The pattern at `userService.ts:118-128` is "no defensive catch in the service, the caller handles." Different from the `getUserDetails` pattern (`userService.ts:26-34`) which rethrows with a wrapper Error, and from `updateUser` (line 36-50) which swallows. The mix-and-match between services is pre-existing and not deletion-specific. Worth knowing the dialog is the sole handler; if a second caller emerged, both would need to discriminate the same codes. Low severity.

5. **`syncUserToBackend` calls `await firebaseUser.getIdToken()` then `await writeFirebaseTokenCookie(token)` unconditionally on every sync.** Lines 129-130. If the cookie is already current (very recent token), this is a redundant POST to `/api/auth/token`. With the two listeners both calling `syncUserToBackend` on overlapping events (audit §2.6), the first session second can produce **four** cookie-write POSTs in addition to two `firebase-sync` POSTs. Pre-existing wasted-network surface, not deletion-relevant. Already noted in Brief F. Low severity.

6. **`UseTokenRefresh` reads `globalCookie` synchronously inside the async handler** (line 20). `getGlobalCookie()` is a sync read of `document.cookie`; the handler is async but the cookie read is at the start of the function before any await. If the cookie isn't set yet (very early mount before any consent banner interaction), the call resolves with `undefined` and the POST sends `allowPreferenceCookies: undefined`. The backend likely tolerates this (or coerces to false), but it's an implicit contract. Low severity, not deletion-relevant.

---

## Open questions for Mastermind

Anything the audit couldn't resolve and that needs Mastermind's judgment before the engineering brief is final.

1. **C-5 wording: alias `clearFirebaseTokenCookie()` or use `writeFirebaseTokenCookie(null)` directly?** Per Q-1.5. Recommend the contract update to read: "`await clearFirebaseTokenCookie()` — a small alias over the existing `writeFirebaseTokenCookie(null)` for call-site readability." Both work; the alias is one extra line of code for a clearer dialog body. Mastermind to choose.

2. **Cookie-clear failure swallowing in the dialog's success path.** Per Q-6.2 and 6.5. If `writeFirebaseTokenCookie(null)` rejects (network blip), today's outer catch would treat it as a deletion error and show `system.error` even though deletion succeeded. Recommend wrapping the cookie-clear in a local try/catch that swallows and logs. Under C-3, the stale cookie cannot cause restoration, so swallowing is safe. Mastermind to confirm and add to the engineering brief.

3. **`UseTokenRefresh.deletionInFlight` flag placement.** Per Q-2.5. The contract's "set immediately before `getIdToken(true)`" is load-bearing because `getIdToken(true)` itself triggers `onIdTokenChanged` synchronously. The engineer must not move `setDeletionInFlight(true)` to before reauth (line 69) — that would let reauth's own `onIdTokenChanged` fire (which it normally does after a successful reauthenticate) without the flag set. Just affirm the contract's wording is the binding constraint.

4. **HMR consideration for the C-7 `onIdTokenChanged` subscription.** Per Q-3.4. The cheap v1 is "fire and forget; duplicate listener in dev is harmless." The tidy v1 is "store the unsubscribe in a module variable and call it on HMR." Negligible practical difference. Mastermind to pick the level of polish for the brief.

5. **`deletionInFlight` flag — should it live in `useAuthStore` or a smaller dedicated store?** Per Q-5.4. The contract puts it in `useAuthStore`. The alternative is a tiny standalone Zustand store for "auth flow state" — but the field is consumed exclusively by `UseTokenRefresh.onIdTokenChanged` and the dialog. Sharing the auth store is fine and matches how `restored` works. No real alternative to push back on; restating for the record.

6. **`/api/auth/token` route handler's body return shape on cookie-clear.** The handler returns `{ok: true}` (JSON). The dialog doesn't consume this body. If a future change wants the response to also carry "and here are the cookies that were cleared," the existing shape is sufficient — but the contract doesn't ask for that. No question for Mastermind; closing the loop on Q-1.1.

7. **The audit confirms the contract's six-surface model.** No seventh surface surfaced during the audit. The relevant additional client-side state stores — `useChatStore.userCache`, `useChatBlockStore`, `useViewTokenStore`, `notificationManager` — all hold per-user state but are not consulted by the auth filter or by `syncUserToBackend`. They are cleared on logout via `useAuthStore.logout` (which is called by `auth.signOut` indirectly via the `onAuthStateChanged` listener that dispatches `set({user: null})`). They do not carry the Firebase token. Confirming Q-7 of the contract: no seventh auth-state surface in the web code.

---

End of audit.
