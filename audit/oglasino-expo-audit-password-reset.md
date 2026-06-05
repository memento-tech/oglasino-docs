# Audit — oglasino-expo: Password-Reset (mobile login surface, current state)

**Repo:** oglasino-expo
**Branch:** `new-expo-dev`
**Type:** READ-ONLY audit (Phase 2). No code changed, nothing staged, no config edited.
**Date:** 2026-06-03
**Feature slug:** `password-reset`
**Brief:** `.agent/brief.md` — inventory the mobile login surface so the password-reset spec is built on real code. Mobile's scope this feature: (a) a link-style "Reset password" entry button on the login surface, and (b) the Option-4 "this account uses Google sign-in" copy when email+password login fails on a social account. The reset page is web-owned (browser funnel). The success-step app-deep-link is deferred to a later session.

Code on `new-expo-dev` is ground truth. Every file:line below was read directly this session.

---

## TL;DR verdict

- **The login surface is two dialogs**, not a screen: `LoginDialog.tsx` (email+password) and `LoginOptionsDialog.tsx` (Google + register/login links). Both are registered modals opened via the dialog store. The "Reset password" entry button belongs in `LoginDialog.tsx`, directly under the password field (next to the existing `loginError` line).
- **A link-style text affordance already has a proven primitive in this exact file**: `<Pressable onPress=…><Text className="text-primary underline">…</Text></Pressable>` (`LoginDialog.tsx:110-115`). `LoginOptionsDialog.tsx` uses a near-identical `text-blue-600 underline` variant. Either is the template; no new primitive needed.
- **🔴 The Option-4 trigger is the hard problem.** When an email+password login is attempted against a Google-only account, modern Firebase (email-enumeration protection, default on new projects) returns **`auth/invalid-credential`** — *indistinguishable from a wrong password*. There is **no** `auth/account-exists-with-different-credential` on the `signInWithEmailAndPassword` path, and the app does **no** provider lookup (`fetchSignInMethodsForEmail` appears nowhere). So a purely client-side, error-code-driven "use Google" message **cannot reliably fire today**. The spec must define a detection mechanism. This is the single most important finding — details in §2.
- **🟠 Login errors are not translated today.** `mapAuthError` (`authErrors.ts`) returns **hard-coded English strings**, not translation keys and not i18n lookups. Any Option-4 copy that follows the existing pattern would also be hard-coded English — which collides with conventions Part 6 (errors are translated, backend-seeded). The spec must decide whether Option-4 copy is a seeded `ERRORS` key or matches the existing hard-coded-English debt. See §2 and §5.
- **The browser funnel already exists and is purely outbound.** `Linking.openURL(...)` is used in five+ places to leave the app. The reset email link opens the web `/reset` page in the browser with **zero mobile-side work for the reset flow itself** — confirmed. The reset *flow* is web-owned.
- **🔵 Inbound browser→app deep linking does NOT exist today.** Definitive answer to brief §4: there is **no** `app/+native-intent.ts`, **no** `Linking.getInitialURL`/`addEventListener('url')` handler, **no** custom `linking` config. The per-tier custom schemes (`oglasino://` / `oglasino-preview://` / `oglasino-dev://`) are registered, and expo-router auto-maps the route tree, but nothing app-side is wired to *catch* a link opened from Safari/Chrome beyond expo-router's default scheme handling. The success-step deep-link Igor builds next session starts from zero on the handler. Evidence in §4.
- **🟡 Section-5 clarification:** mobile has **no `BACKEND_TRANSLATIONS` namespace** (it is absent from the mobile `TranslationNamespace` enum). The brief's parenthetical "the verify feature's dialog copy is backend-seeded" is true only in the loose sense that *all* mobile copy is seeded in the backend translations table and fetched at boot — but it arrives through the **standard frontend namespaces** (`DIALOG` / `ERRORS` / `BUTTONS` / `INPUT`), not `BACKEND_TRANSLATIONS`. The "Reset password" label → `DIALOG` (or `BUTTONS`); the Option-4 message → `ERRORS`. See §5.

