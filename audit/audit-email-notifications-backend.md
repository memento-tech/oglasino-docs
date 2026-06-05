# Audit — email-notifications (oglasino-backend)

**Repo:** oglasino-backend
**Branch:** dev (current checkout; read-only audit, no changes)
**Date:** 2026-06-02
**Phase:** 2 (audit, read-only)
**Method:** code read as ground truth. Every CRITICAL claim and every "does not exist"
was verified by reading the actual file and/or `grep`, not inferred. No prior email
reports or specs were read.

---

## TL;DR for Mastermind

- **Email infra is freshly scaffolded but inert.** `EmailService` exists with a single
  `sendPlainText(to, subject, body)` method, Brevo SMTP is wired in all three envs, and
  `EmailProperties` carries the envelope. But the service has **zero callers in main code**
  and is **plain-text only** (`multipart=false`, `setText(body, false)`). Branded HTML +
  plain-text fallback **needs a new method** — the interface Javadoc says so explicitly.
- **The brief's premise about the test endpoint is wrong.** There is **no**
  `POST /api/secure/admin/email/test` endpoint (and no email endpoint at all). See
  "Brief vs reality" below. Nothing to "not delete."
- **There is no email-verification login gate today (option (c) — nothing).** Verified.
  Firebase's `emailVerified` is captured into the DB at registration from the authoritative
  token, but nothing reads it to block login or any secure endpoint. The feature would have
  to **build** the gate, not inherit it. Trust boundary is clean (no client-supplied value).
- **The event architecture the brief wants to validate is HALF true.** The
  `UserStateChangedEvent` + `@TransactionalEventListener(AFTER_COMMIT)` pattern exists and is
  the right precedent. But the two ban/unban transitions run **outside any Spring transaction**
  (the listener only fires via `fallbackExecution=true`), so AFTER_COMMIT gives no real commit
  boundary there. And **3 of the 10 transitions do not exist in code at all** (deletion
  reminder, deletion postponed by admin, product blocked). Details in Section 4 — this is the
  single biggest design driver.
- **BACKEND_TRANSLATIONS is a solved, proven mechanism for cron/no-request sends.** Recipient
  language lives on `users.preferred_language_id`; `getBackendTranslation(key, langCode)` takes
  the code explicitly, no request context required. Existing push-notification code is a direct
  precedent.

---

## Brief vs reality

