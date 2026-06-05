# Email Notifications

Branded transactional email for Oglasino. Nine emails, all sent server-side via the existing Brevo SMTP relay, all branded HTML with a plain-text fallback, all copy in `BACKEND_TRANSLATIONS` across four locales. The feature also builds the **email-verification login gate** that does not exist today.

**Slug:** `email-notifications`
**Branches:** backend `dev`, web `dev`, mobile `new-expo-dev`, router `stage` (no router code change — see §8).

This spec covers the **whole** feature surface, including pieces deferred to later Mastermind chats. The status ledger in §11 marks what ships here versus what is deferred. This feature is `shipped` when the nine in-scope emails and the verification gate are live; the deferred items in §9 do **not** block `shipped`.

---

## 1. Overview

### The nine emails (in scope)

1. **Registration / verification** — sent when a new email account is created. Carries the verification link. Doubles as the welcome email (there is no separate welcome).
2. **Deletion requested (Day 0)** — sent when a user requests account deletion.
3. **Deletion reminder (Day 6)** — sent by a daily cron to users one day before their 7-day hard-delete.
4. **Deletion cancelled** — sent when a user cancels deletion by signing back in during the grace period.
5. **User banned** — sent when an admin bans a user.
6. **User restored** — sent when an admin unbans a user.
7. **Deletion locked by admin** — sent when an admin locks a user's account from deletion (Igor's "postpone" = the existing deletion-lock mechanism).
8. **Re-registration blocked** — sent once when a banned email attempts to re-register.
9. **New review received** — sent when a user receives a new (approved) review.

### Deferred (later Mastermind chats — see §9)

- **Product blocked** — requires product moderation to be built first (`ModerationState.BANNED` is defined but never set; no admin moderation action exists).
- **Password reset** — does not exist anywhere in web today; it is a full build, not a reuse.
- **Account-deleted confirmation (Day 7)** — not requested; would need the recipient email snapshotted before the in-transaction user-row delete.

### The verification gate

Today, on all three platforms, email registration logs the user straight in — **nothing enforces `emailVerified`** (verified by all four audits). This feature builds the gate: a user who registers with email/password cannot act until they verify. The gate is enforced **server-side** in `FirebaseAuthFilter` against the **live Firebase token claim**. Social-login accounts (Google/Facebook) arrive `email_verified=true` and are never blocked by this gate.

---

## 2. Architecture

### Event-driven, listeners in an email package

Emails are sent by `@TransactionalEventListener(phase = AFTER_COMMIT)` listeners that subscribe to **dedicated domain events** published at each state transition. The listeners live in the email feature's package. Domain services publish "X happened"; they do not know email listens. This mirrors the existing `ProductIndexerEventListener` and `ProductImagesRemovedEventListener` pattern (AFTER_COMMIT + `@Async` + `@Retryable`).

**Dedicated events, not the overloaded `UserStateChangedEvent`.** `UserStateChangedEvent` is already published by five different transitions (deletion-requested, deletion-cancelled, ban, unban, hard-delete) and is consumed for cache revalidation. A single email listener on it could not tell which transition fired. This feature introduces one event per emailed transition, additive to `UserStateChangedEvent` (which is untouched):

- `UserRegisteredEvent(userId)`
- `DeletionRequestedEvent(userId)`
- `DeletionCancelledEvent(userId)`
- `UserBannedEvent(userId)`
- `UserUnbannedEvent(userId)`
- `DeletionLockedEvent(userId)`
- `ReviewApprovedEvent(reviewId)`

The deletion reminder and the re-registration-blocked email are **not** event-driven — see §4.3 and §4.8.

### The AFTER_COMMIT caveat (ban / unban)

