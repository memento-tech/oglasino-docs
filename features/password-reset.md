# Password Reset

Branded password-reset for Oglasino, built on the email-notifications building blocks. A user who registered with email/password can recover access: they request a reset, receive a branded email with a Firebase reset link, land on a web `/reset` page, set a new password client-side, and return to logging in. Web owns the entire flow; both web and mobile funnel to it. Two emails ship — the reset email and a "you signed up with Google" email for social accounts — both branded HTML + plain-text fallback, both four locales.

**Slug:** `password-reset`
**Branches:** backend `dev`, web `dev`, mobile `new-expo-dev`, router `stage` (no router code change — §8).

This spec covers the whole feature surface. The success-step app-deep-link is **scaffolded but deferred** (§9); it does not block `shipped`. The account-method-linking collision is **out of scope** (§9, and a separate `issues.md` entry).

---

## 1. Overview

### What ships

1. **A reset-request endpoint** — `POST /api/auth/request-password-reset`, a sibling of `/auth/resend-verification`: unauthenticated, email-only, per-email Redis throttle (60s + 4/day) on **separate keys**, no account-existence leak, coded responses.
2. **A branded reset email** — minted via Admin SDK `generatePasswordResetLink`, sent via Brevo, carrying a link to the web `/reset` page. Four locales.
3. **A branded "use your social sign-in" email** — sent when the email resolves to a social-only account (no password credential). Explains the account uses a social provider; carries no reset link. Four locales.
4. **A web reset-request page** — `/[locale]/forgot-password`: enter email, call the endpoint, see a conditional "if an account exists…" message + a backend-driven countdown. Both platforms funnel here.
5. **A web reset page** — `/[locale]/reset?oobCode=…`: validate the code, take a new password, call `confirmPasswordReset` client-side, show a branded success step. Mirrors `/verify`.
6. **A "Reset password" entry link** — a link-style text button in `LogInDialog`, on **both** web and mobile. Web navigates to `/forgot-password`; mobile opens that web page via `Linking.openURL`.
7. **A success-step app-deep-link scaffold** — built-but-dormant on the reset page's success step, loudly TODO'd, wired so the inbound handler Igor builds in a later session works out of the box (§9).

### What this feature deliberately does NOT do

- **No login-screen provider hint ("Option 4").** Killed in planning — see §3 and `decisions.md`. The login screen keeps its current generic "Invalid email or password." All provider help moves to the email channel (the "use social" email).
- **No password ever touches our backend.** Password change is a pure client-side Firebase operation (`confirmPasswordReset`); the backend only mints the email link. There is **no schema change** (§5).
- **No native mobile reset UI** beyond the entry button — the browser + web own the request and confirm flow (§7).

---

## 2. Architecture

### The reset-request endpoint — sibling of resend-verification

`POST /api/auth/request-password-reset`, mirroring `AuthController.resendVerification` almost 1:1:

- **Unauthenticated, email-only body.** A new single-field record `RequestPasswordResetRequest(String email)` (no `@Valid`/Jakarta constraints — controller does its own `trimToNull` + `EMAIL_REQUIRED`, matching the resend DTO). Email lowercased (`toLowerCase(Locale.ROOT)`) before keying. Permitted by the existing `/api/auth/**` `permitAll` + filter-skip in `SecurityConfig`.
- **Two Redis throttles on SEPARATE key prefixes** from resend-verification, so a user near their verification-resend limit is not also locked out of password reset:
  - `pwreset:cooldown:<email>` — `setIfAbsent(key,"1",60s)` (SET-NX-EX); on slot held → 429 `PASSWORD_RESET_COOLDOWN` with `retryAfterSeconds` from `getExpire`.
  - `pwreset:daily:<email>` — `increment`; on first increment stamp `expire(key, 24h)`; on the 5th attempt release the cooldown slot just taken and return 429 `PASSWORD_RESET_DAILY_LIMIT`.
  - **Release semantics identical to resend:** limits are **held** on every no-leak no-op branch (so timing/limits cannot enumerate); limits are **released** (delete cooldown + decrement daily) **only** on a send failure (`MailException`).
- **Server-side branching (§3)** decides whether and which email to send. **All branches return the same generic success body** (`200 PASSWORD_RESET_REQUESTED`, carrying `retryAfterSeconds`).