The brief instructs (Section 1, last bullet): *"Confirm the throwaway endpoint
`POST /api/secure/admin/email/test` exists. (Do not delete — that's a later brief.)"*

- **Brief says:** the endpoint exists.
- **Code says:** it does **not** exist. `grep -rn "admin/email\|email/test" src/` → no matches.
  No controller maps any email route (`grep` of controllers for "email" → none). `EmailService`
  is referenced only by its own interface + impl file in `src/main` — **no callers anywhere in
  main code** (`grep -rn "EmailService\|sendPlainText" src/main/java` returns only the
  interface, the impl, and the impl's import). The only consumer is `DefaultEmailServiceTest`.
- **Why this matters:** low/medium. It does not block the audit — the inventory is the point —
  but the spec/brief author is modelling an endpoint that was never built. Whatever later brief
  was meant to "delete the throwaway endpoint" has nothing to delete. If a manual send-path
  smoke test is wanted before the listeners are built, that endpoint still needs to be written.
- **Recommended resolution:** treat the send path as "library wired, never invoked." Either
  (a) the later brief adds the temporary test endpoint AND removes it, or (b) the first listener
  brief is the first real caller and the test endpoint is skipped. Mastermind's call.

This is reported, not worked around — the audit proceeds on the real (no-endpoint) state.

---

## Section 1 — EmailService + Brevo

### 1.1 Method surface (CONFIRMED — handoff was right that it's `sendPlainText` only)

**Interface** — `src/main/java/com/memento/tech/oglasino/service/EmailService.java`

```java
// EmailService.java:12-22
public interface EmailService {
  /** ...envelope (From, From-name, Reply-To) comes from EmailProperties and is never
      client-supplied. @throws MailException if SMTP dispatch fails */
  void sendPlainText(String to, String subject, String body);
}
```

The class Javadoc is explicit about scope (lines 3-11):

> "This brief ships a single method — `sendPlainText(String, String, String)`. **HTML bodies,
> multi-part messages, and template rendering are explicitly out of scope and will land in the
> per-feature briefs that need them.**"

So the codebase itself flags that branded HTML needs a future method. The surface is exactly
one method on both interface and impl — no `sendHtml`, no `sendMultipart`, no template variant.

**Impl** — `src/main/java/com/memento/tech/oglasino/service/impl/DefaultEmailService.java`
(full file, 37 lines):

```java
// DefaultEmailService.java:21-36
@Override
public void sendPlainText(String to, String subject, String body) {
  MimeMessage message = mailSender.createMimeMessage();
  try {
    MimeMessageHelper helper =
        new MimeMessageHelper(message, false, StandardCharsets.UTF_8.name());   // multipart = FALSE
    helper.setFrom(emailProperties.getFromAddress(), emailProperties.getFromName());
    helper.setReplyTo(emailProperties.getReplyTo());
    helper.setTo(to);
    helper.setSubject(subject);
    helper.setText(body, false);                                                // html = FALSE
  } catch (MessagingException | UnsupportedEncodingException ex) {
    throw new MailPreparationException("Failed to build mime message", ex);
  }
  mailSender.send(message);
}
```

Injected deps: `JavaMailSender mailSender`, `EmailProperties emailProperties`.

### 1.2 Envelope (From / From-name / Reply-To)

Set from `EmailProperties` — `src/main/java/com/memento/tech/oglasino/properties/EmailProperties.java`:

```java
@Configuration
@ConfigurationProperties(prefix = "oglasino.email")
public class EmailProperties {
  private String fromAddress;   // oglasino.email.from-address
  private String fromName;      // oglasino.email.from-name
  private String replyTo;       // oglasino.email.reply-to
  // standard getters/setters
}
```

The envelope is server-controlled, never client-supplied (good — trust boundary clean for the
sender identity).

**Per-environment values — all three envs inject from env vars; the concrete addresses are NOT
literal in the config files.** Verified in `application-{dev,stage,prod}.yaml`:

```yaml
oglasino:
  email:
    from-address: ${OGLASINO_EMAIL_FROM_ADDRESS}
    from-name:    ${OGLASINO_EMAIL_FROM_NAME}
    reply-to:     ${OGLASINO_EMAIL_REPLY_TO}
```

- dev: `application-dev.yaml:285-287`
- stage: `application-stage.yaml:311-313`
- prod: `application-prod.yaml:297-299`

The concrete sender values live only in **YAML comments** (documentation) and the secret store,
not in the config keys. Per the comments next to the `spring.mail` block:
- dev + stage verified sender: `noreply-stage@oglasino.com`
- prod verified sender: `noreply@oglasino.com`
- (reply-to is env-injected; the value is not asserted in any comment I read — treat as
  unknown-from-code, supplied via `OGLASINO_EMAIL_REPLY_TO`).

### 1.3 Brevo SMTP transport

Standard Spring `spring.mail` config, identical shape in all three envs (env-injected creds):

```yaml
# Outbound transactional email via Brevo SMTP relay (smtp-relay.brevo.com:587 STARTTLS).
spring:
  mail:
    host: ${OGLASINO_EMAIL_SMTP_HOST}
    port: ${OGLASINO_EMAIL_SMTP_PORT}
    username: ${OGLASINO_EMAIL_SMTP_USERNAME}
    password: ${OGLASINO_EMAIL_SMTP_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls: { enable: true, required: true }
```

- dev block: `application-dev.yaml:57-72` (comment: dev shares the **stage** Brevo SMTP key;
  no separate dev key; free-tier 300/day shared with stage).
- stage block: `application-stage.yaml:75-89` (Brevo key `oglasino-backend-stage`).
- prod block: `application-prod.yaml:72-86` (Brevo key `oglasino-backend-prod`).

`JavaMailSender` is Spring Boot's auto-configured bean from this config. No custom Brevo API
client — it's plain SMTP relay.

### 1.4 Can the current send path produce HTML + plain-text fallback (multipart)?

**No.** Verified line-by-line:
- `MimeMessageHelper(message, false, ...)` → `multipart=false`, so no alternative parts can be
  attached. HTML is **not reachable** through this method.
- `setText(body, false)` → second arg `false` = plain text, not HTML.

To send branded HTML with a plain-text fallback, **a new method is required** (e.g.
`sendHtml(to, subject, htmlBody, textFallback)` constructing `MimeMessageHelper(message, true, ...)`
and calling the two-arg `setText(plain, html)`). The existing method should stay for any
plain-only sends. This matches the interface Javadoc's stated intent.

### 1.5 Admin test endpoint

**Does not exist.** See "Brief vs reality" above. No `POST /api/secure/admin/email/test`,
no email controller, no caller of `EmailService` in main code.

---

## Section 2 — Firebase Admin SDK

### 2.1 Initialization + credentials/scope

`src/main/java/com/memento/tech/oglasino/security/config/FirebaseConfig.java` —
`@PostConstruct init()`:

```java
FirebaseOptions options =
    FirebaseOptions.builder()
        .setCredentials(GoogleCredentials.fromStream(credentialsStream))
        .build();
FirebaseApp.initializeApp(options);
```

- Credential is a **service-account JSON** loaded from a path in config property
  `app.firebase.credentials-path` (env override `FIREBASE_CREDENTIALS_PATH`).
  - dev default: `./secrets-local/firebase-service-account.json`
  - stage/prod: `/run/secrets/firebase.json` (Docker secret).
- `openCredentials()` validates existence + readability and throws a clear
  `IllegalStateException` otherwise.
- Scope: a service-account credential grants the Admin SDK full project-admin authority
  (no narrower scoping applied at init). This is the standard Admin SDK posture.
- SDK version: firebase-admin 9.9.0 (pom.xml).
- **No explicit `FirebaseAuth` bean.** Callers use the singleton `FirebaseAuth.getInstance()`,
  which resolves against the app initialized here.

> SECURITY context (not new, already tracked): the Firebase key is committed in-repo for dev —
> see auto-memory `project_firebase_key_in_git` / `project_secret_rotation_deferred`. Not in
> scope for this feature, flagged only so the email feature does not assume a clean key story.

### 2.2 `generateEmailVerificationLink` / `generatePasswordResetLink` reachability

**Neither is used or imported anywhere.** Verified:
`grep -r "generateEmailVerificationLink\|generatePasswordResetLink\|ActionCodeSettings\|EmailVerification\|PasswordReset"` over `*.java` → **zero matches**.

- They **are** reachable in principle: the Admin SDK is initialized and `FirebaseAuth.getInstance()`
  is already used elsewhere (see below), and both methods live on
  `com.google.firebase.auth.FirebaseAuth`. So calling
  `FirebaseAuth.getInstance().generateEmailVerificationLink(email, actionCodeSettings)` /
  `.generatePasswordResetLink(email, actionCodeSettings)` is available with no new wiring beyond
  building an `ActionCodeSettings` (which carries the continue-URL → the web `/verify` target,
  see Section 6).
- No existing caller. If the feature generates verification links server-side, it is the first
  consumer.

Existing `FirebaseAuth.getInstance()` callers (proof the SDK is live):
- `DefaultFirebaseAuthService.java:40` — `verifyIdToken`
- `DefaultFirebaseUserService.java` — `updateUser` (disable :23 / enable :35), `deleteUser` :44,
  `getUser` :57, `revokeRefreshTokens` :70.

### 2.3 Registration path end-to-end (where the Firebase user exists vs. where the DB row is made)

**Important nuance: the backend does NOT create the Firebase user.** The Firebase account is
created client-side (web/mobile Firebase SDK). The backend only mirrors it into Postgres on
first authenticated sync. Trace:

1. **Controller** — `AuthController.firebaseSync()` (`controller/AuthController.java:52-117`),
   `POST /api/auth/firebase-sync`:
   - resolves + verifies the ID token (`verifyToken`),
   - **email-ban check**: `if (userAuditService.isEmailBanned(email)) { deleteFirebaseUser(uid);
     return 403 EMAIL_BANNED }` (lines 65-77),
   - `SyncResult syncResult = firebaseAuthService.getOrCreateUser(request, servletRequest)` (:79),
   - maps to `AuthUserDTO`, sets `wasRegister` from the server-derived `SyncResult` bit.

2. **Service** — `DefaultFirebaseAuthService.getOrCreateUser()` (`security/service/impl/...:72-97`):
   - `getUser(token)` (existing) `.orElseGet(() -> createUserSynchronized(...))`.

3. **Creation** — `createUserSynchronized()` (`...:99-139`), synchronized per Firebase UID:
   ```java
   User newUser = new User();
   newUser.setFirebaseUid(token.getUid());
   newUser.setEmail(token.getEmail());
   newUser.setEmailVerifiedExternal(token.isEmailVerified());   // <-- authoritative token value
   newUser.setSubscriptionType(SUBSCRIPTION_FREE);
   newUser.setSubscriptionActive(true);
   newUser.setPreferredLanguage(entityManager.getReference(Language.class,
       languageContext.getCurrentLanguageId()));               // <-- language captured at register
   ... role assignment ...
   return new SyncResult(userService.saveUser(newUser), true);  // wasRegister=true
   ```

**Registration trigger point for a verification email:** the cleanest hook is the
**`wasRegister==true`** branch — either in `AuthController.firebaseSync` after the sync, or by
publishing a new domain event from there / from `createUserSynchronized` (see Section 4.1). This
is the only place the backend knows "a brand-new account was just minted." Note this method does
**not** publish any event today.

---

## Section 3 — The login / verification gate (CRITICAL)

### Verdict: **(c) — nothing.** There is no server-side email-verification gate, and Firebase's native sign-in is not blocking unverified users here either (the backend never checks).

Verified by reading `FirebaseAuthFilter` in full and the login controller, plus grepping every
`emailVerified` reference.

### 3.1 What the auth filter enforces (and what it doesn't)

`security/filter/FirebaseAuthFilter.java` (full read). On every request it:
1. verifies the token (`verifyToken`),
2. loads cached `AuthenticatedUserDTO` from Redis,
3. **banned short-circuit**: `if (authData.disabled()) → 403 USER_BANNED` (:83-86),
4. **pending-deletion short-circuit**: clears context, continues anonymously (:93-97),
5. builds `OglasinoAuthentication` and sets it in the context.

There is **no** `emailVerified` check anywhere in the filter. The only blocking conditions are
`disabled` (ban) and `PENDING_DELETION`.

### 3.2 Where `emailVerified` is read — and its trust source (trust boundary: CLEAN)

`grep` of `emailVerified|email_verified|isEmailVerified` across `src/main`:

- **Write (authoritative):** `DefaultFirebaseAuthService.createUserSynchronized:121` —
  `newUser.setEmailVerifiedExternal(token.isEmailVerified())`. Source is **(a) the verified
  Firebase ID token** (`FirebaseToken.isEmailVerified()`), set once at account creation. **Not**
  client-supplied. ✔ Part 11 compliant.
- **Field:** `entity/User.java` carries `boolean emailVerifiedExternal` and a separate
  `boolean verifiedInternally`.
- **Read (display only):** `converter/EntityUserInfoConverter.java:59` —
  `destination.setVerified(source.isVerifiedInternally() || source.isEmailVerifiedExternal())`,
  feeding `UserInfoDTO.verified` (a profile "verified" badge). This is **not** a gate; it's
  surfaced for display.
- **Test-only client input:** `data/user/.../TestUsersImportService` + `ImportUserData` accept
  `emailVerifiedExternal` from an import DTO — dev/test import path only, not a production trust
  surface.

**Critical observations for the feature:**
- `emailVerifiedExternal` is captured **only at registration** from the token, and **never
  refreshed**. A user who registers unverified (token `email_verified=false`) and later verifies
  out-of-band will keep `emailVerifiedExternal=false` in Postgres unless something updates it.
  Whatever builds the gate must decide whether to read the **live token claim** on each sign-in
  (authoritative, always current) or the **stored DB column** (stale after first login). The
  token claim is the safer source.
- `emailVerified` is **not** carried on `OglasinoAuthentication` or the cached
  `AuthenticatedUserDTO`. If the gate is to be enforced in the filter from cached data, the DTO/
  cache projection (`UserRepository.findAuthDataByFirebaseUid`) would need the field added — OR
  the gate reads `decoded.isEmailVerified()` off the already-verified `FirebaseToken` the filter
  holds (no cache change needed, and always live). The latter is cleaner and avoids a stale-cache
  trap.

### 3.3 What this means for the feature

The feature **builds the gate, does not inherit it.** Today an unverified user can sign in,
create products, and hit every `/api/secure/**` endpoint. If "must verify before login/acting"
is a requirement, that's net-new enforcement (filter check on `decoded.isEmailVerified()`, or a
check in `firebaseSync`), plus a product decision about provider accounts (Google sign-in tokens
arrive `email_verified=true`, so a blanket gate only bites email/password registrations — which
is presumably the intent).

---

## Section 4 — Event architecture per transition (CRITICAL — drives the whole design)

### 4.0 Existing event infrastructure (the precedent the brief names — CONFIRMED, with a caveat)

Events (`src/main/java/com/memento/tech/oglasino/events/`):
- `UserStateChangedEvent(Long userId)`
- `ProductUpdateEvent(Long productId)`
- `ProductStateUpdateEvent(Long productId, ProductState productState)`
- `ProductImagesRemovedEvent(Set<String> imageKeys)`

Listeners (`.../listeners/`):
- `UserStateChangedEventListener` —
  `@TransactionalEventListener(phase = AFTER_COMMIT, fallbackExecution = true)` →
  `webRevalidationService.revalidateUserCache(userId)`.
- `ProductIndexerEventListener` — `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` +
  `@Retryable` → Elasticsearch reindex.
- `ProductImagesRemovedEventListener` — `@TransactionalEventListener(AFTER_COMMIT)` + `@Async`
  + `@Retryable` → R2 image cleanup.

So the **brief's architectural direction is the right precedent**: domain events published by
state transitions, consumed by `@TransactionalEventListener(AFTER_COMMIT)` listeners living in a
feature package. An email feature package subscribing to these domain events is consistent with
how indexing and image-cleanup already work.

**The caveat (CRITICAL):** `AFTER_COMMIT` only does something useful if the publish happens
inside a Spring-managed transaction. Two of the user transitions publish from a facade method
that has **no** `@Transactional` (ban/unban — see 4.5/4.6). There, `UserStateChangedEventListener`
fires only because it sets `fallbackExecution=true`, which runs the listener **synchronously,
with no commit boundary**. An email listener that genuinely needs "send only after the row is
durably committed" cannot rely on AFTER_COMMIT semantics on those paths — it would either inherit
`fallbackExecution=true` (fire even without a tx, i.e. no rollback safety) or the transitions
would need to be wrapped in a transaction first. This is the main thing the design must resolve.

### 4.1 Email registration (verification email trigger)
- **(i) Where:** `DefaultFirebaseAuthService.createUserSynchronized` (`...:99-139`); the
  "new account" signal is `SyncResult.wasRegister==true`, surfaced up to
  `AuthController.firebaseSync` (`controller/AuthController.java:79-..`).
- **(ii) Event today:** **none.** No event published on user creation.
- **(iii) Tx:** `getOrCreateUser` runs in a transactional context (`getUser` is `@Transactional`;
  `saveUser` commits). The controller path is request-driven, so a commit boundary is available.
- **(iv) Cleanest event to add:** a `UserRegisteredEvent(userId)` published either from
  `createUserSynchronized` (right where `wasRegister=true` is decided) or from the controller in
  the `wasRegister` branch. Touches only the commit path. **Caveat:** Firebase email-verification
  links are normally generated/sent by the client SDK at sign-up; confirm with web/mobile whether
  the backend is meant to own the verification email at all (Section 6) before adding this event.

### 4.2 Account deletion requested — Day 0
- **(i) Where:** `DefaultUserDeletionService.requestDeletion` (`service/impl/...:98-174`).
- **(ii) Event today:** **YES** — `UserStateChangedEvent(userId)` (:170), published **inside**
  the `writeTx.executeWithoutResult(...)` block.
- **(iii) Tx:** transactional via `TransactionTemplate` (`writeTx`), so AFTER_COMMIT works
  properly here.
- **(iv):** Reuse `UserStateChangedEvent`, OR add a dedicated `DeletionRequestedEvent(userId)` if
  the email listener needs to distinguish "deletion requested" from the other state changes that
  also raise `UserStateChangedEvent` (ban, unban, restore, hard-delete all reuse it — see note at
  end of section). A dedicated event is cleaner for routing to the right email template.

### 4.3 Deletion reminder — day before hard-delete (CRON-driven)
- **(i) Where:** **does not exist.** The hard-delete cron is
  `jobs/UserDeletionScheduledJobs.processScheduledDeletions` (`@Scheduled cron`, :51) which sweeps
  `user_deletion_requests` rows whose `scheduled_deletion_at <= now` and calls
  `userDeletionService.executeScheduledDeletion`. There is **no** reminder job — `grep` for
  `reminder` → nothing.
- **(ii) Event today:** N/A (no reminder code).
- **(iii) Tx:** cron-driven, **no request transaction**. A reminder send happens in the job loop,
  not in a request.
- **(iv) Cleanest design:** a **new** `@Scheduled` job (or an extra query in the existing one)
  that selects `user_deletion_requests` with status PENDING and `scheduled_deletion_at` within the
  next ~24h (and not already reminded — needs a `reminder_sent_at`/flag column to avoid
  re-sending daily). The data model supports it (`scheduled_deletion_at` exists). Because there's
  no transaction-commit-then-send pattern here, the email send is just a direct
  `emailService` call inside the job's per-row try/catch (mirroring how `processScheduledDeletions`
  isolates per-row failures). Events/AFTER_COMMIT do not apply to cron sends.