`disableUser` and `enableUser` (`admin/facade/impl/DefaultUsersFacade`) run **outside any Spring transaction** — they are sequences of self-committing `saveUser` calls plus audit inserts plus Firebase calls. The existing `UserStateChangedEvent` listener fires there only via `fallbackExecution=true` (synchronous, no commit boundary). The ban/unban **email** listeners must therefore use `fallbackExecution=true` + `@Async` and accept that there is no transactional rollback guard — there is nothing to roll back to, and a Brevo failure must never disturb the ban. **Do not** wrap ban/unban in a transaction to manufacture a commit boundary; the audit warns the saveUser→audit→Firebase ordering there is deliberate and fragile.

For the transitions that *do* run in a transaction (registration sync, deletion-requested, deletion-cancelled — the last two via `TransactionTemplate`), AFTER_COMMIT works properly and the email sends only after the row is durably committed.

### The send path

`EmailService` today exposes only `sendPlainText(to, subject, body)` — `multipart=false`, `setText(body, false)`, HTML unreachable. This feature adds an HTML method:

```java
void sendHtml(String to, String subject, String htmlBody, String textFallback);
```

building a `MimeMessageHelper(message, true, UTF-8)` with `setText(textFallback, htmlBody)`. `sendPlainText` stays for any plain-only need. The envelope (From / From-name / Reply-To) continues to come from `EmailProperties` (env-injected, server-controlled, never client-supplied).

### The branded shell

A single shared helper wraps any email body in the branded HTML shell — logo header (from a stable Cloudflare/R2 public URL Igor supplies), footer, locale-aware. Each email supplies its inner content. No template engine (no Thymeleaf/Freemarker) — a Java string-assembly helper, justified under Part 4a by nine same-shaped consumers. The plain-text fallback is assembled alongside.

### Copy and recipient language

All subjects and bodies live in `BACKEND_TRANSLATIONS`, seeded inline-append across all four locale files (`0001-data-web-translations-{EN,RS,CNR,RU}.sql`) per conventions Part 6 Rule 3 (the dedicated-file exception is multi-namespace only; this is one namespace). Read via `getBackendTranslation(key, langCode)`, which takes the language code explicitly — no request context needed, so cron sends work. `%s`-style placeholders filled with `String.formatted(...)`, matching existing notification copy.

**Recipient language** comes from `users.preferred_language_id` (the `User.preferredLanguage` association), read and passed explicitly — exactly as existing push-notification code does (`DefaultFavoriteProductFacade`, `DefaultAdminReviewService`). The one exception is the re-registration-blocked email, which has no user row (§4.8).

**Missing-key risk:** `getBackendTranslation` ends in `.orElseThrow()`. Every new key must exist in all four locales or a send throws — for cron sends an unguarded throw aborts the batch row. Seed all four locales for every key; the cron loop isolates per-row failures in try/catch regardless.

---

## 3. The verification gate (trust boundary — conventions Part 11)

### Current state (factual, all four audits)

Verdict from the backend audit: **(c) — nothing.** No server-side `emailVerified` check; Firebase native sign-in is not blocking either (the backend never asks). `FirebaseAuthFilter` blocks only on `disabled` (ban) and `PENDING_DELETION`. Web's `SessionGuard` checks only "is there a signed-in user" (+ admin). Mobile reads `emailVerified` nowhere. An unverified user has full access on every platform today.

`emailVerifiedExternal` is captured into Postgres once at registration from the token (`createUserSynchronized` → `token.isEmailVerified()`) and **never refreshed** — so the DB column goes stale the moment a user verifies after first login. Display-only today (`UserInfoDTO.verified`).

### Target

Enforce in `FirebaseAuthFilter`, reading `decoded.isEmailVerified()` off the **already-verified live Firebase token** the filter holds — not the stored DB column (stale), not a client-supplied value (forgeable). This is the trust boundary: the server decides, clients render the consequence.

- **Scope:** the gate blocks email/password accounts that are unverified. Social-login tokens arrive `email_verified=true`, so the gate never bites them. This is intended.
- **Placement:** the filter already holds the decoded token; reading the claim there needs no cache/DTO change and is always current. Do **not** add `emailVerified` to the cached `AuthenticatedUserDTO` / `OglasinoAuthentication` (it would inherit the stale-cache trap).
- **Rejection shape:** a coded response per Part 7 (a dedicated code, e.g. `EMAIL_NOT_VERIFIED`, 403) so clients branch on the code, not a message string.