---

## 1. The mobile login surface (where "Reset password" goes)

### 1.1 It is two dialogs, not a screen

There is no `LoginScreen`. Login is delivered as two registered modal dialogs (`src/components/dialog/dialogRegistry.ts:2-4`):

```ts
LOGIN_OPTIONS_DIALOG = 'loginOptionsDialog',
REGISTER_DIALOG = 'registerDialog',
LOGIN_DIALOG = 'loginDialog',
```

They are opened from many entry points via `openDialog(DialogId.LOGIN_OPTIONS_DIALOG)` — `BottomBar.tsx`, `UserMenu.tsx`, `FavoriteButton.tsx`, `FollowUserButton.tsx`, `ReportButton.tsx`, `CallUserButton.tsx`, `ProductReviewButton.tsx`, `ProductUserDetails.tsx`. The two dialogs cross-open each other:
- `LoginOptionsDialog` → opens `LOGIN_DIALOG` ("login with email") and `REGISTER_DIALOG`.
- `LoginDialog` → opens `LOGIN_OPTIONS_DIALOG` (the "or social" affordance).

### 1.2 `LoginDialog.tsx` — the email+password dialog (Reset-password host)

Structure (`src/components/dialog/dialogs/LoginDialog.tsx`):

- Title/description via `DialogTitleDescription` (`:80-83`), keys `login.title` / `login.description` (DIALOG).
- Two `Input`s — email (`:87-93`) and password (`:96-102`) — via the shared `basic/Input.tsx`. Labels `email.label` / `password.label` (INPUT namespace, `tInputs`).
- **Inline login error line** (`:105`): `{loginError && <Text className="ml-3 italic text-red-500">{loginError}</Text>}`. `loginError` is seeded from the store's `error` via a `useEffect` (`:38-42`). **This is the visual neighbor of where Option-4 copy renders.**
- A centered "or social" row (`:108-116`) — a `<Text>` + a link-style `<Pressable>` that opens `LOGIN_OPTIONS_DIALOG`.
- The submit `<Pressable>` (`:118-128`) → `loginInternal` → `useAuthStore.getState().login(userData)` (`:69`).

**Recommended seat for the "Reset password" button:** between the password `<Input>` (`:103`) and the `loginError` line (`:105`), or immediately after the submit button — a single link-style `<Pressable>` row. It needs no new state; it only opens the browser (§3) at the web `/reset` page.

### 1.3 The link-style (non-button) text primitive — already in-file

There is no dedicated `<Link>` component; a link-style affordance is composed inline as a `<Pressable>` wrapping a styled `<Text>`. Two live variants:

- **`LoginDialog.tsx:110-115`** (the "or social" link), `primary` color:
  ```tsx
  <Pressable disabled={loading} className="items-center py-2"
    onPress={() => openDialog(DialogId.LOGIN_OPTIONS_DIALOG)}>
    <Text className="text-primary underline">{tDialog('login.or.social.2')}</Text>
  </Pressable>
  ```
- **`LoginOptionsDialog.tsx:61-67` / `:72-80`** (the register/login links), `blue-600`:
  ```tsx
  <Pressable onPress={() => { onClose(); openDialog(DialogId.LOGIN_DIALOG); }}>
    <Text className="text-blue-600 underline">{tDialog('login.options.or.login.2')}</Text>
  </Pressable>
  ```