### 4.4 Deletion cancelled / undeleted (sign back in during grace)
- **(i) Where:** `DefaultUserDeletionService.cancelDeletionOnLogin` (`service/impl/...:181-230`),
  called from `AuthController.firebaseSync` (:99).
- **(ii) Event today:** **YES** — `UserStateChangedEvent(userId)` (:228), inside the
  `writeTx.executeWithoutResult` block.
- **(iii) Tx:** transactional via `TransactionTemplate`; AFTER_COMMIT works.
- **(iv):** reuse, or dedicated `DeletionCancelledEvent(userId)` for template routing.

### 4.5 User banned
- **(i) Where:** `admin/facade/impl/DefaultUsersFacade.disableUser` (`...:66-100`).
- **(ii) Event today:** **YES** — `UserStateChangedEvent(userId)` (:82).
- **(iii) Tx:** **NO `@Transactional`.** The facade method is a sequence of independent
  `saveUser` (which self-commits + evicts caches), audit insert, Firebase disable, and per-product
  state flips. The in-code comment (:80-81) is explicit: *"Listener uses fallbackExecution=true
  because this facade method runs outside a wrapping Spring tx."* → **AFTER_COMMIT gives no real
  commit boundary here.**
- **(iv):** if an email is wanted on ban, the listener must accept `fallbackExecution=true`
  semantics (fires synchronously, no rollback guard) OR the ban path is wrapped in a transaction
  first (larger change, touches the carefully-ordered saveUser→audit→Firebase sequence the comment
  describes — do not do this casually). Dedicated `UserBannedEvent(userId)` recommended for
  template routing; same fallback caveat applies.