### Client consequence — as shipped (strict client-detection sign-out model)

The feature **evolved past the original backend-rejection model during build.** As shipped, the client **detects** the unverified state itself and signs the user out — an unverified email/password user never holds a session, so there is no in-app "verify" action and the server gate is never the UX trigger.

- **The detection:** on email/password login or registration, the client reads the **live Firebase token** (`signInProvider === 'password' && !emailVerified`) and **immediately calls `signOut()`**, capturing the entered email. Social logins (`google.com`) arrive `email_verified=true` and fall through to normal entry. Verification happens **only** by clicking the emailed link, which opens the web `/verify` page (§6); there is **no in-app verify action** on either platform.
- **Web:** sign-out + a dismissable verify dialog showing the address, with an **email-based** Resend (the user is signed out, so resend takes the email, not a session) and a backend-driven countdown. `SessionGuard` carries **no** verify-state branch; `/verify` runs `applyActionCode` only (no auto-login, no token force-refresh — a fresh login yields a fresh verified token). See §6.
- **Mobile:** the same sign-out model, enforced on **both** sync paths (`initAuthListener` and the explicit `login()`/`register()` chokepoint) — the audit found the existing banned-check covered only `initAuthListener`, and email login fires both, so an unverified user would otherwise land half-authenticated on the explicit path. No verify page (the browser owns it). See §7.

`EMAIL_NOT_VERIFIED` remains the **server-side defense-in-depth gate** (§ Target above) — the real trust boundary — but the client gates itself before any secured call, so the server rejection is never the client UX trigger.

---

## 4. The nine emails — detail

Each: trigger, event/mechanism, transaction semantics, recipient + language, notes.

### 4.1 Registration / verification (= welcome)

