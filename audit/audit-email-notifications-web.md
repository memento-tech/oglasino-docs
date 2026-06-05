# Audit — email-notifications (oglasino-web)

**Repo:** oglasino-web
**Branch:** dev (read-only; no code changed)
**Date:** 2026-06-02
**Scope:** Inventory the current email-registration / verification surface for the planned branded email-verification flow. Code is ground truth.

---

## TL;DR

- Email registration **logs the user straight in.** There is no "check your email / verify" UX anywhere today.
- Nothing in web calls Firebase `sendEmailVerification`.
- The web login gate (`SessionGuard`) checks only "is there a signed-in Firebase user" (+ admin for admin routes). It does **not** check `emailVerified`. An unverified user is today indistinguishable from a verified one and gets full access.
- No `/verify` route exists. Building one is a small, well-supported add: a public `[locale]` page reading `oobCode` from the query string and calling `applyActionCode` from `firebase/auth` (already a dependency, v12).
- **Brief vs reality on Section 4:** there is *no* password-reset flow in web at all — no `sendPasswordResetEmail`, no "forgot password" link, no reset route. The spec should record "web has no password-reset entry point today," not "Firebase-native, no change." Details in Section 4.

---

## Section 1 — Registration flow today

### The path, end to end

1. **Form:** `src/components/popups/dialogs/RegisterDialog.tsx`. A `useState` form (`displayName`, `email`, `password`) with client-side Zod-style structural checks (`validateEmail`, `validatePassword`, displayName length/pattern) and a reCAPTCHA gate. On submit it calls `register(userData)` from `useAuthStore`.
2. **Store action:** `useAuthStore.register` (`src/lib/store/useAuthStore.ts:197`). Sets `loading`, calls `registerUserFirebase(email, password, displayName)`, maps any error via `mapAuthError`, clears `loading`. It does **not** hydrate `user` itself.
3. **Service:** `registerUserFirebase` (`src/lib/service/reactCalls/authService.ts:179`):
   - Stashes `displayName` in the one-shot module cell `nextRegisterDisplayName`.
   - `createUserWithEmailAndPassword(auth, email, password)` — **this auto-signs the user in** (Firebase behaviour). No `sendEmailVerification` call follows.
   - `ensureUserInFirestore(res.user)` creates the `users/{uid}` + `userchats/{uid}` docs and registers the FCM token.
4. **Backend sync is listener-driven, not action-driven.** `createUserWithEmailAndPassword` triggers `onIdTokenChanged`, handled by the sole listener `UseTokenRefresh` (`src/components/client/initializers/UseTokenRefresh.tsx:47`). That listener writes the SSR token cookie, then calls `syncUserToBackend` → `POST /auth/firebase-sync` (`authService.ts:140`), threading the stashed `displayName` through that single POST. The returned `AuthUserDTO` is pushed into `useAuthStore.user` (`setUser`), and `track('sign_up', …)` fires (`wasRegister` distinguishes sign-up from login).

### Where the user lands

`RegisterDialog` has a `useEffect` on `isAuthenticated()` (`RegisterDialog.tsx:40`): the moment the store's `user` becomes non-null (i.e. the listener finished hydrating), it calls `onClose()` and `router.refresh()`. **Net effect: register → dialog closes → page soft-refreshes with the user fully logged in.** There is no intermediate "verify your email" screen, no blocking state, no toast. Identical shape to login.

### `sendEmailVerification`?

**No.** Grep across `src` + `app` for `sendEmailVerification` / `emailVerif*` / `verifyEmail` returns nothing. Web never asks Firebase to send a verification email, and never reads `firebaseUser.emailVerified`.

---

## Section 2 — The login gate, web side

### What gates protected routes

`app/[locale]/(portal)/(protected)/layout.tsx` wraps protected children in `SessionGuard` (`src/components/client/SessionGuard.tsx`). `SessionGuard`:

- `await auth.authStateReady()`, then reads `auth.currentUser`.
- If no current user → `router.replace(`/${locale}`)`.
- If `isAdminRoute` → additionally require `useAuthStore.isAdmin`.
- Otherwise renders children.

**There is no `emailVerified` check** — not in `SessionGuard`, not in `UseTokenRefresh`, not in `syncUserToBackend`, not in the store. `AuthUserDTO` (`src/lib/types/user/AuthUserDTO.ts`) has no `emailVerified` field; the backend sync response carries `disabled` (ban), `deletionStatus`, `banReason` — nothing about verification.

