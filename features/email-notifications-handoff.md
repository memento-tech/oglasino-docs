# Email Notifications — Handoff

A starting map for the next Mastermind, so they begin from reality rather than a cold read. The full spec is [`email-notifications.md`](email-notifications.md); this is the close-out summary plus the deferred work and the architecture worth carrying forward.

**Status:** `shipped` (backend + web + mobile live). On-device smoke of the mobile verify flow is owed by Igor but does not block `shipped`.

---

## What shipped

- **The nine in-scope emails** + a **social-welcome** email (a plain welcome for social signups that *have* an email; social-no-email is an INFO-logged skip), all branded HTML + plain-text fallback, all copy in `BACKEND_TRANSLATIONS` across four locales.
- **The verification flow — strict client-detection sign-out model.** An unverified email/password user never holds a session: the client reads the live Firebase token (`signInProvider === 'password' && !emailVerified`), signs out immediately, and shows a dismissable verify dialog. Verification happens only via the emailed link → web `/verify` (`applyActionCode`, no auto-login). Mobile enforces the same gate on **both** sync paths (`buildUserSession` + `initAuthListener`).
- **`EMAIL_NOT_VERIFIED`** remains the **server-side defense-in-depth gate** in `FirebaseAuthFilter` (reads the live token claim), but the client gates itself first, so it is not the client UX trigger.
- **Email-based, unauthenticated `/auth/resend-verification`** — 60s gap + 4/day per-email Redis throttle (both released on send failure), no account-existence leak, coded responses (`VERIFICATION_EMAIL_SENT` + `retryAfterSeconds`, `VERIFICATION_RESEND_COOLDOWN`, `VERIFICATION_RESEND_DAILY_LIMIT`, `EMAIL_SEND_FAILED`). Registration auto-send is exempt from the cap.
- **Event + listener architecture** — seven dedicated domain events (`UserRegisteredEvent`, `DeletionRequestedEvent`, `DeletionCancelledEvent`, `UserBannedEvent`, `UserUnbannedEvent`, `DeletionLockedEvent`, `ReviewApprovedEvent`) with `@TransactionalEventListener(AFTER_COMMIT)` listeners; plus the non-event deletion-reminder cron and the re-registration-blocked send-once path.
- **Two V1-fold schema columns** — `user_deletion_requests.reminder_sent_at`, `banned_user_audit.reblock_notified_at`.
- **An email-logging standard** established during build.
- Full backend suite **805 passed, 0 failures**. Native-translator review completed across all four locales.

---

## Deferred work — why, and what each needs

The most valuable part: three emails were deferred because they depend on things that don't exist yet. **Don't start the email — start the dependency.**

- **Product-blocked email** — blocked on **product moderation not existing.** `ModerationState.BANNED` is defined but **never set**; there is no admin moderation action, endpoint, or service. This is a moderation feature (the block transition + admin UI + service), with a `ProductBlockedEvent` + email listener downstream. Start the moderation feature, not the email.
- **Password reset** — **does not exist in web at all** (no `sendPasswordResetEmail`, no forgot-password UI, no route). A full build, not a reuse. If branded, it would follow the same Admin-SDK-link-via-Brevo + own-page pattern as verification.
- **Account-deleted (Day 7) confirmation** — not requested. Would need the recipient email **snapshotted before** the in-transaction hard-delete (`runHardDelete` currently snapshots only `firebaseUid` / `profileImageKey` / `wasBanned`).

---

## Architecture + conventions to carry forward

- **Events live in the email package; listeners in `listeners/`, senders in `service.impl`.** Use dedicated domain events, **never** the overloaded `UserStateChangedEvent`.
- **`fallbackExecution=true`** for transitions that run **outside** a transaction (ban / unban / lock / review-approve); AFTER_COMMIT-clean for the `TransactionTemplate` paths (deletion request / cancel / lock).
- **Verify transaction boundaries in code — do NOT trust the audit's tx claims.** The audit was wrong that registration runs in a request transaction; an explicit `TransactionTemplate` was required. `getOrCreateUser` / `firebaseSync` are **not** `@Transactional`.
- **All translations are Latin-typed, including RU** (phonetic transliteration, soft-sign as `'` → `''` when escaped). This bit us once (a brief shipped Cyrillic RU and had to be patched). Web UI copy is **backend-seeded** via `/public/translations`, not local message files. `getBackendTranslation` is `.orElseThrow()` — every key must exist in all four locales.
- **The verification gate reads the live Firebase token claim, never the stale DB column.** Verification status is authoritative only from Firebase. `emailVerifiedExternal` is stale-by-design (see open issues).
- **Reusable building blocks for any future email:** `EmailService.sendHtml` + `EmailLayout` + `EmailMasking` + `WebLocales` + `TransactionalEmailSender`. The review email has a dedicated sender because it is the only one interpolating user-supplied content (the listing title, **HTML-escaped**) and targets `/owner/reviews`.

---

## Open issues + cleanups

**Open issues** (see [`../issues.md`](../issues.md), both 2026-06-03):
- `registeredWithProvider` stored as the firebase-claim Map's `toString()` (medium) — feeds `AuthUserDTO.providerId`; the email feature reads the claim, not the column, so it is unaffected.
- `emailVerifiedExternal` stale-by-design (low/medium) — must not be read as authoritative anywhere.

**Cleanups** (non-blocking — see spec §9):
- Two dual-key translation redundancies (`verify.email.resend.dailylimit` / `email.verification.daily_limit`; `verify.email.resend.failed` / `email.send_failed`) — drop the unused twin once the rendered key per surface is confirmed.
- Inert mobile `account-verification.tsx` stub + its dead `UserMenu` "verify account" entry (an unverified user is never logged in) → Ω teardown candidate, alongside the orphaned Facebook scaffolding.
- Optional: relocate the email classes into a dedicated `email` package.

**Deliverability note:** the first test email junked (SPF didn't include Brevo); after the SPF fix it inboxes. Now a **monitor cold-domain reputation** item, not a launch blocker.