Either pattern is the template for "Reset password". `text-primary underline` (matching the same dialog's existing link) is the natural choice for in-file visual consistency. **No new styling primitive is required.**

---

## 2. Login submit path + Option-4 attach point (CRITICAL)

### 2.1 The exact call chain

`LoginDialog.loginInternal` (`LoginDialog.tsx:61-70`) → `useAuthStore.getState().login(userData)` →
`authStore.login` (`authStore.ts:102-118`) → `loginUserFirebase(email, password)` (`authService.ts:175-179`) →
`signInWithEmailAndPassword(auth, email, password)` (`authService.ts:176`) → `buildUserSession(res.user)`.

### 2.2 The catch block (quoted verbatim — the Option-4 attach point)

`src/lib/store/authStore.ts:102-118`:

```ts
login: async ({ email, password }) => {
  try {
    set({ loading: true, error: null });
    const backendUser = await loginUserFirebase(email, password);
    set({ user: backendUser });
    if (backendUser) trackAuthEvent(backendUser);
  } catch (err: any) {
    if (err instanceof EmailNotVerifiedError) {
      set({ awaitingVerificationEmail: err.email ?? email });
      return;
    }
    console.error('Email login failed', err);
    set({ error: mapAuthError(err) });
  } finally {
    set({ loading: false });
  }
},
```

The error string is produced by `mapAuthError` and lands in store `error` → mirrored to `loginError` → rendered at `LoginDialog.tsx:105`. **Option-4 copy must be produced here (or in `mapAuthError`) and will surface on that same red line.** Note the existing `console.error('Email login failed', err)` is a standing Part 4 debug-logging item (pre-existing; see §Adjacent observations — not in this audit's scope to fix).

### 2.3 `mapAuthError` — the error→copy mapper (quoted)

`src/lib/utils/authErrors.ts` (whole file):

```ts
export function mapAuthError(err: unknown): string | null {
  const code = (err as { code?: string } | null)?.code;
  switch (code) {
    case 'auth/popup-closed-by-user':
    case 'auth/cancelled-popup-request':
      return null;
    case 'auth/popup-blocked':
      return 'Browser blocked the login popup. Please allow popups for this site and try again.';
    case 'auth/invalid-credential':
    case 'auth/wrong-password':
    case 'auth/user-not-found':
      return 'Invalid email or password.';
    case 'auth/email-already-in-use':
      return 'An account with this email already exists.';
    case 'auth/weak-password':
      return 'Password is too weak. Choose a stronger one.';
    case 'auth/too-many-requests':
      return 'Too many failed attempts. Please try again later.';
    case 'auth/network-request-failed':
      return 'Network error. Please check your connection and try again.';
    case 'auth/account-exists-with-different-credential':
      return 'This email is already registered with a different sign-in method.';
  }
  const message = (err as { message?: string } | null)?.message;
  return message ?? 'Login failed. Please try again.';
}
```

**Two facts the spec must build on:**

1. **All returns are hard-coded English literals** — no i18n, no backend-seeded keys. (See §5 / §Conventions: this conflicts with Part 6, which says errors are seeded + translated. It is pre-existing debt, not something this feature created.)
2. **There is no "this is a Google account" branch**, and there is no provider lookup anywhere in the login path (grep for `fetchSignInMethodsForEmail` → **zero hits**; the only `providerId` reads are post-authentication: `DeleteAccountConfirmationDialog.tsx:46` and the backend-sync body at `authService.ts:147`).

### 2.4 🔴 What Firebase actually returns for a social account — and why Option-4 is non-trivial

**Today's behavior:** an email+password login against an account that exists **only** with the Google provider throws from `signInWithEmailAndPassword`, is caught at `authStore.ts:108`, and `mapAuthError` returns **`'Invalid email or password.'`** (via the `auth/invalid-credential` case). The user is told their credentials are wrong, with **no** hint that the account is Google-backed. The Option-4 copy does not exist in any form today.

**Why a client-only Option-4 cannot be reliably built on the error code (platform gotcha):**

- New Firebase projects enable **email-enumeration protection** by default. Under it, `signInWithEmailAndPassword` returns the generic **`auth/invalid-credential`** for *all* of: wrong password, non-existent account, and "account exists but with a different (e.g. Google) provider." These cases are **deliberately indistinguishable** to prevent account enumeration.
- `auth/account-exists-with-different-credential` is thrown on **credential-linking / OAuth `signInWithCredential`** collisions — **not** on the `signInWithEmailAndPassword` path. So the one error code that *names* the situation never reaches this catch block for this flow.
- `fetchSignInMethodsForEmail` (the classic "which providers does this email use?" probe) is **also neutered by email-enumeration protection** — it returns an empty array regardless. Relying on it would be both absent today and ineffective under the current Firebase posture.

**Consequence for the spec:** the Option-4 "use Google" message **needs a deliberate detection design**, not just a new `case` in `mapAuthError`. Options the spec should weigh (flagged, not chosen — that is Mastermind's call):
  - (a) A **backend signal** — e.g. the login attempt (or a small pre-check endpoint) tells the client the email is registered with Google. This keeps detection server-side (Part 11 trust posture) but is a new contract.
  - (b) **Disable email-enumeration protection** in Firebase so `auth/account-exists-with-different-credential` / provider probes become usable — a security-posture decision with enumeration trade-offs, cross-cutting (web shares the Firebase project).
  - (c) **Don't auto-detect**: show a generic "wrong credentials — if you signed up with Google, use the Google button" hint on *every* failed email+password login. Lowest-tech, no detection, but always-on copy. (This may be exactly what "Option-4" already means upstream — confirm against the Mastermind plan.)

This is the brief's central ask ("what error does Firebase return, and how is it handled right now") and its answer changes the shape of the mobile work. **Flagged for Mastermind in §9.**

### 2.5 Coexistence with the email-verification sign-out gate (brief §2 requirement)

The just-shipped email-verification feature added a client-side sign-out gate on **both** sync paths. Option-4 copy must not collide with it. How each path handles login errors today:

**Path A — explicit `login()` / `buildUserSession`** (`authService.ts:158-170`):
```ts
export const buildUserSession = async (firebaseUser) => {
  if (await isAwaitingEmailVerification(firebaseUser)) {
    await auth.signOut();
    throw new EmailNotVerifiedError(firebaseUser.email);
  }
  await ensureUserInFirestore(firebaseUser);
  return await syncUserToBackend(firebaseUser);
};
```
The store catch (`authStore.ts:109-112`) branches on `err instanceof EmailNotVerifiedError` **first** and `return`s early (seeding `awaitingVerificationEmail`, opening the verify dialog) — it never reaches `mapAuthError`. `isAwaitingEmailVerification` (`verificationService.ts:30-33`) is `signInProvider === 'password' && !emailVerified`, so **only unverified password accounts** hit this branch; a Google account returns `false` and passes through.

**Path B — `initAuthListener` / `onIdTokenChanged`** (`authStore.ts:243-307`). `signInWithEmailAndPassword` also fires this listener. It independently runs the same verification gate (`:273-277`: set `awaitingVerificationEmail`, `auth.signOut()`, `return`) and separately catches `USER_BANNED` / `EMAIL_BANNED` from the backend sync (`:296-300`). It does **not** call `mapAuthError` and does **not** surface invalid-credential errors (those never reach it — a failed `signInWithEmailAndPassword` never produces a signed-in user for the listener to process).

**Takeaway:** Option-4 copy belongs **only** on Path A's `mapAuthError` branch (or a sibling branch in `authStore.login`'s catch, *after* the `EmailNotVerifiedError` check). It is structurally separate from the verification gate — a wrong-credential / wrong-provider failure throws from `signInWithEmailAndPassword` **before** any user object exists, so neither the verification sign-out nor the ban sign-out is in play. **No conflict**, provided the Option-4 branch sits after the `instanceof EmailNotVerifiedError` early-return.

---

## 3. The browser funnel (how mobile sends users to the web)

**Pattern: `Linking.openURL(...)`, outbound only.** Live call sites:

- `src/components/init/HardUpdateScreen.tsx:18` — `Linking.openURL('https://memento-tech.com')`
- `src/components/init/SoftUpdateModal.tsx:19` — same
- `src/components/pricingPage/SupportButton.tsx:12` — `Linking.openURL('https://ko-fi.com/oglasino')`
- `src/components/product/CallUserButton.tsx:65` — `Linking.openURL('tel:${phoneNumber}')`
- `src/components/messages/Message.tsx:42` — `Linking.openURL(run.url!)` (in-message links)
- `src/components/dialog/dialogs/AppVersionConfigurationDialog.tsx:23` — external site

`expo-web-browser` is also a dependency (`authService.ts:4`, `WebBrowser.maybeCompleteAuthSession()` at `:29`) but is used only for the Google OAuth session, not for general outbound navigation. **The established funnel for "open a web page" is `Linking.openURL`.**

**Confirmed: nothing to build mobile-side for the reset *flow*.** The reset page is web-owned. Mobile's only browser interaction for password-reset is opening the web `/reset` (or equivalent) URL via `Linking.openURL` — and that may not even be needed if "Reset password" simply deep-links the user to the web reset entry the same way other outbound links work. The reset email link, when tapped from the user's mail client, opens the web `/reset` page in the browser with **no app involvement**. The success-step *return* to the app (the inbound deep-link) is the deferred next-session piece — see §4.

---

## 4. Inbound deep-link config (DEFERRED build — audit only)

### 4.1 The per-tier scheme config (quoted from `app.config.ts`)

`app.config.ts:31-32` derives the scheme per environment, and `:40` sets it:

```ts
const appScheme =
  ENV === 'production' ? 'oglasino' : ENV === 'preview' ? 'oglasino-preview' : 'oglasino-dev';
// ...
scheme: appScheme,
```

Per-tier bundle IDs (`app.config.ts:7-12`): `com.oglasino` / `com.oglasino.preview` / `com.oglasino.development`. expo-router is the entry (`app.config.ts:88-89`, plugin `'expo-router'`). These match the earlier deep-link audit (`.agent/audit-deep-links.md` §1.1), which additionally verified the native manifests carry the scheme.

### 4.2 🔵 Does the app handle an INBOUND browser→app deep link today? — **NO. Definitively no.**

Evidence (all confirmed this session on `new-expo-dev`):

| Check | Result |
|---|---|
| `app/+native-intent.ts` (expo-router inbound rewrite hook) | **Not present** (`ls` → "NO +native-intent.ts") |
| `Linking.getInitialURL` | **Zero hits** in `src/` and `app/` |
| `Linking.addEventListener('url', …)` | **Zero hits** |
| `Linking.useURL` / `useLinkingURL` | **Zero hits** |
| custom `linking` config / `getStateFromPath` | **None** (expo-router owns routing by default) |

The only inbound routing that exists is **expo-router's default**: the registered custom scheme (`oglasino-dev://...`) maps to the file-based route tree automatically, so `oglasino-dev://product/123/slug` would open the app on that screen *if launched*. But there is **no app-authored handler** that catches a URL opened from an external browser, inspects it, or runs any custom logic (locale-strip, success-step routing, token handling). `Linking` usage is **100% outbound** (§3).

**For the deferred next session:** the success-step app-deep-link (web `/reset` success → back into the app) starts from zero on the handler side. The scheme to target per tier is `oglasino://` (prod) / `oglasino-preview://` / `oglasino-dev://`. The richer cross-repo picture (universal/app links vs. custom scheme, the locale-prefix mismatch that needs `app/+native-intent.ts`, the web-hosted `.well-known` blocker) is already documented in `.agent/audit-deep-links.md` — the password-reset success deep-link should be scoped against that audit, because a custom-scheme link works app-installed-only and needs no web `.well-known`, whereas a real `https://` return link does. **Which scheme the web scaffold targets is a cross-repo contract Igor brokers.**

---

## 5. Translation consumption (where the copy comes from)

### 5.1 The loading layer

Mobile fetches translations per namespace from the backend at boot and registers them into a single react-i18next instance:

- `src/i18n/fetchNamespace.ts:31` — `GET /public/translations?namespace=${namespace}&lang=${lang}`, flattened then nested.
- `src/lib/store/bootStore.ts:531-557` — Gate 4 registers each namespace's payload into i18n via `init()` (first boot) or `addResourceBundle(activeLang, ns, payload, true, true)` (later passes), with `ns: Object.values(TranslationNamespace)` and `defaultNS: COMMON_SYSTEM`.
- `src/i18n/useTranslations.ts` — thin wrapper: `useTranslation(namespace)` → returns `t`. Components call `useTranslations(TranslationNamespace.DIALOG)` etc.

So **every** user-visible mobile string is backend-seeded (in the backend translations table) and fetched at boot — including the verify feature's dialog copy.

### 5.2 🟡 There is no `BACKEND_TRANSLATIONS` namespace on mobile

The mobile `TranslationNamespace` enum (`src/i18n/types.ts:6-38`) contains: COMMON, COMMON_SYSTEM, ERRORS, VALIDATION, PAGING, BUTTONS, INPUT, DIALOG, HEADER, FOOTER, NAVIGATION, INTRO, EXTRA_PRODUCTS, COOKIES, MESSAGES_PAGE, DASHBOARD_PAGES, ABOUT_PAGE, FREE_ZONE_PAGE, PRICING_PAGE, METADATA. **`BACKEND_TRANSLATIONS` is absent**, and grep finds no consumer of it (`BACKEND_TRANSLATIONS` appears nowhere in `src/`/`app/` outside the enum file — and it is not even in the enum).

This is consistent with conventions Part 6: `BACKEND_TRANSLATIONS` is for strings the **backend emits directly** to the user (push bodies, emails) bypassing the frontend translation layer — which is precisely the channel the password-reset *email* uses, not the in-app copy.

**The verify feature's dialog copy** (the brief's cited precedent) confirms the pattern: `VerifyEmailDialog.tsx:19-21` consumes `DIALOG`, `ERRORS`, and `BUTTONS` — **standard frontend namespaces**, not `BACKEND_TRANSLATIONS`.

### 5.3 Where the password-reset copy should come from (for the spec)

- **"Reset password" entry-button label** → a `DIALOG` key (it lives inside `LoginDialog`, alongside `login.*` keys) — or `BUTTONS` if Mastermind prefers it grouped with button labels. Consumed via `tDialog('reset.password.label')` (or `tButtons(...)`).
- **Option-4 "this account uses Google sign-in" copy** → an `ERRORS` key (Part 6: "anything red-on-screen the user needs to … acknowledge … all new error-like keys go [to ERRORS]"). It renders on the same red `loginError` line.

⚠️ **Tension to resolve (see §2.3 point 1):** the *existing* login-error copy is hard-coded English in `mapAuthError`, **not** seeded keys. Adding the Option-4 string as a proper `ERRORS` key would be the convention-correct path but would sit **inconsistently** next to a mapper full of English literals. The spec must pick: (a) seed Option-4 as an `ERRORS` key and translate it (correct, but a lone island), or (b) match the existing hard-coded-English debt (consistent-but-wrong). Recommend (a) with a note that `mapAuthError`'s broader i18n migration is separate debt. **Flagged for Mastermind.**

---

## 6. Brief vs reality

1. **Option-4 detection is not achievable from the Firebase error code today (platform gotcha).**
   - Brief says: add Option-4 "this account uses Google sign-in" copy "when an email+password login fails because the account is a Google/social account," implying a detectable failure condition.
   - Code says: the failure surfaces as `auth/invalid-credential` at `authStore.ts:108` → `mapAuthError` → `'Invalid email or password.'`. There is **no provider detection** (`fetchSignInMethodsForEmail` absent; `account-exists-with-different-credential` is not thrown on this path), and Firebase email-enumeration protection makes the social-account case **indistinguishable** from a wrong password. (§2.3, §2.4.)
   - Why this matters: a client-only, error-driven Option-4 trigger cannot reliably fire. The spec needs an explicit detection design (backend signal, Firebase posture change, or an always-on generic hint).
   - Recommended resolution: Mastermind chooses the detection mechanism in the spec (§2.4 a/b/c). Mobile cannot implement Option-4 deterministically without one.

2. **Login errors are hard-coded English, not seeded translations.**
   - Brief says (§5): the Option-4 copy comes from the backend-seeded translation layer (verify-feature precedent).
   - Code says: `mapAuthError` (`authErrors.ts`) returns English string literals; the login path consumes no `ERRORS` namespace for these messages.
   - Why this matters: a seeded Option-4 `ERRORS` key would be correct per Part 6 but inconsistent with the surrounding mapper; a hard-coded Option-4 string would match the code but violate Part 6.
   - Recommended resolution: spec seeds Option-4 as an `ERRORS` key (translated), and notes `mapAuthError`'s full i18n migration as separate pre-existing debt. (§5.3.)

3. **"BACKEND_TRANSLATIONS" framing (minor clarification, not a blocker).**
   - Brief says (§5): "the verify feature's dialog copy is backend-seeded," consume `BACKEND_TRANSLATIONS` for auth copy.
   - Code says: mobile has **no** `BACKEND_TRANSLATIONS` namespace; auth/dialog copy comes through `DIALOG`/`ERRORS`/`BUTTONS` (`VerifyEmailDialog.tsx:19-21`). All of it is still backend-seeded, just via the standard namespaces. (§5.2.)
   - Why this matters: the spec should name `DIALOG`/`BUTTONS` (label) and `ERRORS` (Option-4) as the target namespaces, not `BACKEND_TRANSLATIONS`.
   - Recommended resolution: spec uses standard namespaces; reserve `BACKEND_TRANSLATIONS` for the backend-emitted reset *email* body, not in-app copy.

None of these block the inventory; all three shape the spec. No code was changed.

---

## 7. Definition-of-done check (brief)

- ✅ §1 login surface: two dialogs documented, `LoginDialog.tsx` host identified, link-style primitive located (`LoginDialog.tsx:110-115`).
- ✅ §2 submit path + Option-4 attach point: chain quoted, catch block quoted (`authStore.ts:102-118`), `mapAuthError` quoted, social-account error behavior answered, verification-gate coexistence documented (both sync paths).
- ✅ §3 browser funnel: `Linking.openURL` pattern + sites listed; reset flow confirmed web-owned, nothing mobile-side to build for it.
- ✅ §4 inbound deep-link: `app.config.ts` scheme config quoted; **inbound browser→app deep-linking does NOT exist today** — answered definitively with evidence (no `+native-intent.ts`, no `Linking` inbound listeners, no custom `linking` config).
- ✅ §5 translation consumption: loading layer documented; `BACKEND_TRANSLATIONS` absence flagged; target namespaces named.

---

## 8. Config-file impact (per CLAUDE.md closure gate)

- **`app.config.ts`** — not edited (read-only audit; would be a forbidden config edit anyway). The success-step deep-link next session may need `ios.associatedDomains` / `android.intentFilters` *if* a real `https://` return link is chosen over a custom scheme — that is the cross-repo decision flagged in `.agent/audit-deep-links.md`, not this audit's call.
- **`state.md`** — no Expo-backlog row exists for `password-reset` yet. If the team wants to track mobile's slice, a row should be added by Docs/QA (I do not write `state.md`). Draft in §9.
- **`issues.md`** — no new entry authored here. One pre-existing adjacent observation (the `console.error` debug-logging line in `authStore.login`) is flagged in §9 for Mastermind to route, not written by me.
- **`conventions.md` / `decisions.md`** — no change.

---

## 9. For Mastermind

1. **🔴 The Option-4 detection mechanism is an open design decision, and it gates the mobile work.** A client-only, error-code-driven "use Google" message cannot reliably fire: Firebase returns the generic `auth/invalid-credential` for a social-account email+password attempt (email-enumeration protection), the app does no provider lookup, and `fetchSignInMethodsForEmail` is both absent and neutered by that protection. Pick one for the spec: (a) backend signal that the email is Google-registered; (b) relax Firebase email-enumeration protection (security-posture call, shared with web); (c) an always-on generic "if you signed up with Google, use the Google button" hint on every failed email+password login (no detection). Mobile implements whichever you choose — it cannot pick for you. (§2.4.)

2. **🟠 Login-error copy is hard-coded English in `mapAuthError`, not seeded/translated.** Decide whether Option-4 copy is a proper seeded `ERRORS` key (convention-correct per Part 6, but a lone translated string in an English-literal mapper) or matches the existing debt. Recommend the seeded key, with `mapAuthError`'s full i18n migration logged as separate pre-existing debt. (§2.3, §5.3.)

3. **Reset-password entry button is low-risk and well-supported.** It is a single link-style `<Pressable>` in `LoginDialog.tsx` (template at `:110-115`), label from a `DIALOG`/`BUTTONS` key, action = open the web reset URL via `Linking.openURL` (the established funnel). No new primitive, no new state. (§1.3, §3.)

4. **Inbound browser→app deep-linking is genuinely absent.** The deferred success-step deep-link starts from zero on the handler side; only the per-tier custom scheme + expo-router defaults exist. Scope it against `.agent/audit-deep-links.md` (custom-scheme = app-installed-only, no web `.well-known`; real `https://` return = cross-repo, web-blocking, needs `app/+native-intent.ts` for the locale-prefix mismatch). The scheme the web scaffold targets per tier is `oglasino://` / `oglasino-preview://` / `oglasino-dev://`. (§4.)

5. **Suggested `state.md` Expo-backlog row (Docs/QA to add — I can't write `state.md`):**
   > `| Password reset | not-started | Mobile slice: (a) link-style "Reset password" button in LoginDialog.tsx → opens web /reset via Linking.openURL; (b) Option-4 "use Google" copy on social-account login fail — BLOCKED on detection-mechanism decision (Firebase email-enumeration protection makes the social-account case indistinguishable from wrong-password client-side). Success-step inbound deep-link deferred. See oglasino-expo .agent/audit-password-reset.md. |`

6. **Part 4b adjacent observation (pre-existing, out of scope, not fixed):** `authStore.ts:113` (`console.error('Email login failed', err)`) and `:135`/`:149`/`:164` (sibling `console.error`s in register/google/facebook catches) are ad-hoc debug logging — a standing Part 4 cleanliness item in the auth store. Severity low (developer-facing only). Not fixed (out of scope, read-only audit). Route to a future auth-cleanup touch. The Facebook scaffolding around `loginWithFacebook` is already tracked (issues.md 2026-06-01 "Orphaned Facebook sign-in scaffolding").

   **Part 4a simplicity evidence:** Added — nothing (read-only audit). Considered and rejected — nothing. Simplified or removed — nothing.

---

## Conventions check

- **Part 4 (cleanliness):** read-only audit; no code, no imports, no debug logging added. Confirmed. One pre-existing Part 4 violation (the auth-store `console.error`s) flagged in §9.6, not fixed (out of scope).
- **Part 4a (simplicity) / Part 4b (adjacent observations):** structured evidence in §9.6.
- **Part 6 (translations):** the central translation finding — mobile has no `BACKEND_TRANSLATIONS` namespace; auth copy flows through `DIALOG`/`ERRORS`/`BUTTONS`; login errors are currently un-seeded English literals — is documented in §5 and §6 and flagged for the spec. No translations added.
- **Part 7 (error contract):** N/A — the Option-4 case is a Firebase client-SDK error (`auth/*`), not the backend `{errors:[{field,code,translationKey}]}` envelope. Documented as such (§2.4).
- **Part 11 (trust boundaries):** noted — any backend-side Option-4 detection (§9.1 option a) must derive "email uses Google" server-side, not trust a client claim.
- **Hard rules:** no config-file edits (`app.config.ts` read only), no other-repo reads/edits, no git ops, no deploys, no staging. `app.config.ts` scheme block and the native-scheme facts were cross-checked against the prior `.agent/audit-deep-links.md` (which verified the native manifests directly).

**Cleanup performed:** none needed (no code changed).
