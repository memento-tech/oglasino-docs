# Audit — oglasino-web — password-reset

**Repo:** oglasino-web
**Branch:** dev
**Type:** READ-ONLY Phase-2 audit. No code changes. Inventory only.
**Date:** 2026-06-03

This audit reads the actual code so the password-reset spec is built on reality. The
reset page mirrors the just-shipped `/verify` page. Seven sections per the brief; each
quotes file paths / component names / error handling from code.

---

## 1. The `/verify` page (the pattern to mirror for `/reset`)

### Files

- Server entry: `app/[locale]/(portal)/(public)/verify/page.tsx`
- Client child: `app/[locale]/(portal)/(public)/verify/VerifyEmailClient.tsx` (`'use client'`)
- Firebase call wrapper: `src/lib/service/reactCalls/emailVerificationService.ts`

### How it reads `oobCode` from `searchParams`

`page.tsx` is an `async` server component. It awaits `searchParams` (a `Promise` under
Next 15), normalizes the possibly-array param to a single string, and hands it down as a
prop — defaulting to `null` when absent:

```tsx
// page.tsx:8-17
export default async function VerifyPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}) {
  const { oobCode } = await searchParams;
  const code = Array.isArray(oobCode) ? oobCode[0] : oobCode;
  return <VerifyEmailClient oobCode={code ?? null} />;
}
```

The server component does **no** Firebase work — it only extracts the code. All Firebase
action runs in the client child.

### How it calls `applyActionCode`

`VerifyEmailClient` runs the call inside a `useEffect`, through the service wrapper
`confirmEmailVerification`:

```tsx
// VerifyEmailClient.tsx:32-45
useEffect(() => {
  if (!oobCode) return;
  let cancelled = false;
  (async () => {
    const ok = await confirmEmailVerification(oobCode);
    if (cancelled) return;
    setStatus(ok ? 'success' : 'error');
  })();
  return () => { cancelled = true; };
}, [oobCode]);
```

The wrapper (`emailVerificationService.ts:26-33`):

```ts
export async function confirmEmailVerification(oobCode: string): Promise<boolean> {
  try {
    await applyActionCode(auth, oobCode);
    return true;
  } catch {
    return false;
  }
}
```

`applyActionCode(auth, oobCode)` is imported from `firebase/auth`
(`emailVerificationService.ts:4`). It needs only the `oobCode` and the browser `auth`
instance — no signed-in user, because verification links are usually opened in a fresh
browser. The call **does not create or refresh a session** — it only flips the
`email_verified` claim; the user logs in again afterward (documented in the wrapper's
comment, lines 20-25, and the client comment, lines 28-31).

### `(public)` placement (out of `SessionGuard`)

The verify route lives at `app/[locale]/(portal)/(public)/verify/`. The `(public)` route
group's layout (`app/[locale]/(portal)/(public)/layout.tsx`) renders only
`SelectableFilterManagerWrapper` + `Footer` — **no `SessionGuard`**. `SessionGuard` is
applied only by the **`(protected)`** group layout
(`app/[locale]/(portal)/(protected)/layout.tsx`), which wraps children in
`<SessionGuard isAdminRoute={false}>`. So the verify page is reachable
unauthenticated — exactly what an email-link landing needs. The reset page must sit under
`(public)` for the same reason.

`page.tsx`'s own header comment states this intent explicitly (lines 3-7): "Lives under
(public) so SessionGuard never redirects it — email links are often opened in a fresh,
unauthenticated browser."

### Branded success / failure states

`VerifyEmailClient` is a three-state machine: `type VerifyStatus = 'verifying' |
'success' | 'error'` (line 14). It renders branded, centered, in-portal content (not a
dialog) for each:

- **verifying** — `<Spinner />` + `tDialog('verify.page.verifying')` (lines 57-62)
- **success** — `<h1>` `tDialog('verify.page.success.title')`, `<p>`
  `tDialog('verify.page.success.body')`, and a `<Button variant="outline">` labelled
  `tButtons('verify.page.success.login.label')` (lines 64-72)