### The send path & branded shell — reuse, build nothing new

Reuse the shipped building blocks verbatim: `EmailService.sendHtml(to, subject, htmlBody, textFallback)`, the branded `EmailLayout.wrap(...)` shell (logo from the R2 CDN), the plain-text fallback assembly pattern, `getBackendTranslation(key, langCode)` (`.orElseThrow()` — every key in all four locales), and `WebLocales.compound(langCode)`. Two new senders live alongside `VerificationEmailSender` in `service.impl` (so they can reuse the package-private `WebLocales`), following the same string-assembly (no template engine) pattern.

### Link minting — the same oobCode extract-and-rewrite trick

The reset sender calls `FirebaseAuth.getInstance().generatePasswordResetLink(email, settings)` with the same `ActionCodeSettings` shape verification uses (`.setUrl(webProperties.getBaseUrl())`, `.setHandleCodeInApp(false)`), then **discards Firebase's link and keeps only the `oobCode`** (parse with `UriComponentsBuilder`, pull `oobCode`, throw if blank), rebuilding as:

```
<webBase>/<compoundLocale>/reset?oobCode=<oobCode>
```

`<webBase>` = `WebProperties.getBaseUrl()` (apex host per env — dev `https://stage.oglasino.com`). `<compoundLocale>` = `WebLocales.compound(langCode)` (`sr→rs-sr`, `en→rs-en`, `ru→rs-ru`, `cnr→me-cnr`). `generatePasswordResetLink` returns the same `oobCode` query shape as `generateEmailVerificationLink`, so the trick carries unchanged. A `FirebaseAuthException` from minting propagates as 500 (distinct from the 502 SMTP path) — but minting only happens after provider detection has already succeeded for that email, so the email is known to exist.

### Recipient language

From the user row — `recipient.getPreferredLanguage().getCode()` — for both emails (both branches have a real user row, since unknown emails send nothing). No `X-Lang` fallback is needed on a send path here.

---

## 3. The provider gate & no-leak contract (trust boundary — conventions Part 11)

This is the heart of the feature. The endpoint's only client input is the **email**. Everything else is server-derived, and **the response is identical across every case** so the form cannot be used to enumerate accounts or providers.

### The four account states, and the identical response

After the throttle passes, the server branches (in this order):

| # | State | Detection (server-side) | Action | On-screen / response |
|---|---|---|---|---|
| 1 | **Banned** | `userAuditService.isEmailBanned(email)` (SHA-256 hash lookup, retention window) | **No email** | generic success |
| 2 | **Unknown** | Admin SDK `getUserByEmail(email)` throws `FirebaseAuthException` / `USER_NOT_FOUND` | **No email** | generic success |
| 3 | **Has a password credential** | `getUserByEmail(email).getProviderData()` contains a `UserInfo` with `getProviderId() == "password"` | **Reset email** (with link) | generic success |
| 4 | **Social-only** (no password) | provider list present but **lacks** `password` (e.g. only `google.com`) | **"use social" email** (no link) | generic success |

**The provider rule is "contains `password`," not "is social."** Because the Firebase project is set to **"Link accounts that use the same email"** (confirmed by Igor, 2026-06-03 — one Firebase user per email, multiple providers *linkable* to it), `getProviderData()` can list **both** `password` and `google.com` for one account. If `password` is among them, the user *has* a password to reset → branch 3 (reset email), regardless of whether Google is also linked. Branch 4 fires only when no `password` entry exists at all.