### 4.6 User restored / unbanned
- **(i) Where:** `admin/facade/impl/DefaultUsersFacade.enableUser` (`...:102-143`).
- **(ii) Event today:** **YES** — `UserStateChangedEvent(userId)` (:117).
- **(iii) Tx:** **NO `@Transactional`** (same shape as ban; `fallbackExecution` reliance).
- **(iv):** same caveat as 4.5; dedicated `UserUnbannedEvent(userId)` recommended.

### 4.7 Deletion postponed by admin
- **(i) Where:** **no "postpone" exists.** The nearest mechanism is an admin **lock**:
  `UserDeletionLock` rows + `lockFromDeletion` in `DefaultUserDeletionService`, checked by the cron
  (`UserDeletionScheduledJobs.processScheduledDeletions:74` —
  `deletionLockRepository.existsActiveLockForUser(...)` skips locked users). That prevents
  hard-delete; it does not "postpone" a scheduled date. There is **no** extend/postpone method
  (`grep postpone|extendDeletion` → nothing).
- **(ii) Event today:** none (locking publishes no event — confirm with Mastermind whether
  "postpone" maps to "lock" semantically).
- **(iii) Tx:** lock creation is request-driven (admin action).
- **(iv):** if "postponed" == "admin locked from deletion," a `DeletionPostponedEvent(userId)` (or
  reuse of a lock event) would be added at the lock site. If "postponed" means moving
  `scheduled_deletion_at` forward, that capability **does not exist** and is net-new feature work.
  **Needs a product definition before any event is added.**