### What an unverified user sees today

Nothing different. Because `createUserWithEmailAndPassword` signs them in and the backend sync succeeds, an unverified user is **fully authenticated** and can do everything a verified user can. The only login-time gate that exists is the **ban** path: `mapAuthError` handles `auth/user-disabled` (`useAuthStore.ts:24`) and `syncUserToBackend` handles `disabled` / `EMAIL_BANNED` / `USER_BANNED` (`authService.ts:146,157`) by signing out and flipping `accountBanned`. Email-verification has no equivalent and would need to be added (a new gate — out of scope to build here, but note it: the brief's "cannot log in until verified" rule has **no enforcement point in web today**).

---

## Section 3 — A `/verify` landing page

### Does it exist?

**No.** No `app/**/verify` directory, no route, no component. Confirmed by `find app -type d -iname "*verify*"` (empty) and grep for `applyActionCode` / `oobCode` (empty). (Note: `app/[locale]/owner/account-verification/page.tsx` exists but is an unrelated "not ready yet" placeholder — KYC-style, renders a static `DASHBOARD_PAGES.not.ready.page.*` message. Not email-verification.)

### Locale-routing shape a new page would need

All pages live under `app/[locale]/…`. The planned URL `oglasino.com/[locale]/verify` would most naturally be a **public** page (no auth required to land on it from an email) — i.e. `app/[locale]/(portal)/(public)/verify/page.tsx`. Putting it under `(public)` keeps it out of `SessionGuard` (the gate lives in the sibling `(protected)` group) and gives it the standard portal chrome. A bare `app/[locale]/verify/page.tsx` would also work but skips the portal layout.

- **`[locale]` is a compound routing locale**, not a bare language. Valid values come from `src/i18n/routing.ts`: `rs-sr`, `rs-en`, `rs-ru`, `rsmoto-sr/-en/-ru`, `me-sr`, `me-cnr`, `me-en`, `me-ru` (default `rs-sr`). The email link must embed one of these exact strings or `[locale]/layout.tsx` calls `notFound()` (it validates with `hasLocale(routing.locales, locale)`).
- **Reading the locale:** server side via `getRoutingLocale()` (`src/i18n/getRoutingLocale.ts` — reads the `x-next-intl-locale` header next-intl sets from the segment) or `await params` (`params: Promise<{ locale: string }>`). Client side via `useRoutingLocale()` (`src/i18n/useRoutingLocale.ts` — `useParams().locale`).

### Reading the `oobCode` query param

Two established patterns in this repo:

- **Server component:** the page receives `searchParams: Promise<Record<string, string | string[] | undefined>>` and does `const { oobCode } = await searchParams` (pattern used by `app/[locale]/(portal)/(public)/page.tsx` and the admin pages).
- **Client component:** `useSearchParams()` from `next/navigation` (already used in `PortalConfigDialog.tsx`, `NavigationProgressBar.tsx`, `HomeLink.tsx`).

Because `applyActionCode` runs against the **browser** Firebase `auth` instance, the actual apply must happen in a client component. A natural shape: a server `page.tsx` that reads `oobCode` (and optionally `mode`) and renders a `'use client'` child that runs the apply and renders branded success/failure.

### Is `applyActionCode` reachable from web client code?

**Yes.** `firebase` is `^12.13.0` (`package.json`). The modular client SDK exports `applyActionCode` (and `checkActionCode`, `sendEmailVerification`, `confirmPasswordReset`) from `firebase/auth`. The repo already imports sign-in primitives from exactly that path (`authService.ts:4`). The `auth` instance is exported from `src/lib/config/firebaseClient.ts`.

- **Import path:** `import { applyActionCode } from 'firebase/auth';`
- **Call shape (modular v9+/v12):** `await applyActionCode(auth, oobCode);` — does **not** require a signed-in user; the `oobCode` itself is the credential, validated server-side by Firebase. Throws on expired/invalid/already-used codes (`auth/invalid-action-code`, `auth/expired-action-code`), which the page maps to a branded failure state.

### Where would "resend verification email" go?

**No existing endpoint.** Web's only `/auth*` calls are `POST /auth/firebase-sync` (`authService.ts:140`), `GET /auth/firebase/{uid}` (`userService.ts:79`), and `GET /secure/admin`. None resend verification.

Two candidate destinations (reporting, not building):

1. **Client-side Firebase `sendEmailVerification(auth.currentUser)`** — works only if a user is signed in, and sends Firebase's **default, un-branded** email. This contradicts the feature's premise (branded email sent backend-side via Brevo, link to our domain), so it's the wrong tool for this flow.
2. **A new backend endpoint** (e.g. `POST /auth/resend-verification`) that regenerates the Firebase verification link via the Admin SDK with our-domain `ActionCodeSettings` and sends it through Brevo. **This is the one consistent with the feature** — it would need the Backend engineer agent to build it. Web would just `BACKEND_API.post(...)` it. **No such endpoint exists today; flag for Backend.**

---

## Section 4 — Password reset (confirm only) — BRIEF VS REALITY

The brief asks me to "confirm the password-reset entry point is Firebase-native (`sendPasswordResetEmail`) and identify where it's triggered."

**There is no password-reset flow in web at all.** Findings:

- No call to `sendPasswordResetEmail`, `confirmPasswordReset`, or `verifyPasswordResetCode` anywhere in `src` / `app` (grep empty).
- No "forgot password" / "reset password" link or button in `LogInDialog.tsx`, `LoginOptionsDialog.tsx`, or anywhere else.
- No `/reset`, `/forgot`, or password-reset route directory.
- No `forgot` / `password.reset` translation keys in `src/messages`.

So the premise ("the entry point is Firebase-native, just confirm it") does not match the code: **the entry point doesn't exist.** The accurate statement for the spec is *"web has no password-reset flow today; nothing to leave untouched."* If the intent is that password reset stays Firebase-native **once added**, that's a future build, not a current state. I did not change anything — just flagging the mismatch so the spec records reality rather than a non-existent "no change."

---

## Section 5 — Seams

### Backend seam (the email carries the right link shape)

The `/verify` page assumes the branded email (sent backend-side via Brevo) carries a link to `https://oglasino.com/<compound-locale>/verify?oobCode=<firebase-action-code>`, where the `oobCode` is a Firebase email-verification action code minted by the Admin SDK (`generateEmailVerificationLink`) with `ActionCodeSettings.url` pointed at our domain.
**What to confirm with Backend:** (a) the link path uses a **valid compound routing locale** from `routing.ts` (the backend must know/choose the user's locale to build it); (b) the query param is literally named `oobCode` and whether Firebase's usual `mode=verifyEmail` / `apiKey` params are present or stripped — the page should tolerate extras but must find `oobCode`; (c) the action code is a *verify-email* code (so `applyActionCode` is correct), not a sign-in or reset code.

### Router seam (the `/verify` URL routes to this page through the worker)

The page assumes `/<locale>/verify` is forwarded by the Cloudflare worker to the Next.js origin like any other `[locale]` page — not swallowed by the maintenance matrix, not matched by the admin-request regex, not redirected.
**What to confirm with Router:** that an arbitrary new locale-prefixed path (`/me-cnr/verify`, etc.) passes through to origin unchanged, isn't shadowed by a redirect/maintenance rule, and that the `oobCode` query string survives the forward (the worker forwards with `redirect: "manual"`; confirm query preservation). Email links are often opened in fresh sessions / different browsers, so the route must work for an **unauthenticated** visitor — which is why a `(public)` placement (no `SessionGuard`) matters.

### Trust note (conventions Part 11)

The success/failure shown by the page is trustworthy because `applyActionCode` is validated server-side by Firebase — the client only relays the `oobCode`, it cannot forge a "verified" outcome. The page must derive success strictly from `applyActionCode` resolving, never from the mere presence of an `oobCode` in the URL.

---

## What building the `/verify` page would touch (inventory, not a build)

- **New:** `app/[locale]/(portal)/(public)/verify/page.tsx` (+ a `'use client'` child to run `applyActionCode`). New translation keys (DIALOG or a PAGES namespace + ERRORS for failure copy) — keys identified, seeded by Backend per conventions Part 6, not by web.
- **Likely touched to add the login gate:** `SessionGuard.tsx` and/or `UseTokenRefresh.tsx` / `useAuthStore` — to actually block unverified users (the brief's "cannot log in until verified" has no enforcement point today). Whether the gate is client-side (`firebaseUser.emailVerified`) or backend-driven (a new `AuthUserDTO` field) is a spec decision; flagging that it's a *new* gate, not an edit to an existing one.
- **Resend affordance:** depends on the new backend endpoint (Section 3) — web-side it's one `BACKEND_API.post` + a button.
- **Untouched:** `next.config.ts`, routing locales, password-reset (none exists).