> ⚠️ **Provider detection MUST use the Admin SDK `getUserByEmail(email).getProviderData()` path — NOT the persisted `User.registeredWithProvider` column**, which is garbage (stores the firebase-claim Map's `toString()`; `issues.md` 2026-06-03). The DB lookup (`userRepository.findByEmail`) gives existence only, not the live provider list. Note: `getUserByEmail` in this codebase resolves to the *Postgres* lookup by default; the provider check must explicitly call `FirebaseAuth.getInstance().getUserByEmail(...)` (Admin SDK), which is **available in firebase-admin 9.9.0 but unused today** — new wiring.

### No leak, no lie

- **No leak:** branches 1–4 return the byte-identical generic success body and code, and consume the rate limits identically, so neither response content nor limit-state distinguishes the cases. (Send-vs-no-send timing is a known, accepted residual carried from the resend-verification posture; the throttle bounds it.)
- **No lie:** every **real** account receives **an** email — branch 3 a reset link, branch 4 a "use social" explanation. Only a **non-existent** email (branch 2) and a **banned** email (branch 1) receive nothing — and there is no real person at an unknown address to mislead; the banned case is deliberate (a banned user must not reset back in; ban communication is owned by the reblock flow).
- **On-screen copy is conditional, never a promise:** "If an account exists for this email, you'll receive an email shortly." True in all four cases.

### The "use social" email (branch 4)

Branded, no link. Copy explains the account uses a social sign-in and to use that provider's button to log in. The provider display name is interpolated (`%s` placeholder filled via `String.formatted`), so the backend maps the detected social `providerId` → a display name (`google.com` → "Google"). For the launch set that is Google only (Facebook scaffolding is orphaned/dead — `issues.md` 2026-06-01); adding a provider later is copy-free.

### Banned check ordering

Check banned (cheap hash lookup, pre-user-row) **before** the Admin SDK lookup, so a banned email never reaches link-minting or provider detection. A banned email always lands in branch 1.

---

## 4. The two emails — detail

### 4.1 Password-reset email (branch 3)

- **Trigger:** reset-request endpoint, branch 3 (email has a `password` provider, not banned).
- **Mechanism:** `generatePasswordResetLink` → extract `oobCode` → rebuild `<webBase>/<locale>/reset?oobCode=…` → `EmailService.sendHtml`.
- **Recipient language:** user row `preferred_language.code`.
- **Keys:** `email.password.reset.{subject,heading,body,cta,note}` + shared `email.common.signoff.{regards,team}`.

### 4.2 "Use social sign-in" email (branch 4)

- **Trigger:** reset-request endpoint, branch 4 (social-only, no password provider).
- **Mechanism:** direct `EmailService.sendHtml` — **no link minted** (there is no password to reset). Provider display name interpolated via `%s`.
- **Recipient language:** user row `preferred_language.code`.
- **Keys:** `email.password.social.{subject,heading,body,cta,note}` (body carries a `%s` provider-name placeholder) + shared signoff.

---

## 5. Schema changes — NONE

There are **no schema changes**. The password lives only in Firebase; `confirmPasswordReset` is a client-side Firebase operation the backend never sees. The throttle uses Redis (runtime keys, not schema). This is a clean contrast with email-notifications, which folded two columns.

---

## 6. Web (`oglasino-web`, `dev`)

### 6.1 Reset-request page — `/[locale]/(portal)/(public)/forgot-password/page.tsx`

- `(public)` placement (out of `SessionGuard`), same rationale as `/verify` (opened cold/unauthenticated). `[locale]` validated by the `[locale]` root layout's `notFound()`; use `useRoutingLocale()` for URL building, `useTranslations()` for copy.
- Single email input + submit. On submit, `POST /auth/request-password-reset` with `{ email }` only (no session) via `BACKEND_API`. Branch on coded responses, mirroring the resend-verification client service (`emailVerificationService.ts`): success → conditional "if an account exists…" message + backend-driven countdown (`retryAfterSeconds`); `PASSWORD_RESET_COOLDOWN` → re-seed countdown; `PASSWORD_RESET_DAILY_LIMIT` → limit copy; everything else (incl. `EMAIL_SEND_FAILED`) → generic failure copy.
- **Both platforms funnel here** (mobile opens this page in the browser — §7), which is why it is a standalone page, not a dialog.

### 6.2 Reset page — `/[locale]/(portal)/(public)/reset/page.tsx` (+ `'use client'` child)

Mirrors `/verify` structure (async server component awaits `searchParams`, extracts `oobCode`, passes to client child; `(public)` placement). The client child is a state machine richer than verify because it has an input step:

`'verifying-code' → 'form' → 'submitting' → 'success' | 'error'`

- On mount: `verifyPasswordResetCode(auth, oobCode)` to validate the code (and recover the target email to display). Throws (expired / invalid / already-used) → `'error'` (branded, with a "request a new link" affordance back to `/forgot-password`).
- `'form'`: new-password + confirm-password inputs. On submit: `confirmPasswordReset(auth, oobCode, newPassword)`. `auth/weak-password` (Firebase policy) → inline error, stay on form; confirm-mismatch → inline error.
- `'success'`: branded "password changed" + (a) the **live** "Log in here" fallback (opens `LOGIN_OPTIONS_DIALOG` / `router.push(\`/${locale}\`)`, same as verify's `goToLogin`) and (b) the **dormant, TODO'd** "Open in app" deep-link affordance (§6.4 / §9).
- **Trust boundary (Part 11):** success is set **only** when `confirmPasswordReset` resolves — never from `oobCode` presence (initial state is `'error'` when `oobCode` is missing). Mirror the `/verify` pattern (`VerifyEmailClient.tsx:26,39`, locked by `emailVerificationService.test.ts`). Add a sibling unit test asserting success is derived only from `confirmPasswordReset` resolving, never from `oobCode` presence.
- Firebase client capability confirmed present: `confirmPasswordReset`, `verifyPasswordResetCode` importable from `firebase/auth` (`firebase@12.13.0`), same import shape as the already-used `applyActionCode`.

### 6.3 "Reset password" entry link in `LogInDialog`

A link-style ghost Button — the existing primitive `variant="ghost" className="h-auto p-0 … underline underline-offset-3"` (`LogInDialog.tsx:162-170`) — seated below the password input. It `onClose()`s the login dialog and `router.push(\`/${locale}/forgot-password\`)`. No new dialog, no `DialogId`.

### 6.4 Success-step deep-link scaffold (dormant — see §9)

Built-but-dormant. Centralize the per-tier app-scheme target in **one constant/helper** so the later session wires the real inbound handling in one place. The "Open in app" affordance calls that helper (currently a best-effort no-op / web-fallback) and carries a loud `// TODO(deep-link, next session)` marker. There is **no** existing `.well-known` / app-association infra in web today (footer store badges are dead links — `issues.md` 2026-05-31), so this scaffold is greenfield; the later session also owns `apple-app-site-association` / `assetlinks.json` if a real `https://` return is chosen over a custom scheme.

### 6.5 Translations web must IDENTIFY (Backend seeds — web does not write SQL)

Per conventions Part 6, web identifies keys; Backend seeds them in the four `0001-data-web-translations-*.sql` files. Web consumes via next-intl. Observe the parent/child collision rule (Part 6 Rule 2) — suffix leaves (`.label`/`.title`), never collide a parent with a leaf.

- **DIALOG:** `reset.request.{title,body,sent,countdown}`, `reset.page.{verifying,form.title,form.body,success.title,success.body,error.title,error.body}`.
- **BUTTONS:** `reset.password.label` (entry link), `reset.request.submit.label`, `reset.page.submit.label`, `reset.page.success.login.label`, `reset.page.success.app.label` (dormant), `reset.page.error.retry.label`.
- **ERRORS:** `reset.request.dailylimit`, `reset.request.failed`, `reset.page.code.invalid`, `reset.page.weak.password`, `reset.page.mismatch`.
- **INPUT:** `password.new.label`, `password.confirm.label`.
- **Untouched:** the frozen `VALIDATION` namespace (new error-like keys go to `ERRORS`).

---

## 7. Mobile (`oglasino-expo`, `new-expo-dev`)

Mobile's **entire** scope this feature is one entry button. No reset flow, no provider logic, no Option-4 copy (killed — §3).

- **A link-style "Reset password" `<Pressable>` in `LoginDialog.tsx`**, seated under the password input (template: the in-file "or social" link, `LoginDialog.tsx:110-115`, `text-primary underline`). Label from a `DIALOG` key (`reset.password.label`) consumed via `tDialog(...)` — mobile has **no** `BACKEND_TRANSLATIONS` namespace; auth copy flows through `DIALOG`/`ERRORS`/`BUTTONS`, fetched at boot. (If the team prefers grouping with button labels, `BUTTONS` is equally valid — engineer's call, note it.)
- **Action:** `Linking.openURL` (the established outbound funnel) to the web reset-request page. Construct `<webBase>/<compoundLocale>/forgot-password`, where:
  - `<webBase>` is the **tier-matched** web origin (prod `https://oglasino.com` / stage `https://stage.oglasino.com`), selected from the app's `ENV` (`app.config.ts`).
  - `<compoundLocale>` is derived from the app's active language via a small lang→compound map mirroring backend `WebLocales.compound` (`sr→rs-sr`, etc.); if unavailable, fall back to the bare `<webBase>/forgot-password` and let web's proxy assign the default locale.
- **No Option-4 copy.** `mapAuthError` (`authErrors.ts`) is untouched; its hard-coded-English login-error literals remain pre-existing debt (§9). The login path is not modified at all.
- **No new state, no new primitive.** Coexistence with the email-verification sign-out gate is a non-issue: the reset button is a **pre-login affordance** that only opens a browser — it touches no auth path, fires no `signInWithEmailAndPassword`, and sits structurally apart from both sync-path gates.

---

## 8. Router (`oglasino-router`, `stage`) — no code change

Mirrors the verify finding. `/[locale]/forgot-password` and `/[locale]/reset?oobCode=…` on the apex host are ordinary frontend traffic — not caught by the `/api/*`, admin, mobile, or maintenance branches — and forward to the Vercel origin with the query string passed through **verbatim**. No worker change, no new binding.

- **Email-link host must be the apex** (`oglasino.com` prod / `stage.oglasino.com` stage), **never `api.oglasino.com`**. This holds by construction: the reset sender builds from `WebProperties.getBaseUrl()`, already configured to the apex.
- **Maintenance edge case (product note, not a bug):** a reset click *during maintenance* returns 503; if the Firebase `oobCode` expires before maintenance ends, the link fails and the user needs a fresh email (the request path covers this).

---

## 9. Out of scope — deferred

Each recorded with *why* and *what it would take*. None blocks `shipped`.

- **Success-step inbound app deep-link.** Igor builds it in a later session. The web scaffold is dormant + TODO'd (§6.4). Inbound browser→app handling does **not** exist in the app today — no `app/+native-intent.ts`, no `Linking.getInitialURL`/`addEventListener('url')`, no custom `linking` config; only the per-tier custom schemes (`oglasino://` prod / `oglasino-preview://` / `oglasino-dev://`) + expo-router defaults. Scope against `oglasino-expo .agent/audit-deep-links.md` (custom-scheme = app-installed-only, no web `.well-known`; a real `https://` return is cross-repo, web-blocking, and needs `app/+native-intent.ts` for the locale-prefix mismatch). The scheme the web scaffold targets per tier is a cross-repo contract Igor brokers next session.
- **Account-method-linking collision** (separate `issues.md` entry — ITEM 5 of this brief, and its own future Mastermind feature). Under the confirmed "Link accounts that use the same email" setting, Firebase can link a password credential to an existing social account, so the collision is partially mitigated at the platform level — but the **in-app linking UX is unbuilt**, and a social-only account still fails email+password login with a generic error. **Not a reset blocker** — reset is correct regardless (social-only accounts get the provider-gated "use social" email, never a reset link).
- **`mapAuthError` full i18n migration** (web + mobile hard-coded-English login errors) — pre-existing debt; untouched here (Option 4 killed, so no new login-error copy is added). Logged for a future auth-cleanup touch; the orphaned Facebook scaffolding (`issues.md` 2026-06-01) and the auth-store `console.error` debug logging (low severity, developer-facing) are natural companions for that sweep.
- **Optional: generalize the resend-verification client service** into a shared "email-action request" helper — deferred; do not over-abstract for one new caller (Part 4a) unless the engineer finds it clean in passing.

---

## 10. Trust boundaries (conventions Part 11)

| Surface | Client-supplied | Server-derived / authoritative |
| --- | --- | --- |
| Reset-request endpoint | the email | existence (Admin SDK `USER_NOT_FOUND`), provider list (`getProviderData()` contains `password`?), banned (`isEmailBanned` hash), language (user `preferred_language`); **no account-existence leak** (identical generic success across banned/unknown/social/password; limits consumed identically); abuse bounded by 60s + 4/day per-email throttle on separate `pwreset:*` keys |
| Reset link minting | nothing | Admin SDK `generatePasswordResetLink` server-side |
| New password | the new password + `oobCode` (Firebase-validated; cannot forge "reset") | `confirmPasswordReset` resolving in Firebase; the password **never** reaches our backend |
| Reset-page success | the `oobCode` | `confirmPasswordReset` resolving (never `oobCode` presence) |
| Email envelope | nothing | From / From-name / Reply-To from `EmailProperties` (env-injected) |
| Recipient address (both emails) | nothing | the `User` row's email |
| Recipient language | nothing | `users.preferred_language_id` |
| Provider → display name | nothing | server-side map (`google.com` → "Google") |

---

## 11. Status ledger

Status syntax per conventions Part 1: `[ ]` not started · `[~]` in progress · `[x]` complete · `[!]` blocked.

**Foundation** — reuses shipped email-notifications blocks; no new foundation required.

**Backend**
- [x] `RequestPasswordResetRequest` DTO + `POST /api/auth/request-password-reset` (unauthenticated, email-only, lowercased)
- [x] `pwreset:*` Redis throttle (60s + 4/day, **separate keys**, resend release semantics)
- [x] Provider + banned branching (banned-check first; Admin SDK `getUserByEmail`/`getProviderData` "contains `password`"; no-leak identical responses; limits held on no-op)
- [x] Reset email sender (`generatePasswordResetLink` → `oobCode` rewrite → `/reset`)
- [x] "Use social" email sender (no link; provider-name `%s`)
- [x] Coded responses: 200 `PASSWORD_RESET_REQUESTED` (+`retryAfterSeconds`), 400 `EMAIL_REQUIRED`, 429 `PASSWORD_RESET_COOLDOWN` (+`retryAfterSeconds`), 429 `PASSWORD_RESET_DAILY_LIMIT`, 502 `EMAIL_SEND_FAILED`
- [x] `BACKEND_TRANSLATIONS` seeds: `email.password.reset.*` + `email.password.social.*` × 4 locales (+ reuse `email.common.signoff.*`)
- [x] Frontend-namespace seeds for web/mobile keys (`DIALOG`/`BUTTONS`/`ERRORS`/`INPUT`, §6.5 / §7) × 4 locales

**Web**
- [x] `/[locale]/(portal)/(public)/forgot-password` page (request + countdown + conditional message)
- [x] `/[locale]/(portal)/(public)/reset` page (`verifyPasswordResetCode` → form → `confirmPasswordReset`; success-from-resolve; unit test)
- [x] "Reset password" entry link in `LogInDialog` → `/forgot-password`
- [x] Success-step deep-link scaffold (dormant, TODO, single-constant target)

**Mobile**
- [x] "Reset password" link in `LoginDialog.tsx` → `Linking.openURL(<webBase>/<locale>/forgot-password)` with tier + compound-locale URL construction

**Router**
- [x] Confirm clean — no change (apex-host constraint holds by construction)

### Deferred (do NOT block `shipped`)
- [ ] Success-step inbound app deep-link — later session (handler does not exist today)
- [ ] Account-method-linking collision — separate feature (`issues.md` 2026-06-03)

---

## 12. Session log

- **2026-06-03 — planning:** Phase 1 intake, three Phase-2 audits (`.agent/audit-password-reset.md` in backend/web/mobile), Phase-3 seam analysis (five findings resolved; Option 4 killed; provider rule set to contains-`password` under the confirmed "link accounts" setting; banned → no email; mobile scoped to entry button only). Spec authored. Engineering briefs (Phase 5) pending.
- **2026-06-03 — Phase 5 landed (code-complete across all repos):** backend (874/0), web (293/0), mobile (422/0) briefs all landed. §11 ledger flipped to as-built — every Backend/Web/Mobile/Router row `[x]`; the two **Deferred** rows (success-step inbound app deep-link, account-method-linking collision) stay `[ ]`, both still out of scope. Feature is **`web-stable`**; mobile slice is **code-complete pending on-device Ψ smoke**. Reconciled in `state.md` (active-feature block + Expo-backlog row) and two `issues.md` entries (mobile share-URL host hardcode, web `Input` interpolated Tailwind width) — see the Docs/QA close-out session.