### 4.8 Re-registration blocked (banned user re-registers within 12 months)
- **(i) Where:** `AuthController.firebaseSync` (:65-77) →
  `userAuditService.isEmailBanned(email)` (`DefaultUserAuditService:37-41`, queries
  `banned_user_audit` by email-hash where `retention_until > now`). On hit: delete the just-created
  Firebase user and return `403 EMAIL_BANNED` (translationKey `email.banned`).
- **(ii) Event today:** **none** — the request is rejected with 403 before any user row exists.
- **(iii) Tx:** the reject happens **before** `getOrCreateUser`; there is no user entity and no
  transaction around the rejection. Email-banned check is a read.
- **(iv):** if an email is wanted here ("your re-registration was blocked"), note there is **no
  local user, no userId, and no preferred-language record** — the only identity is the email + the
  request's `X-Lang` header (the user isn't authenticated as an Oglasino user). An email send here
  cannot use the recipient-language-from-DB mechanism (Section 5); it must fall back to the request
  language or a default. Cleanest is a direct `emailService` call in the controller's reject
  branch (no event — nothing commits), if this email is desired at all. **Flag:** sending mail to
  a banned email on every re-reg attempt is an abuse/spam vector — rate-limit or skip per the
  rate-limiting discipline (auto-memory `feedback_rate_limiting_scope`).