- **Trigger:** a brand-new account, signalled by `wasRegister==true` in `AuthController.firebaseSync` (the account is created client-side by the Firebase SDK; the backend mirrors it on first sync). Web/mobile do **not** call `sendEmailVerification` today, so there is no competing Firebase-sent email.
- **Mechanism:** publish `UserRegisteredEvent(userId)` from the `wasRegister` branch; the email listener generates the verification link via Firebase Admin SDK `generateEmailVerificationLink(email, actionCodeSettings)` and sends the branded Brevo email. `ActionCodeSettings.url` points at the web verify page (§6).
- **Tx:** request-driven, commit boundary available — AFTER_COMMIT proper.
- **Language:** from the new `User.preferredLanguage` (captured at registration).
- **Notes:** the user is signed in the instant they register (Firebase session exists), but the client immediately signs them out under the §3 sign-out model until they click the emailed link. **Resend (as shipped):** `POST /auth/resend-verification` is **email-based and unauthenticated** (the user is signed out, so it takes the email in the body, not a session). It regenerates the link via Admin SDK and re-sends via Brevo — the client-side `sendEmailVerification` is the wrong tool (unbranded, Firebase-sent). It enforces a **60s gap + 4 per day** per-email Redis throttle (both released on send failure), has **no account-existence leak** (an unknown or already-verified address returns the same generic success; limits are *not* released on the no-leak skip so it can't be used to enumerate), and returns distinct coded responses: 200 `VERIFICATION_EMAIL_SENT` (carries `retryAfterSeconds`), 429 `VERIFICATION_RESEND_COOLDOWN` (carries remaining `retryAfterSeconds`), 429 `VERIFICATION_RESEND_DAILY_LIMIT` (no countdown), 502/`EMAIL_SEND_FAILED` on relay failure. The registration auto-send is exempt from the cap. The endpoint is part of this feature.

### 4.2 Deletion requested (Day 0)

- **Trigger:** `DefaultUserDeletionService.requestDeletion`.
- **Mechanism:** publish `DeletionRequestedEvent(userId)` inside the existing `writeTx.executeWithoutResult` block (alongside the current `UserStateChangedEvent`).
- **Tx:** `TransactionTemplate` — AFTER_COMMIT proper.
- **Language:** from the user.

### 4.3 Deletion reminder (Day 6 — cron)

- **Trigger:** a **new daily `@Scheduled` cron at 13:00**, selecting `user_deletion_requests` rows that are PENDING, whose `scheduled_deletion_at` is within the next ~24h, and whose new `reminder_sent_at` column is NULL. Send → stamp `reminder_sent_at`.
- **Mechanism:** direct `emailService` call inside the job's per-row try/catch (mirroring the existing `processScheduledDeletions` per-row isolation). **Not** event-driven — there is no transaction-commit-then-send here.
- **Why the column:** "email everyone 6 days old" without a flag double-sends or misses, because day-6 is a 24h window and the cron fires once a day on a boundary that drifts (DST, missed runs, exact-hour arithmetic). The `reminder_sent_at` guard makes it exactly-once. The column is email-feature-owned (it exists only so the reminder sends once) and is added to the V1 schema fold (§5).
- **Language:** load the user (or just `preferred_language.code`) per row.

### 4.4 Deletion cancelled

- **Trigger:** `DefaultUserDeletionService.cancelDeletionOnLogin` (called from `firebase-sync`).
- **Mechanism:** publish `DeletionCancelledEvent(userId)` inside the `writeTx` block.
- **Tx:** `TransactionTemplate` — AFTER_COMMIT proper.
- **Language:** from the user.

### 4.5 User banned

- **Trigger:** `DefaultUsersFacade.disableUser`.
- **Mechanism:** publish `UserBannedEvent(userId)`; listener `fallbackExecution=true` + `@Async` (no commit boundary — see §2 caveat).
- **Tx:** **none** — `fallbackExecution` semantics, no rollback guard.
- **Language:** from the user.
- **Notes:** the email should explain the action and how to appeal (per the legal drafts / User Deletion spec posture).

### 4.6 User restored / unbanned

- **Trigger:** `DefaultUsersFacade.enableUser`.
- **Mechanism:** publish `UserUnbannedEvent(userId)`; same `fallbackExecution` + `@Async` shape as ban.
- **Tx:** **none** — same caveat.
- **Language:** from the user.

### 4.7 Deletion locked by admin ("postpone")

- **Trigger:** the admin lock site — `lockFromDeletion` / `UserDeletionLock` creation in `DefaultUserDeletionService`. (Igor's "postpone" maps to the existing deletion-lock mechanism, which blocks the hard-delete cron; there is no separate "move the scheduled date" capability and none is built here.)
- **Mechanism:** publish `DeletionLockedEvent(userId)` at the lock site.
- **Tx:** request-driven (admin action).
- **Language:** from the user.

### 4.8 Re-registration blocked

- **Trigger:** `AuthController.firebaseSync` rejects a banned email via `userAuditService.isEmailBanned(email)` (queries `banned_user_audit` by email-hash, `retention_until > now`) and returns 403 `EMAIL_BANNED` — **before** any user row exists.
- **Mechanism:** **send once, ever, per ban.** A new nullable column on `banned_user_audit` (e.g. `reblock_notified_at`) guards it: on a blocked re-reg attempt, if the flag is NULL, send the "your account is banned, here's how to appeal" email and stamp the flag; if already stamped, reject silently. Direct `emailService` call in the reject branch — **not** event-driven (nothing commits a user).
- **Self-clearing on unblock:** the flag lives on the `banned_user_audit` record. The intended design is that it self-clears because a future re-ban creates a *fresh* audit row with a NULL flag. **This hinges on whether unban (`enableUser`) removes/expires the `banned_user_audit` row or merely flips the user's `disabled` flag — the backend audit did not trace this. The implementing brief must verify the unban ↔ `banned_user_audit` relationship in code and adjust:** if unban leaves the audit row lingering, add an explicit flag-clear at `enableUser`; if unban removes/replaces the row, the self-clearing design holds with no extra step.
- **Tx:** pre-tx reject — no user, no transaction.
- **Language:** **not available from a user row** (none exists). Use the request `X-Lang` header or a default. Recorded as an accepted constraint.
- **Abuse note:** sending on every blocked attempt is a spam vector; the send-once flag *is* the rate limit — exactly one email per ban, never again until re-banned.

### 4.9 New review received

- **Trigger:** review approval — `admin/service/impl/DefaultAdminReviewService` (the point where the target user is already notified by an existing **push** notification, `notif.review.new.*`), not raw review creation.
- **Mechanism:** publish `ReviewApprovedEvent(reviewId)` at the same point the push fires, with an email listener mirroring the push. (The existing push is a *direct* call, not event-driven; the event is the brief's stated direction and keeps email isolated in its own listener. Engineer may note the surrounding-style mismatch under Part 4a, but the event keeps the email package decoupled.)
- **Tx:** request-driven (admin approves).
- **Language:** the target user's `preferred_language.code`, already resolved at that site for the push.

---

## 5. Schema changes (pre-production V1 fold, conventions Part 12)

Both columns edited into `V1__init_schema.sql` in place (pre-production, no environment carries production data). Both are email-feature-owned.

1. **`user_deletion_requests.reminder_sent_at TIMESTAMP NULL`** — guards the once-only deletion reminder (§4.3).
2. **`banned_user_audit.reblock_notified_at TIMESTAMP NULL`** (final name at engineer's discretion) — guards the once-only re-registration-blocked email (§4.8).

No other schema changes. `emailVerifiedExternal` already exists and is not used by the gate (the gate reads the live token, §3).

---

## 6. Web (`oglasino-web`, `dev`)

- **New verify page:** `app/[locale]/(portal)/(public)/verify/page.tsx` (+ a `'use client'` child). `(public)` placement keeps it out of `SessionGuard` (email links open in fresh/unauthenticated sessions). Reads `oobCode` from `searchParams`; the client child calls `applyActionCode(auth, oobCode)` from `firebase/auth` (already a dependency, v12). Success → branded "verified, log in now"; failure (expired / invalid / already-used) → branded error + resend affordance. Success is derived strictly from `applyActionCode` resolving — never from the mere presence of `oobCode` (Part 11).
- **`[locale]` is a compound routing locale** (`rs-sr`, `me-cnr`, …). The email link must embed a valid one from `routing.ts` or `[locale]/layout.tsx` calls `notFound()`.
- **Register UX:** after registration, show a "check your email" state instead of `router.refresh()`-ing the user into the app.
- **Login-blocked UX:** when the backend gate rejects an unverified login (the `EMAIL_NOT_VERIFIED` code), surface a translated "verify your email" state with the resend affordance.
- **Resend:** one `BACKEND_API.post` to the new `/auth/resend-verification` endpoint + a button.
- **Untouched:** password reset (untouched by this feature; built later as a separate feature — `features/password-reset.md`; §9), routing locales, `next.config.ts`.

## 7. Mobile (`oglasino-expo`, `new-expo-dev`) — as shipped

- **Client-side gate, not a backend-rejection consumer.** The client detects the unverified email/password state off the live Firebase token (`isAwaitingEmailVerification`: `signInProvider === 'password' && !emailVerified`) and **signs the user out immediately**, on **both** sync paths — the explicit `login()`/`register()` chokepoint (`buildUserSession` signs out + throws a typed `EmailNotVerifiedError`, caught by the store which seeds `awaitingVerificationEmail` and never sets `user`) and `initAuthListener` (`onIdTokenChanged`, which `signInWithEmailAndPassword` also fires). The audit's key finding was that the existing banned-check covered only `initAuthListener`; both paths are now gated, so an unverified user cannot slip in half-authenticated on the explicit path. Social logins (`google.com`) are exempt.
- **A dismissable verify dialog** surfaces off the `awaitingVerificationEmail` flag via the existing `AccountStateDialogsInit` pattern, with an **email-based** Resend (unauthenticated `POST /auth/resend-verification` with `{ email }` — the user is signed out) that branches on the backend **code** (`VERIFICATION_EMAIL_SENT` / `VERIFICATION_RESEND_COOLDOWN` / `VERIFICATION_RESEND_DAILY_LIMIT` / `EMAIL_SEND_FAILED`) and a backend-seeded countdown. No verify page (the browser owns it; the email → browser → app round-trip is unobstructed).
- `EMAIL_NOT_VERIFIED` is the server's defense-in-depth gate, **not** the mobile UX trigger (the client gates itself first).
- The inert `account-verification.tsx` stub + its `UserMenu` "verify account" entry point are left as-is for a later Ω sweep — in this model an unverified user is never logged in, so that dashboard entry now leads nowhere (see §9 and the issues.md cleanup note).

## 8. Router (`oglasino-router`, `stage`) — no code change

Confirmed by audit: `/[locale]/verify?oobCode=...` on the apex host is ordinary frontend traffic — not caught by the `/api/*`, admin-regex, mobile, or maintenance branches — and forwards to the Vercel origin with the `oobCode` query string passed through **verbatim** (`${path}${search}` construction; `redirect: "manual"` additionally passes any verify-page redirect back to the browser). No worker change, no new binding.

- **Email-link host must be the apex** (`oglasino.com` prod / `stage.oglasino.com` stage). **Never `api.oglasino.com`** (that forwards to the backend, not Vercel). `www.` works but adds a 301 hop. Keep `verify` as its own segment under the locale — not nested under `/admin`, not prefixed `/api`.
- **Maintenance edge case (product note, not a bug):** a verify click *during maintenance* returns 503; if the Firebase `oobCode` expires before maintenance ends, the link fails and the user needs a fresh email (the resend path covers this).

## 9. Out of scope — deferred to future Mastermind chats

Each is recorded with *why* and *what it would take*, so the next Mastermind starts from here. None blocks this feature reaching `shipped`.

- **Product-blocked email.** `ModerationState.BANNED` is defined but **never set** anywhere; there is no admin product-moderation action, endpoint, or service. This is a **product-moderation feature** (build the block transition transactionally, admin UI, service) with an email at the end — not email work. The email listener (`ProductBlockedEvent`) is downstream of that build. Defer to a moderation feature.
- **Password reset.** Did **not exist** in web at the time of this feature — no `sendPasswordResetEmail`, no forgot-password link, no route, no reset UI. It was a full build (forgot-password entry + the reset flow), not a "reuse Firebase" confirm. **Built separately 2026-06-03 as the `password-reset` feature — see `features/password-reset.md` (gap closed).**
- **Account-deleted confirmation (Day 7).** Not requested. Would need the recipient email **snapshotted before** the in-transaction user-row delete in `runHardDelete` (which today snapshots only `firebaseUid`/`profileImageKey`/`wasBanned`). Note for whoever wants it.

### Cleanups deferred (non-blocking — for a future tidy-up)

- **Two dual-key translation redundancies.** `verify.email.resend.dailylimit` / `email.verification.daily_limit` and `verify.email.resend.failed` / `email.send_failed` — each twin pair carries identical copy. Drop the unused key once it's confirmed which one web actually renders per surface.
- **Inert mobile `account-verification.tsx` stub + dead `UserMenu` entry point.** Under the sign-out model an unverified user is never logged in, so the dashboard "verify account" entry leads nowhere → Ω teardown candidate, alongside the orphaned Facebook scaffolding already logged (issues.md 2026-06-01).
- **Optional: relocate the email classes** (`EmailLayout`, the senders, `WebLocales`, the listeners) into a dedicated `email` package — deferred throughout to avoid churning shipped code mid-feature.

### Close-out notes

- **Deliverability.** The first test email junked because SPF didn't include Brevo; after the SPF fix it inboxes. No longer a launch blocker — downgraded to a **monitor cold-domain reputation** item.
- **Native-translator review.** Completed across all four locales (EN/RS/RU/CNR) — satisfies the "native review at close" item. (RU is Latin-transliterated per the house convention, not Cyrillic — see the handoff doc.)

## 10. Trust boundaries (conventions Part 11)

| Surface | Client-supplied | Server-derived / authoritative |
| --- | --- | --- |
| Verification gate | nothing trusted from client | `emailVerified` read off the verified live Firebase token in `FirebaseAuthFilter` (server defense-in-depth; the client also gates itself off the live token before any secured call — §3) |
| Resend endpoint (`/auth/resend-verification`) | the email to resend to (it *is* the identity; the user is signed out) | **no account-existence leak** (unknown / already-verified → same generic success; limits not released on the no-leak skip, so it can't enumerate); abuse bounded by a per-email **60s + 4/day** Redis throttle |
| Email envelope | nothing | From / From-name / Reply-To from `EmailProperties` (env-injected) |
| Recipient address (8 user-bound emails) | nothing | the `User` row's email |
| Recipient address (re-reg blocked) | the email being rejected (it *is* the identity) | matched against `banned_user_audit` by hash; no user row |
| Recipient language | nothing (except re-reg: request `X-Lang`) | `users.preferred_language_id` |
| Review email listing title | nothing | the `Review`/product row's title — **HTML-escaped** before HTML interpolation (it is the only email that interpolates user-supplied content; raw in the plain-text fallback) |
| Verify-page success | the `oobCode` (Firebase-validated, cannot forge "verified") | `applyActionCode` resolving server-side |

All nine emails fire off server-authoritative state transitions. The verification email link is minted by the Admin SDK server-side.

## 11. Status ledger

Status syntax per conventions Part 1: `[ ]` not started · `[~]` in progress · `[x]` complete · `[!]` blocked. Updated as briefs ship.

### In scope (this feature)

**Foundation**
- [x] `EmailService.sendHtml(to, subject, htmlBody, textFallback)` + multipart helper
- [x] Branded HTML shell helper (logo from R2 URL) + plain-text fallback assembly
- [x] `BACKEND_TRANSLATIONS` seeds for all nine emails (subjects + bodies), four locales, inline-append

**Verification gate**
- [x] `FirebaseAuthFilter` reads live-token `emailVerified`; `EMAIL_NOT_VERIFIED` coded rejection; social logins exempt
- [x] `/auth/resend-verification` backend endpoint — evolved to **email-based, unauthenticated**, with a 60s gap + 4/day per-email throttle and no account-existence leak (see §3 / §4.1)

**Events + listeners (backend)**
- [x] `UserRegisteredEvent` + verification-email listener (Admin SDK link → Brevo)
- [x] `DeletionRequestedEvent` + listener
- [x] `DeletionCancelledEvent` + listener
- [x] `UserBannedEvent` + listener (`fallbackExecution` + `@Async`)
- [x] `UserUnbannedEvent` + listener (`fallbackExecution` + `@Async`)
- [x] `DeletionLockedEvent` + listener
- [x] `ReviewApprovedEvent` + listener
- [x] Daily 13:00 deletion-reminder cron (`reminder_sent_at`-guarded, direct send)
- [x] Re-registration-blocked send-once (flag on `banned_user_audit`; unban↔audit relationship verified in code)

**Schema**
- [x] `user_deletion_requests.reminder_sent_at` (V1 fold)
- [x] `banned_user_audit.reblock_notified_at` (V1 fold)

**Web**
- [x] `/[locale]/(portal)/(public)/verify` page (`applyActionCode` only, branded success/failure, no auto-login)
- [x] Register/login unverified → **sign-out + dismissable verify dialog** (the as-shipped model; no "check your email" keep-logged-in state, no `SessionGuard` verify branch — see §3 / §6)
- [x] Email-based Resend in the dialog + backend-driven countdown

**Mobile**
- [x] Sign-out + verify dialog on both sync paths (client-side gate, not a backend-rejection consumer — see §3 / §7)

**Router**
- [x] Confirmed clean — no change (audit, 2026-06-02)

**Added during build (not in the original nine)**
- [x] **Social-welcome email** — a plain welcome for social signups that *have* an email (social-no-email is an INFO-logged skip, no send). Added during Brief 4; noted here so the ledger matches reality.

**Removed during build**
- [x] The temporary smoke endpoint `POST /api/secure/admin/email/test` (`EmailTestAdminController`, Brief 1) was **removed** (Brief 8) once the real email paths became live callers of `EmailService`. Its scoped Part-7 error-contract deviation (the `{exception, message}` failure body) is gone with it — this surface is now strictly codes-only.

### Deferred (future Mastermind chats — do NOT block `shipped`)

- [ ] Product-blocked email — needs product moderation built first
- [ ] Password reset — full build, does not exist today
- [ ] Account-deleted (Day 7) confirmation — needs pre-delete email snapshot

### Notes carried from audit

- The throwaway test endpoint `POST /api/secure/admin/email/test` did not exist at spec-authoring time; it was then **created in Brief 1** as a temporary smoke harness (its own Javadoc declared it temporary) and **removed in Brief 8** once the real email listeners became the live callers of `EmailService`. See the "Removed during build" ledger line above.
- `getBackendTranslation` is `.orElseThrow()` — every new key must exist in all four locales.

---

## 12. Session log

- **2026-06-02 — backend `-1`/`-2`:** `EmailService.sendHtml` + multipart helper; branded `EmailLayout` shell + plain-text fallback; the temporary `EmailTestAdminController` smoke endpoint (later removed).
- **2026-06-03 — backend `-3`:** seeded 43 `email.*` keys × 4 locales into `BACKEND_TRANSLATIONS` (RU Latin-transliterated per Igor's mid-session PATCH; the original brief's Cyrillic was wrong for this repo).
- **2026-06-03 — backend `-4`/`-5`:** registration auto-send (verification + the new plain **social-welcome** email + seeds); email-logging standard.
- **2026-06-03 — backend `-6` (Brief 5b):** `/auth/resend-verification` made **email-based + unauthenticated** — 60s + 4/day per-email Redis throttle, no account-existence leak, coded responses incl. `retryAfterSeconds`.
- **2026-06-03 — backend `-7`/`-8`/`-9`/`-10`:** the `FirebaseAuthFilter` live-token gate + `EMAIL_NOT_VERIFIED`; the seven dedicated events + listeners; deletion-reminder cron (`reminder_sent_at`); re-registration-blocked send-once (`reblock_notified_at`); review-approved email with the HTML-escaped listing title; both V1 schema columns.
- **2026-06-03 — backend `-11` (Brief 8):** removed the temporary smoke endpoint; full suite **805 passed, 0 failures**.
- **2026-06-02/`-03` — web `-1`..`-3`:** evolved to the strict **sign-out** verify model — `/verify` runs `applyActionCode` only (no auto-login), `SessionGuard` verify branch reverted, dismissable verify dialog with the email-based Resend + backend-driven countdown.
- **2026-06-02/`-03` — expo `-1`/`-2`:** the client-side gate on **both** sync paths (`buildUserSession` + `initAuthListener`), sign-out + verify dialog via `AccountStateDialogsInit`, email-based Resend.
- **2026-06-03 — router:** confirmed clean by audit, no code change.
- **Native-translator review** completed across all four locales at close. **On-device smoke** of the mobile verify flow is owed by Igor on a real device (per the expo `-2` smoke checklist) but does not block `shipped`.