- **error** — same shape with `verify.page.error.title` / `.body` /
  `tButtons('verify.page.error.login.label')` (lines 74-82)

Both terminal buttons call `goToLogin` (lines 50-53), which opens
`DialogId.LOGIN_OPTIONS_DIALOG` and `router.push(\`/${locale}\`)` — verified users log in
fresh; failed users log in to request a new link. **No session is created on this page.**

### Success derived strictly from the Firebase call resolving (not from `oobCode` presence)

This is the trust-boundary property. Initial state is `'error'` if `oobCode` is missing,
else `'verifying'`:

```tsx
// VerifyEmailClient.tsx:26
const [status, setStatus] = useState<VerifyStatus>(oobCode ? 'verifying' : 'error');
```

`'success'` is set **only** when `confirmEmailVerification` returns `true`, i.e. only when
`applyActionCode` resolved without throwing (line 39: `setStatus(ok ? 'success' :
'error')`). Mere presence of an `oobCode` never yields success. The wrapper comment
(lines 28-31) and the service comment (line 21) both cite conventions Part 11 explicitly.
This is locked by a unit test: `emailVerificationService.test.ts:119` ("returns true only
when applyActionCode resolves") and `:125` ("returns false when applyActionCode rejects
(success is never derived from oobCode presence)").

### The resend affordance

`/verify` itself has **no** resend control — resend lives in a separate dialog,
`VerifyEmailDialog` (`src/components/popups/dialogs/VerifyEmailDialog.tsx`), shown after
an unverified login/registration, not on the link-landing page. See §4 for its mechanics.
The `/verify` page's only affordance is "go to login."

### Routing-locale handling — `[locale]` is a compound routing locale

`[locale]` is a **compound** segment combining tenant + language. The full supported set
(`src/i18n/routing.ts:5-16`):

```
rs-sr, rs-en, rs-ru, rsmoto-sr, rsmoto-en, rsmoto-ru, me-sr, me-cnr, me-en, me-ru
```

`defaultLocale: 'rs-sr'`; `localeDetection: false`; `localeCookie: false` (proxy.ts owns
locale, not next-intl).

**Validation / `notFound()` on invalid locale** happens in the `[locale]` root layout,
`app/[locale]/layout.tsx:26-28`:

```tsx
if (!hasLocale(routing.locales, locale)) {
  notFound();
}
```

`hasLocale` checks the segment against `routing.locales`. An unsupported segment ⇒
`notFound()` (renders the 404 boundary). The same layout also `notFound()`s when no
base-site resolves (`layout.tsx:34-36`). Because the reset page sits under
`app/[locale]/...`, it inherits this validation for free — no per-page locale check
needed.

**Compound-vs-formatter locale split.** next-intl is fed a *formatter* locale (e.g.
`cnr`), not the compound routing locale — the layout passes `oglasinoLocale` from
`getTenantLocale(locale)` into `<NextIntlClientProvider locale={oglasinoLocale}>`
(`layout.tsx:38-41`). Client code that needs the **compound** locale for URL construction
uses `useRoutingLocale()` (`src/i18n/useRoutingLocale.ts`), which returns
`useParams().locale` (the raw `[locale]` segment). `VerifyEmailClient` uses exactly this
(`const locale = useRoutingLocale()`, line 21) to build `router.push(\`/${locale}\`)`. The
reset page must do the same: `useRoutingLocale()` for URL building, `useTranslations()`
for copy. (`getTenantLocale` lives at `src/translations/lib/util/getTenantLocale.ts`;
`me-cnr` → `oglasinoLocale: 'cnr'`, formatter locale `cnr-ME`.)

---

## 2. `LoginDialog` (where the "Reset password" entry button goes)

### File & structure

`src/components/popups/dialogs/LogInDialog.tsx` — exports `LoginDialog` (the email +
password form). Registered as `DialogId.LOGIN_DIALOG` (`dialogRegistry.ts:4`,
`'loginDialog'`). The **social** dialog is a separate component,
`LoginOptionsDialog.tsx` (`DialogId.LOGIN_OPTIONS_DIALOG`), which offers Google + links to
the email dialog and register dialog.

Entry points into the auth dialogs: `LogInButton.tsx`
(`src/components/client/buttons/LoginButton.tsx`) opens `LOGIN_OPTIONS_DIALOG`. From
there the user can cross-link to `LOGIN_DIALOG` (email/password) or `REGISTER_DIALOG`.

`LoginDialog` structure (`LogInDialog.tsx`):

- Wrapped in `<DrawerDialog>` (responsive dialog/drawer). Title `tDialog('login.title')`,
  description `tDialog('login.description')` (lines 127-135).
- A top-of-form `<MessageText messages={[error || '']} severity={'error'} />` that renders
  the auth-store `error` (line 138).
- Two `<Input>`s: email (`tInputs('email.label')`, lines 141-146) and password
  (`isPassword`, `tInputs('password.label')`, lines 150-156). Form state is `useState<
  LoginUserRequestDTO>({ email, password })` with prop-drilled
  `handleChange(field, value)` doing a spread-merge (lines 30-72) — the repo's standard
  form pattern, not React Hook Form.
- A link-style toggle to the social/options dialog (lines 160-171).
- The submit `<Button variant="outline">` labelled `tDialog('login.submit.label')`, wired
  to `loginInternal` (lines 173-178).

### Where a link-style "Reset password" text button would naturally sit

Directly inside the form column, most naturally **below the password input** (after the
`<div className="h-5" />` spacer at line 158) and/or adjacent to the
`login.or.social` toggle block (lines 160-171), before the submit button. That block is
the existing precedent for a secondary text affordance inside this dialog.

### What styling primitive a link-style (non-button) text affordance uses elsewhere

The codebase's link-style text affordance is a shadcn **ghost Button stripped to look
like an inline link**:

```tsx
// LogInDialog.tsx:162-170  (the "log in with social" toggle)
<Button
  variant="ghost"
  className="h-auto p-0 text-base underline underline-offset-3 xl:text-sm"
  onClick={() => { onClose(); openDialog(DialogId.LOGIN_OPTIONS_DIALOG); }}>
  {tDialog('login.or.social.2')}
</Button>
```

The identical primitive (`variant="ghost"` + `h-auto p-0 ... underline
underline-offset-3`) recurs in `LoginOptionsDialog.tsx:75-83` and `:87-95` (the
login/register cross-links) and `RegisterDialog.tsx:225-233`. A "Reset password" entry
would mirror this exactly — it would `onClose()` the login dialog and `openDialog(...)` a
new reset dialog (or `router.push` the `/reset-request` route), depending on the spec's
chosen entry shape.

### Login submit path — what calls `signInWithEmailAndPassword`

Submit chain: `loginInternal` (LogInDialog.tsx:120-124) → on valid form,
`await login(userData)` from `useAuthStore`. The store action
(`useAuthStore.ts:180-192`):

```ts
login: async (userData: LoginUserRequestDTO) => {
  try {
    set({ loading: true, error: null });
    await loginUserFirebase(userData.email, userData.password);
  } catch (err) {
    console.error('[auth.store] auth action failed:', err);
    const friendly = mapAuthError(err);
    // null = user cancelled (e.g., closed popup) — don't show an error
    if (friendly !== null) set({ error: friendly });
  } finally {
    set({ loading: false });
  }
},
```

`loginUserFirebase` (`src/lib/service/reactCalls/authService.ts:168-171`) is what actually
calls Firebase:

```ts
export const loginUserFirebase = async (email: string, password: string): Promise<void> => {
  const res = await signInWithEmailAndPassword(auth, email, password);
  await ensureUserInFirestore(res.user);
};
```

`signInWithEmailAndPassword` is imported from `firebase/auth` (`authService.ts:6`). Note
the **listener-is-sole-hydrator** contract: the action returns `void`; the
`onIdTokenChanged` listener (UseTokenRefresh) is the sole caller of `syncUserToBackend`
and the sole hydrator of `useAuthStore.user` (comment block `authService.ts:160-167`).

### How login errors are caught and surfaced today

The catch in the store action (above) calls `mapAuthError(err)` and, if non-null, sets the
store `error` string. The dialog renders it via the top `<MessageText>` (LogInDialog
line 138). The full mapper (`useAuthStore.ts:10-56`) — **quoted in full because it is the
exact attach point for any Option-4 "this account uses Google sign-in" copy**:

```ts
function mapAuthError(err: unknown): string | null {
  const code = (err as { code?: string } | null)?.code;

  // User-cancelled actions — silent.
  if (code === 'auth/popup-closed-by-user' || code === 'auth/cancelled-popup-request') {
    return null;
  }

  // Banned user attempting sign-in. ... returning null suppresses the raw Firebase message
  if (code === 'auth/user-disabled') {
    useAuthStore.getState().setAccountBanned({ reason: null });
    return null;
  }

  if (code === 'auth/popup-blocked') {
    return 'Browser blocked the login popup. Please allow popups for this site and try again.';
  }
  if (
    code === 'auth/invalid-credential' ||
    code === 'auth/wrong-password' ||
    code === 'auth/user-not-found'
  ) {
    return 'Invalid email or password.';
  }
  if (code === 'auth/email-already-in-use') {
    return 'An account with this email already exists.';
  }
  if (code === 'auth/weak-password') {
    return 'Password is too weak. Choose a stronger one.';
  }
  if (code === 'auth/too-many-requests') {
    return 'Too many failed attempts. Please try again later.';
  }
  if (code === 'auth/network-request-failed') {
    return 'Network error. Please check your connection and try again.';
  }
  if (code === 'auth/account-exists-with-different-credential') {
    return 'This email is already registered with a different sign-in method.';
  }

  return (err as { message?: string } | null)?.message ?? 'Login failed. Please try again.';
}
```

### When email+password login fails on a Google/social account — what Firebase returns and how it's handled

**There is no provider-specific branch on the email+password path today.** When a user
enters an email that belongs to a Google-only (social) account and submits the
email+password form, modern Firebase (Identity Platform with **email-enumeration
protection** enabled — the default for current projects) returns the **unified
`auth/invalid-credential`** code, indistinguishable from a wrong password on a real
email/password account. `mapAuthError` maps `auth/invalid-credential` (alongside
`auth/wrong-password` / `auth/user-not-found`) to the generic **"Invalid email or
password."** (lines 32-38). So:

- **Today the email+password login path does NOT leak provider** — a Google account hit
  with a password attempt yields the same generic message as any bad credential. There is
  no "this account uses Google sign-in" copy, and no branch where it would attach without
  being added.
- The **only** "different sign-in method" copy that exists is for
  `auth/account-exists-with-different-credential` (lines 51-53), which fires on the
  **OAuth popup** path (`signInWithPopup`), not the email+password path — i.e. when a user
  tries Google sign-in for an email already linked to a *different* provider.

**Seam note for Mastermind (see "For Mastermind"):** Option 4 ("this account uses Google
sign-in") cannot be derived client-side from `auth/invalid-credential` while
email-enumeration protection is on — `fetchSignInMethodsForEmail` is intentionally
neutered under that protection. Distinguishing "wrong password" from "wrong provider"
requires either (a) disabling enumeration protection (re-introduces the existence leak —
contradicts the no-leak goal), or (b) a backend lookup, or (c) accepting the generic
message. This is a cross-repo decision, not a web-only one.

---

## 3. The existing-leak posture (CRITICAL — informs the no-leak design)

What the **current** web auth surface reveals about account existence / provider:

### Registration — `auth/email-already-in-use` → **LEAKS existence**

Register submit (`RegisterDialog.tsx:164-176`) → `register(userData)` store action
(`useAuthStore.ts:197-209`) → `registerUserFirebase` → `createUserWithEmailAndPassword`
(`authService.ts:173-181`). On `auth/email-already-in-use`, `mapAuthError` returns:

```
'An account with this email already exists.'
```

(`useAuthStore.ts:39-41`). Rendered on the register dialog via its `<MessageText
messages={[error || '']} severity="error">` (`RegisterDialog.tsx:190`). **This is a direct
account-existence leak** — an attacker can enumerate which emails are registered by
attempting registration. This is the single clearest existing leak on the surface and is
the primary thing the no-leak reset design should be measured against (the reset flow must
not reintroduce the same class of leak via its own responses).

### Login on a wrong-method account — **does NOT leak (generic message)**

As detailed in §2: `auth/invalid-credential` → "Invalid email or password."
Wrong-password, no-such-user, and wrong-provider all collapse to the same generic message
on the email+password path. **No existence or provider leak here today.**

### OAuth different-credential — **LEAKS provider (narrow path)**

`auth/account-exists-with-different-credential` → "This email is already registered with a
different sign-in method." (`useAuthStore.ts:51-53`). Fires only on the Google
`signInWithPopup` path. Narrow, but it does reveal that the email exists under another
provider.

### Verify-pending dialogs — reveal verification state only to the just-authenticated user

After a successful email/password sign-in or registration where the account is unverified,
both `LogInDialog` (`useEffect`, lines 44-63) and `RegisterDialog` (lines 59-78) detect
`isAwaitingEmailVerification()`, **sign the user out**, capture the typed email, and open
`VERIFY_EMAIL_DIALOG`. This is not a leak to a third party — it only surfaces to someone
who already proved the password — but it is the verification-state surface the reset flow
sits next to.

### Summary for the no-leak reset design

| Surface | Current behavior | Leak? |
|---|---|---|
| Register, existing email | "An account with this email already exists." | **Yes — existence** |
| Login (email+pw), any bad credential / wrong provider | "Invalid email or password." | No |
| OAuth popup, different provider | "This email is already registered with a different sign-in method." | **Yes — provider (narrow)** |

The reset-request flow should follow the **login** path's discipline (uniform response
regardless of whether the email exists or which provider it uses), **not** the
registration path's behavior. The registration leak is pre-existing and out of scope to
fix here, but it is the contrast case the spec should call out.

---

## 4. The resend-verification client call (the pattern to sibling)

### The `BACKEND_API.post` call

`src/lib/service/reactCalls/emailVerificationService.ts:60-76`:

```ts
export async function resendVerificationEmail(email: string): Promise<ResendVerificationResult> {
  try {
    const res = await BACKEND_API.post('/auth/resend-verification', { email });
    return { status: 'sent', retryAfterSeconds: extractRetryAfterSeconds(res.data) };
  } catch (err) {
    if (isErrorWithCode(err, 'VERIFICATION_RESEND_COOLDOWN')) {
      return {
        status: 'cooldown',
        retryAfterSeconds: extractRetryAfterSeconds((err as { data?: unknown }).data),
      };
    }
    if (isErrorWithCode(err, 'VERIFICATION_RESEND_DAILY_LIMIT')) {
      return { status: 'daily-limit' };
    }
    return { status: 'failed' };
  }
}
```

### Request shape

`POST /auth/resend-verification` with body `{ email }` — **email only, no session**. The
unverified user has already been signed out at the detection point, so there is no token
to attach; the address is the one captured from the login/register form (comment lines
53-59). The request goes through `BACKEND_API` (`src/lib/config/api.ts`), which attaches
`X-Base-Site` and `X-Lang` headers in its request interceptor (api.ts:99-100) and — since
there is no `auth.currentUser` — no `Authorization` header (api.ts:102-104).

### How it branches on the coded responses

The result type (`emailVerificationService.ts:39-43`):

```ts
export type ResendVerificationResult =
  | { status: 'sent'; retryAfterSeconds: number }
  | { status: 'cooldown'; retryAfterSeconds: number }
  | { status: 'daily-limit' }
  | { status: 'failed' };
```

Branching:

- **Success (HTTP 2xx)** ⇒ `{ status: 'sent', retryAfterSeconds }`. The client does **not**
  inspect a `VERIFICATION_EMAIL_SENT` code — success is the 2xx itself; it reads
  `retryAfterSeconds` off the success body.
- **`VERIFICATION_RESEND_COOLDOWN`** ⇒ `{ status: 'cooldown', retryAfterSeconds }`
  (re-seeds the countdown from the error body).
- **`VERIFICATION_RESEND_DAILY_LIMIT`** ⇒ `{ status: 'daily-limit' }` (no countdown).
- **Everything else (incl. `EMAIL_SEND_FAILED`)** ⇒ `{ status: 'failed' }`. Note:
  `EMAIL_SEND_FAILED` is **not** matched explicitly — it folds into the generic `failed`
  branch. A reset sibling can do the same, or add an explicit branch if the dialog needs
  distinct copy.

Coded-response detection uses `isErrorWithCode(err, CODE)`
(`src/lib/utils/isErrorWithCode.ts`), which tolerates both axios error shapes
(`err.response.data.errors[0].code` and the interceptor-unwrapped
`err.data.errors[0].code`). The `BACKEND_API` response interceptor rejects non-special
errors with `error.response` (api.ts:69), so in this path `err.data.errors[0].code` and
`err.data.retryAfterSeconds` both read off the response body. This matches conventions
Part 7 (codes, not messages).

### The backend-driven countdown mechanism

`extractRetryAfterSeconds(data)` (`emailVerificationService.ts:45-51`) reads a numeric
`retryAfterSeconds` from the response body (success **or** cooldown error), validates it
(`Number.isFinite && > 0`, `Math.ceil`), and falls back to
`DEFAULT_RESEND_COOLDOWN_SECONDS = 60` (line 37) when absent. The **backend** is the source
of the remaining seconds, so the countdown stays accurate across dialog dismiss/refresh.

The dialog that drives it, `VerifyEmailDialog.tsx`:

- Seeds `cooldownSeconds` from the result (`applyResult`, lines 52-72): `sent` sets it from
  `retryAfterSeconds` and shows the success message; `cooldown` sets it silently;
  `daily-limit` shows an error and sets `dailyLimitReached`; `failed` shows an error.
- Ticks down locally with a `setTimeout` decrement effect (lines 46-50).
- Disables the resend button while `loading || cooldownSeconds > 0 || dailyLimitReached`
  (line 82) and renders a countdown `<MessageText severity="info">` while
  `cooldownSeconds > 0` (lines 101-106).
- Copy keys: `tDialog('verify.email.resend.sent')`, `tErrors('verify.email.resend.
  dailylimit')`, `tErrors('verify.email.resend.failed')`, `tDialog('verify.email.resend.
  countdown', { seconds })`, button `tButtons('verify.email.resend.label')`.

**Sibling guidance:** a password-reset *request* call should mirror this almost exactly —
`POST /auth/<reset-request-endpoint>` with `{ email }` only (no session), branch on
backend codes, and reuse the same backend-driven `retryAfterSeconds` countdown pattern,
with reset-specific translation keys. The same `ResendVerificationResult`-style discriminated
union and `extractRetryAfterSeconds` helper are a clean template (could be generalized, but
see Part 4a note — do not over-abstract for one new caller without the spec confirming it).

---

## 5. Firebase client capability

`firebase/auth` is present and used (the verify page imports `applyActionCode` from it).
Installed version: **`firebase@12.13.0`** (`package.json` declares `"firebase":
"^12.13.0"`; `node_modules/firebase/package.json` confirms `12.13.0`). The browser client
is initialized in `src/lib/config/firebaseClient.ts` (singleton `app`, exported `auth`,
`browserLocalPersistence`).

I verified by runtime resolution (`node -e "require('firebase/auth')"`) that the following
are all importable **functions** from the same `firebase/auth` package/version the verify
page already uses:

```
confirmPasswordReset:    function
verifyPasswordResetCode: function
sendPasswordResetEmail:  function
applyActionCode:         function   (already imported by emailVerificationService.ts)
checkActionCode:         function
```

So the reset page can:
- `verifyPasswordResetCode(auth, oobCode)` — validate the code and recover the target email
  (mirrors `checkActionCode`).
- `confirmPasswordReset(auth, oobCode, newPassword)` — complete the reset.

…with the exact same import/usage shape as `applyActionCode`. And a reset **request** page
can use `sendPasswordResetEmail(auth, email)` client-side **if** the spec chooses the
Firebase-direct path — though the resend-verification precedent (§4) routes through the
**backend** (`/auth/resend-verification`) for rate-limit + branded-email + no-leak control,
which is the more likely model for the reset request. (Decision for Mastermind; both are
capability-available.)

**Trust-boundary carry (conventions Part 11):** as with `/verify`, the reset page must
derive success **strictly** from `confirmPasswordReset` resolving — never from the presence
of an `oobCode`. The `/verify` page already does this for `applyActionCode`
(`VerifyEmailClient.tsx:26,39`; locked by `emailVerificationService.test.ts:119,125`), so
the pattern carries directly.

---

## 6. Router / deep-link config (scaffold target for the deferred app-redirect)

**Nothing exists yet.** No URL-scheme / universal-link / app-association config is present
in web:

- No `public/.well-known/` directory at all (`public/` top-level contains only images,
  `favicon.ico`, the two `firebase-messaging-sw*.js` files, and `design`/`logo` folders).
- No `apple-app-site-association` file anywhere in the repo.
- No `assetlinks.json` anywhere in the repo.
- No app-redirect / deep-link helper module. A repo-wide grep for
  `apple-app-site|assetlinks|universal.?link|deep.?link|app.?store|play.?store|itunes|
  apps.apple|play.google` returns only (a) `/catalog` **slug** deep-linking copy in the
  design topics doc (`app/[locale]/design/topics.ts` — unrelated, it's about category URL
  routing) and (b) a single "RN App Store Submission" feature label in
  `app/[locale]/wants/page.tsx:127` (a roadmap/wants page string, not config).

The closest existing artifact to an "app" affordance is the **footer store badges**:
`src/components/server/layout/Footer.tsx:80-87` renders `<GooglePlayGetIt />` and
`<AppleStoreGetIt />` (icon components from `src/components/icons/`) inside
`cursor-pointer` `<div>`s — but they are **not wrapped in any link** and carry **no store
URL**. (This matches the open issues.md entry "2026-05-31 — Web: app-store / play-store
badges are dead links (icons only).") So even the store-badge surface is non-functional
today.

**Conclusion for the scaffold:** there is no existing deep-link/app-association
infrastructure for the reset success page's deferred app-redirect button to target. The
scaffold will be greenfield — built-but-dormant per the brief. When the deep-link work is
done later, it will also need `.well-known/apple-app-site-association` +
`assetlinks.json` (neither exists), and likely a tie-in to the still-dead footer store
badges. Documented here so the later session knows nothing is pre-wired.

---

## 7. Translation consumption

Web copy is **backend-seeded**, fetched at SSR time and provided to `next-intl` — there
are **no local message JSON files** for this content.

### Mechanism

- Translations are fetched per namespace+lang from the backend:
  `GET {NEXT_PUBLIC_API_URL}/public/translations?namespace=<ns>&lang=<lang>`
  (`src/translations/lib/translationsCache.ts:27-28`). The backend returns an array of
  `{ key, value }` DTOs; the cache flattens then **nests** them (`toNested`, lines 3-18)
  into the object shape next-intl expects.
- The fetch is wrapped in Next's `unstable_cache` (`getNamespaceTranslations`, lines
  82-89) with `revalidate: 86400` and tag `'translations'`, so backend `Cache-Control`
  can't defeat caching and entries are shared cross-instance. Cache-bust via the
  `'translations-v1'` key or `revalidateTag('translations')`.
- `src/i18n/request.ts` (`getRequestConfig`) resolves the routing locale, derives the
  formatter locale via `getTenantLocale`, and `loadAllNamespaces(oglasinoLocale)` to build
  the `messages` object handed to next-intl (lines 16-26). The `[locale]` layout wraps the
  tree in `<NextIntlClientProvider locale={oglasinoLocale}>` (`layout.tsx:41`).
- Components consume copy with next-intl's `useTranslations(TranslationNamespaceEnum.X)`.
  The namespace enum is `src/translations/types/TranslationNamespaceEnum.ts`; namespaces are
  fixed and mirror the backend `TranslationNamespace` enum (conventions Part 6).

### Which namespaces the auth/verify copy uses (the reset page will mirror)

From the audited components:
- `DIALOG` — dialog titles/bodies/labels (`verify.page.*`, `verify.email.*`, `login.*`,
  `register.*`).
- `BUTTONS` — button labels (`verify.page.success.login.label`, `verify.email.resend.
  label`, etc.).
- `ERRORS` — error copy (`verify.email.resend.dailylimit`, `verify.email.resend.failed`;
  register `displayName.*`).
- `INPUT` — field labels (`email.label`, `password.label`, `username.label`).
- `VALIDATION` — **frozen** namespace; existing structural-validation keys only
  (`email.empty`, `email.bad`, `password.empty`, `password.bad`, `suspicion`). Per
  conventions Part 6, **new** error-like keys go to `ERRORS`, not `VALIDATION`.

### Implication for the reset spec

Reset/login copy is **backend-seeded**. Per CLAUDE.md "Stack reminders" and conventions
Part 6 Rule 3, **web does not add keys to the SQL seed** — the Backend engineer agent does.
The reset spec must enumerate the new keys (DIALOG for the reset page/dialog copy, BUTTONS
for labels, ERRORS for failure/limit copy, INPUT for the new-password field label) so Igor
can pass that list to Backend. Web's job is to *identify* the missing keys and *consume*
them, not to seed them. Parent/child key-collision rule (Part 6 Rule 2) applies — e.g. if
`reset.password` is a leaf, don't also add `reset.password.<suffix>`; suffix the parent
(`.label`/`.title`).

---

## Trust-boundary focus (conventions Part 11) — confirmation

The `/verify` page already derives success strictly from the Firebase call resolving, not
from `oobCode` presence:
- Initial state is `'error'` when `oobCode` is missing (`VerifyEmailClient.tsx:26`).
- `'success'` is set **only** when `applyActionCode` resolves (`confirmEmailVerification`
  returns `true`, lines 37-39).
- Locked by `emailVerificationService.test.ts:119` and `:125`.

The reset page must replicate this for `confirmPasswordReset` — success only when the
Firebase call resolves, never from the mere presence of `oobCode`. The pattern carries.

---

## Definition of done — checklist

- [x] §1 verify page — paths, oobCode read, applyActionCode call, (public) placement,
  branded states, success-from-resolve, resend affordance, compound-locale handling +
  `notFound()`.
- [x] §2 LoginDialog — structure, link-style primitive, submit path
  (`signInWithEmailAndPassword`), catch block quoted in full, Google-account-login behavior
  (`auth/invalid-credential` → generic; no provider branch).
- [x] §3 leak posture — registration leaks existence; login does not; OAuth narrow leak;
  exact copy quoted; summary table.
- [x] §4 resend-verification — request shape, code branches, backend-driven countdown.
- [x] §5 Firebase capability — `firebase@12.13.0`; `confirmPasswordReset` /
  `verifyPasswordResetCode` / `sendPasswordResetEmail` confirmed importable.
- [x] §6 deep-link config — nothing exists (stated explicitly); footer badges are the only
  near-surface and are themselves dead.
- [x] §7 translation consumption — backend-seeded via next-intl; namespaces; web identifies
  but does not seed keys.

No code was changed. This is inventory only.