### 4.9 Product blocked
- **(i) Where:** **does not exist.** `ModerationState.BANNED` is defined in the enum but is
  **never set** anywhere. `grep` shows `setModerationState` only at
  `DefaultProductService:107` (always `APPROVED` on create) and in the test importer. There is no
  admin product-moderation controller/service that blocks/bans a product. The only product-state
  machine in use is `ProductState` (ACTIVE/INACTIVE/…) via `changeProductStateAsSystem`
  (used by the ban flow to hide a banned user's products), which is a different axis from
  moderation.
- **(ii) Event today:** product *state* changes publish `ProductStateUpdateEvent` (for ES
  reindex), but there is **no moderation-block transition** to publish from.
- **(iii) Tx:** N/A (transition doesn't exist).
- **(iv):** "product blocked" is **net-new feature work** — it needs an admin endpoint/service
  that sets `ModerationState.BANNED` (transactionally), and only then can it publish a
  `ProductBlockedEvent(productId, ownerId)` for an email listener. The owner's language is reachable
  via the product → owner → `preferred_language` (Section 5). **Needs the moderation transition
  built first; the email listener is downstream of that.**

### 4.10 Review received (a user receives a new review)
- **(i) Where:** review creation: `DefaultReviewService.reviewProduct`
  (`service/impl/...:39-72`) — saves the `Review` (reviewer, targetUser, targetProduct). Approval/
  notification happens later in `admin/service/impl/DefaultAdminReviewService` (admin approves a
  review), which already sends **push notifications** (`notif.review.new.*`) to the target user via
  `getBackendTranslation`.
- **(ii) Event today:** **none** (no ApplicationEvent on review create or approve). The existing
  notification on approval is a direct synchronous `notificationsService.sendAsyncNotification`
  call, not an event/listener.
- **(iii) Tx:** `reviewProduct` is not annotated `@Transactional` directly (relies on
  repository-level tx); `DefaultAdminReviewService` approval is request-driven.
- **(iv):** decide the trigger — "review received" likely means **on approval** (the moment the
  target user is told), matching where the push notification already fires. The cleanest seam is to
  publish a `ReviewApprovedEvent(reviewId)` from `DefaultAdminReviewService` at the same point it
  sends the push, and have the email listener mirror the push. The target user's language is
  already resolved there (`review.getTargetUser()....getPreferredLanguage().getCode()`). Whether
  this needs to be event-driven vs. a direct email call alongside the existing push is a
  consistency call — the rest of the codebase already does the push **directly** (not via event),
  so a direct email call there would match the surrounding style (Part 4a "match the surrounding
  code's style"); an event would match the brief's stated direction. **Mastermind decides.**

### Summary table

| # | Transition | Exists? | Where | Event today | Tx / Cron | AFTER_COMMIT usable? |
|---|---|---|---|---|---|---|
| 1 | Email registration | yes | `DefaultFirebaseAuthService.createUserSynchronized` | none | request tx | yes (add event) |
| 2 | Deletion requested D0 | yes | `DefaultUserDeletionService.requestDeletion:170` | `UserStateChangedEvent` | TransactionTemplate | **yes** |
| 3 | Deletion reminder | **NO** | — (would be a new cron near `UserDeletionScheduledJobs`) | n/a | cron | n/a (direct send) |
| 4 | Deletion cancelled | yes | `DefaultUserDeletionService.cancelDeletionOnLogin:228` | `UserStateChangedEvent` | TransactionTemplate | **yes** |
| 5 | User banned | yes | `DefaultUsersFacade.disableUser:82` | `UserStateChangedEvent` | **no @Transactional** | **NO** (fallbackExecution) |
| 6 | User unbanned | yes | `DefaultUsersFacade.enableUser:117` | `UserStateChangedEvent` | **no @Transactional** | **NO** (fallbackExecution) |
| 7 | Deletion postponed by admin | **NO** (only "lock") | `lockFromDeletion` + `UserDeletionLock` | none | request | needs definition |
| 8 | Re-registration blocked | yes | `AuthController.firebaseSync:65-77` | none (403) | pre-tx reject | n/a (no user/lang) |
| 9 | Product blocked | **NO** | — (`ModerationState.BANNED` never set) | none | n/a | needs the transition built first |
| 10 | Review received | yes (create) / push on approve | `DefaultReviewService.reviewProduct` / `DefaultAdminReviewService` | none (push is direct) | request | add event or direct send |

**Cross-cutting flag:** `UserStateChangedEvent` is **overloaded** — it is published by deletion-
requested, deletion-cancelled, ban, unban, AND hard-delete (`DefaultUserDeletionService.runHardDelete:408`,
inside its `writeTx`). A single email listener on this event cannot tell which transition fired
without re-reading state. For per-transition email templates, **introduce dedicated events** (one
per emailed transition) rather than overloading `UserStateChangedEvent`. The hard-delete path
itself (transition not in the brief's ten, but adjacent) deletes the `users` row inside the tx and
only snapshots `firebaseUid`/`profileImageKey`/`wasBanned` — **not the email** — so any "account
deleted" confirmation email would need the recipient email captured into the snapshot **before**
the in-tx delete.

---

## Section 5 — BACKEND_TRANSLATIONS

### 5.1 Read path
`TranslationService.getBackendTranslation(String code, String langCode)`
(`service/TranslationService.java:18`), impl
`DefaultTranslationService.getBackendTranslation` (`service/impl/...:106-110`):

```java
return Optional.ofNullable(backendTranslations.get(code))
    .map(m -> m.get(langCode))
    .orElseThrow();
```

Locale is passed **explicitly** as a `langCode` string (e.g. `"en"`, `"sr"`, `"ru"`, `"cnr"`).
No request context required. (Note: `.orElseThrow()` — a missing key or missing locale throws
`NoSuchElementException`; the email feature must seed all four locales for every key or guard the
call.)

### 5.2 In-memory map / warmup
`DefaultTranslationService.backendTranslations` (`:28`):
`volatile Map<String, Map<String,String>>` keyed **key → langCode → value**. Loaded in
`@PostConstruct initBackendTranslations()` (:42-45) via `indexBackendTranslations()` (:135-146),
which reads `translationRepository.findByTranslationNamespace(BACKEND_TRANSLATIONS)` and groups by
key then language code. Rebuilt on BACKEND_TRANSLATIONS updates. So at runtime backend translations
are an in-memory lookup, not a DB hit per send.

### 5.3 Per-locale resolution with NO request context (cron / backend-originated send)
**Solved pattern, proven in production code.** The recipient's language is read from the **User
entity** and passed explicitly:

- `entity/User.java:77-79` — `@ManyToOne Language preferredLanguage` (`preferred_language_id`,
  non-null).
- Precedent 1 — `DefaultFavoriteProductFacade.notifyUserOnSavedProduct` (:96-120): takes a
  `LanguageDTO preferredLanguage`, calls
  `getBackendTranslation("notif.product.saved.title", preferredLanguage.code())`.
- Precedent 2 — `DefaultAdminReviewService.sendReviewerSuccessNotification` (:83-108):
  `var lang = review.getReviewer().getPreferredLanguage().getCode();` then
  `getBackendTranslation("notif.review.check.success.title", lang)`.

So for a cron email (e.g. deletion reminder), the job loads the `User` (or just its
`preferred_language.code`) and passes the code into `getBackendTranslation`. **No `X-Lang`, no
`LocaleContextHolder` needed.** For request-scoped sends, `LanguageContext` (@RequestScope, set by
`CurrentLanguageFilter` from the `X-Lang` header / `OglasinoAuthentication.preferredLanguage`) is
available but not required.

`%s`-style placeholders are used in existing rows and filled with Java `String.formatted(...)`
(see review/follow notifications), so dynamic values (names, dates) in email copy follow the same
convention.

### 5.4 Seeding
SQL seeds in `src/main/resources/data/translations/`:
`0001-data-web-translations-{EN,RS,CNR,RU}.sql`. One INSERT per locale file:
`oglasino_translation (id, language_id, translation_namespace, translation_key, translation_value, created_at)`.
Existing BACKEND_TRANSLATIONS rows live in the `2100`-block (EN file lines ~2-17), e.g.:
```sql
(2104, 3, 'BACKEND_TRANSLATIONS', 'notif.review.new.title', 'New review', CURRENT_TIMESTAMP),
(2108, 3, 'BACKEND_TRANSLATIONS', 'notif.product.saved.title', 'Someone added your product to favorites ❤️', CURRENT_TIMESTAMP),
```
Per conventions Part 6 Rule 3 + auto-memory `feedback_translation_seed_inline_append`: email keys
**inline-append** to the BACKEND_TRANSLATIONS group at the next free ID in each of the four files
(the dedicated-file exception is multi-namespace only — not this). `language_id` differs per file
(EN uses `3` in the example; verify each file's id at seed time). Note `me/cnr` aliases to `sr`
per Part 9 — but the CNR seed file exists and is populated separately, so seed all four.

### 5.5 Precedent for backend-originated localized text
Push notifications already do exactly the "backend emits localized text, recipient language from
DB" thing the email feature needs: `DefaultFavoriteProductFacade`, `DefaultAdminReviewService`,
and the follow/saved/review notification keys. The email feature is a parallel sink to the same
`getBackendTranslation(key, recipientLangCode)` mechanism — no new translation infrastructure
required, only new keys + (for HTML) a body-assembly step that the translation layer doesn't cover.

---

## Section 6 — Seams (assumptions about other repos / Firestore)

Read-only from this repo; per CLAUDE.md I do not read or audit sibling repos. Each item states
the assumption and what's needed to confirm it.

1. **Web `/verify` page target (verification & password-reset links).** If the backend generates
   Firebase action links (`generateEmailVerificationLink` / `generatePasswordResetLink`), each needs
   an `ActionCodeSettings` continue-URL pointing at a web page that completes the action and then
   lands the user somewhere sensible. **Assumption:** web exposes a verify/reset landing route
   (path TBD — `/verify`? `/auth/action`?) and a base URL per base-site/locale. **Need from web:**
   the exact URL(s), whether per-base-site/locale, and whether web (not backend) is already
   generating these links via the client SDK today (if so, the backend should not duplicate them).
   Also: the action-link domain must be authorized in the Firebase console.

2. **Router (Cloudflare Worker) verification-link routing.** Any link domain the email points at
   passes through the edge worker. **Assumption:** the worker forwards the verify/reset path to web
   origin and does not strip Firebase's `oobCode`/`mode` query params, and is not gated by
   maintenance for that path. **Need from router:** confirmation the verify path is routed to web
   and query strings are preserved end-to-end.

3. **Firebase action-handler vs. custom page.** Firebase can host its own default action handler,
   or links can be customized to the app's own page. **Assumption:** the project uses a custom
   continue-URL into web rather than Firebase's default handler. **Need:** confirm which (this is a
   Firebase-console + web decision), since it changes what `ActionCodeSettings` the backend builds.

4. **Firestore.** No Firestore touch is required for transactional email itself. The only adjacent
   Firestore concern is the **hard-delete chat cleanup** (`PendingChatCleanup` queued in
   `runHardDelete` for the Sunday messaging cron) — unrelated to email, noted only so the email
   feature does not assume it owns the deletion-completion side effects. **Assumption:** email is
   Postgres + Brevo only; no Firestore reads/writes added. **Need:** none, unless an "account
   deleted" email must reflect chat-cleanup state (it should not).

5. **Recipient email source for the banned/re-reg path (8) and hard-delete confirmations.** For
   re-registration-blocked there is no local user; the email is the only identity and language must
   come from the request `X-Lang` or a default (Section 4.8). For an "account deleted" confirmation,
   the email must be snapshotted before the in-tx user delete (Section 4 cross-cutting flag).
   **Need from product/Mastermind:** confirmation these two emails are even in scope, given the
   identity/language constraints.

---

## Out-of-scope observations (Part 4b)

- **(low/medium)** `EmailService` + Brevo config + `EmailProperties` are fully wired but have
  **zero production callers** (only `DefaultEmailServiceTest`). Dead-until-used scaffolding from a
  prior brief. Not a bug; flagged so Mastermind knows the send path has never executed against a
  real SMTP server in any running profile. The brief's "test endpoint" was the intended smoke path
  and was never built (see "Brief vs reality").
- **(low)** `getBackendTranslation` uses `.orElseThrow()` with no fallback — a missing key or
  missing locale throws at send time. Existing push code lives with this; the email feature should
  ensure all four locales exist for every new key (or guard), especially for cron sends where an
  exception would abort a batch row.
- **(low)** `emailVerifiedExternal` is captured once at registration and never refreshed
  (Section 3.2). Latent staleness if anything ever reads the DB column as authoritative. The
  display-only `UserInfoDTO.verified` is the only reader today, so it's harmless now — but a future
  gate that reads the column instead of the live token would inherit stale data.

I did not fix any of these — read-only audit, out of scope.

---

## Config-file impact

- conventions.md: no change.
- decisions.md: no change (this is a Phase-2 audit; any decisions come from Mastermind's seam
  analysis).
- state.md: no change required by me. (When the feature is planned, Docs/QA would add an
  `email-notifications` pipeline row — drafting that is Mastermind's, not this audit's.)
- issues.md: no change required by me. The "Brief vs reality" non-existent test endpoint and the
  three non-existent transitions are reported here for Mastermind's seam analysis; whether any
  become `issues.md` entries is Mastermind's call.

No implicit config-file dependency is created by this audit.
