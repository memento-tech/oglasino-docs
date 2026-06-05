# Audit — password-reset — oglasino-backend

**Repo:** oglasino-backend
**Branch:** dev
**Type:** READ-ONLY audit. No code changes made.
**Date:** 2026-06-03
**Scope:** the current backend auth + email surface, audited from code (not from the email-notifications spec), to ground a future password-reset feature spec.

All file:line references are from a read of the working tree on `dev` at audit time.

---

## 0. Headline findings (read this first)

1. **The Admin SDK can do everything a password-reset endpoint needs, but the project does not call those methods today.** `firebase-admin` is at **9.9.0** (`pom.xml:106-107`). That version exposes `generatePasswordResetLink(email, ActionCodeSettings)` and `getUserByEmail(email) → UserRecord → getProviderData() → UserInfo[].getProviderId()`. **Neither is called anywhere in the repo** (grep: `generatePasswordResetLink` → none; `getUserByEmail`/`getProviderData` on `FirebaseAuth` → none). They would be new wiring, but they sit on the same already-initialized `FirebaseAuth.getInstance()` the project already uses for `generateEmailVerificationLink`, `getUser`, `updateUser`, `deleteUser`, `revokeRefreshTokens`.
2. **`getUserByEmail` in this codebase means the *Postgres* lookup**, not the Admin SDK. `UserService.getUserByEmail` (`DefaultUserService:59`) is `userRepository.findByEmail` against our DB. The resend-verification endpoint uses *that* (DB) to decide whether to send. A password-reset endpoint that needs the **Firebase sign-in provider** for an email cannot get it from the DB reliably (see §3) and must call the Admin SDK's `getUserByEmail`.
3. **The resend-verification endpoint is a complete, mirror-able no-leak pattern** — unauthenticated, email-only body, two Redis throttles, identical success body on real-send and no-op. Password-reset can sibling it almost 1:1 (§1).
4. **Trust boundary holds**: the only client input on both the existing resend endpoint and a future reset endpoint is the email. Existence, provider, and verified-state are all server-derivable (DB for existence/verified; Admin SDK for provider). Nothing forces trusting the client — **provided** the provider check uses the Admin SDK and not the persisted `registeredWithProvider` column (which is garbage — §3, and `issues.md` 2026-06-03).

---

## 1. The resend-verification endpoint (the pattern to sibling)

### Location & shape

- **Controller:** `controller/AuthController.java`, method `resendVerification` (`AuthController:187-258`).
- **Mapping:** `@PostMapping("/resend-verification")` on a class-level `@RequestMapping("/api/auth")` → **`POST /api/auth/resend-verification`**.
- **Request body:** `ResendVerificationRequest` (`dto/ResendVerificationRequest.java`) — a single-field record `record ResendVerificationRequest(String email) {}`. Note: **no `@Valid`, no Jakarta constraints** on this DTO; the controller does its own null/blank handling (`StringUtils.trimToNull`, then `EMAIL_REQUIRED` if null). Contrast `firebase-sync`, which uses `@Valid LoginRequest`.
- **Auth:** **Unauthenticated.** `SecurityConfig` permits `/api/auth/**` (`SecurityConfig:81-82` `.requestMatchers("/api/public/**", "/api/auth/**", "/internal/**").permitAll()`), and `/api/auth/**` is also in the security-filter skip list (`SecurityConfig:28`). No Firebase principal is read — the recipient comes from the body. Documented rationale (`AuthController:160-166`): under the strict sign-out model an unverified user has no session when they click "Resend".

### Email normalization

`email = email.toLowerCase(Locale.ROOT)` (`AuthController:195`) before keying throttles and the user lookup — matches Firebase's lowercased storage and the stored `users.email`.

### Redis throttle mechanism

Two independent **email-keyed** limits, both on `StringRedisTemplate redisTemplate` (`AuthController:83`). Constants (`AuthController:67-71`):

| Constant | Value |
| --- | --- |
| `RESEND_COOLDOWN_KEY_PREFIX` | `"verify:resend:cooldown:"` |
| `RESEND_DAILY_KEY_PREFIX` | `"verify:resend:daily:"` |
| `RESEND_COOLDOWN` | `Duration.ofSeconds(60)` |
| `RESEND_DAILY_WINDOW` | `Duration.ofHours(24)` |
| `RESEND_DAILY_LIMIT` | `4` |

Keys are `prefix + email` (lowercased): `verify:resend:cooldown:<email>` and `verify:resend:daily:<email>`.

**60s gap (`AuthController:200-214`):** `redisTemplate.opsForValue().setIfAbsent(cooldownKey, "1", RESEND_COOLDOWN)` (SET-NX-EX). First caller in the window takes the slot. On `Boolean.FALSE` (slot already held), it reads remaining TTL via `getExpire(cooldownKey, TimeUnit.SECONDS)` and returns 429 `VERIFICATION_RESEND_COOLDOWN` with `retryAfterSeconds = ttl > 0 ? ttl : 60`.

**4-per-24h cap (`AuthController:216-230`):** `redisTemplate.opsForValue().increment(dailyKey)`. On the **first** increment (`daily == 1L`) it stamps `expire(dailyKey, RESEND_DAILY_WINDOW)` (24h TTL set once, so the window is fixed from first use, not sliding). If `daily > RESEND_DAILY_LIMIT` (i.e. the 5th attempt), it **releases the cooldown slot it just took** (`redisTemplate.delete(cooldownKey)` — because this request is rejected, not served) and returns 429 `VERIFICATION_RESEND_DAILY_LIMIT`.

### How/when limits are released vs held (critical for the no-leak property)

- **Held (not released):** the no-leak no-op paths (unknown address, already-verified). Both limits stay consumed so timing/limits can't probe registration (`AuthController:232-244`).
- **Released:** only a **send failure**. On `MailException` from `verificationEmailSender.send(...)`, it `delete(cooldownKey)` **and** `decrement(dailyKey)` (`AuthController:248-255`), so a transient relay error neither locks the user for 60s nor burns a daily attempt.
- The daily-cap rejection releases the cooldown slot only (the daily counter stays incremented for that over-limit attempt).

### No-account-existence-leak handling

After the throttles pass: `Optional<User> recipient = userService.getUserByEmail(email)` (DB lookup, `AuthController:234`).

- `recipient.isEmpty()` → log "no account", return **200** `VERIFICATION_EMAIL_SENT` (no mail sent) (`:235-238`).
- `recipient.get().isEmailVerifiedExternal()` → log "already verified", return **200** `VERIFICATION_EMAIL_SENT` (no mail sent) (`:239-244`). ⚠️ See §3/§5 caveat: `emailVerifiedExternal` is stale-by-design (`issues.md` 2026-06-03) — for *resend* this is acceptable per the email-notifications close-out (the throttles + no-leak bound the harmless re-send), but a password-reset design must not lean on this column for any authoritative decision.
- Otherwise → `verificationEmailSender.send(recipient.get())`, return **200** `VERIFICATION_EMAIL_SENT`.

All three branches return the **same success-shaped body**; only logging differs.

### Every coded response

| HTTP | Code (`errors[].code`) | translationKey | Body type | Extra payload |
| --- | --- | --- | --- | --- |
| 200 | `VERIFICATION_EMAIL_SENT` | — | `VerificationResendResult(String code)` | none |
| 400 | `EMAIL_REQUIRED` | `email.required` | `ProductErrorResponse` | none |
| 429 | `VERIFICATION_RESEND_COOLDOWN` | `email.verification.cooldown` | `VerificationResendCooldownResponse(List<FieldError> errors, long retryAfterSeconds)` | **`retryAfterSeconds`** |
| 429 | `VERIFICATION_RESEND_DAILY_LIMIT` | `email.verification.daily_limit` | `ProductErrorResponse` | none (no countdown — distinct code is how the client tells the two 429s apart) |
| 502 | `EMAIL_SEND_FAILED` | `email.send_failed` | `ProductErrorResponse` | none |

Envelope shapes:
- `VerificationResendResult` (`dto/VerificationResendResult.java`): `record VerificationResendResult(String code) {}` — **note this success body carries a bare `code` string, NOT the `{errors:[...]}` envelope.**
- `VerificationResendCooldownResponse` (`dto/VerificationResendCooldownResponse.java`): `record (List<ProductErrorResponse.FieldError> errors, long retryAfterSeconds)` — mirrors the standard `{errors:[{field,code,translationKey}]}` envelope **plus** a top-level `retryAfterSeconds`, so generic client error handling still finds the code in `errors[].code`.
- The plain error paths use `AuthController.errorResponse(status, code, translationKey)` (`:311-317`) → `ProductErrorResponse(List.of(new FieldError(null, code, translationKey)))` — `field` is always `null` (object-level).

### How it mints the verification link

In `VerificationEmailSender.buildVerifyUrl(email, langCode)` (`service/impl/VerificationEmailSender.java:66-95`):

```java
ActionCodeSettings settings =
    ActionCodeSettings.builder()
        .setUrl(webProperties.getBaseUrl())     // app.web.base-url (dev: https://stage.oglasino.com)
        .setHandleCodeInApp(false)
        .build();
String firebaseLink = FirebaseAuth.getInstance().generateEmailVerificationLink(email, settings);
```

It then **discards Firebase's link and keeps only the `oobCode`**: parses `firebaseLink` with `UriComponentsBuilder`, pulls `getQueryParams().getFirst("oobCode")`, throws `IllegalStateException` if blank, and rebuilds the URL as:

```
<webBase>/<compoundLocale>/verify?oobCode=<oobCode>
```

- `<webBase>` = `WebProperties.getBaseUrl()` (`@ConfigurationProperties("app.web")`, `properties/WebProperties.java`; configured `app.web.base-url`, dev `${WEB_BASE_URL:https://stage.oglasino.com}` at `application-dev.yaml:183-184`).
- `<compoundLocale>` = `WebLocales.compound(langCode)` (`service/impl/WebLocales.java`): `sr→rs-sr`, `en→rs-en`, `ru→rs-ru`, `cnr→me-cnr`, default `rs-sr`. (Comment notes backend↔web coupling — if web's routing locale set changes, this map must change.)
- The web `/verify` page is what applies the action; the backend only embeds the `oobCode`.

`FirebaseAuthException` from link generation is wrapped as `IllegalStateException` (NOT a `MailException`), so it would propagate as a 500 — distinct from the 502 `EMAIL_SEND_FAILED` SMTP path.

> **Password-reset mirror note:** `generatePasswordResetLink(email, settings)` returns a link with the same `oobCode` query param shape, so the exact `UriComponentsBuilder` extract-and-rewrite trick works unchanged — a reset sender would build `<webBase>/<locale>/reset?oobCode=…` (whatever the web reset route is) the same way.

---

## 2. The Admin SDK password-reset capability

### Initialization / injection

- **Init:** `security/config/FirebaseConfig.java` — `@Configuration`, `@PostConstruct init()` builds `FirebaseOptions` from `GoogleCredentials.fromStream(...)` at `app.firebase.credentials-path` and calls `FirebaseApp.initializeApp(options)` once (guarded by `FirebaseApp.getApps().isEmpty()`). No `FirebaseAuth` bean is exposed — every caller uses the static `FirebaseAuth.getInstance()`.
- **Usage convention:** all Admin SDK calls go through `FirebaseAuth.getInstance().<method>(...)` directly inside services (e.g. `VerificationEmailSender:75`, `DefaultFirebaseUserService`, `DefaultFirebaseAuthService:62`). There is no injected wrapper around `FirebaseAuth`.

### `generatePasswordResetLink` availability

- **Library:** `firebase-admin` **9.9.0** (`pom.xml:106-107`).
- That version's `com.google.firebase.auth.FirebaseAuth` exposes **`String generatePasswordResetLink(String email)`** and **`String generatePasswordResetLink(String email, ActionCodeSettings settings)`**, both throwing `FirebaseAuthException` — the exact sibling of the already-used `generateEmailVerificationLink(String, ActionCodeSettings)`.
- **Not called anywhere today** (grep confirmed). It is available; it is unused.

### `ActionCodeSettings` / continue-URL construction (to mirror)

The verification path's `ActionCodeSettings` (`VerificationEmailSender:67-71`) is the template:
- `.setUrl(webProperties.getBaseUrl())` — the continue URL (base only).
- `.setHandleCodeInApp(false)`.
- Builder pattern, no dynamic-link domain set.

A reset sender would build the same `ActionCodeSettings`, call `generatePasswordResetLink`, extract `oobCode`, and rewrite to a web reset route. The `<webBase>` and `WebLocales.compound` helpers are reusable as-is (`WebLocales` is package-private in `service.impl`, so a reset sender placed in that package can reuse it directly — same as `WelcomeEmailSender`).

---

## 3. Provider detection (CRITICAL — trust boundary)

### Can the backend determine an account's sign-in provider for an email, server-side?

**Yes — but only via an Admin SDK call that the project does not make today.**

- The Admin SDK path is `FirebaseAuth.getInstance().getUserByEmail(email)` → `UserRecord` → `UserRecord.getProviderData()` returns `UserInfo[]`, each `UserInfo.getProviderId()` yielding `password`, `google.com`, etc. `UserRecord` also has `getEmail()`, `isEmailVerified()`, `isDisabled()`, `getUid()`. **None of this is wired in the repo** (grep: no `getProviderData`, no `FirebaseAuth...getUserByEmail`, no `com.google.firebase.auth.UserInfo` import).

### What the codebase does today (and why it is NOT a usable provider source for reset)

1. **`getUserByEmail` here is the DB lookup, not the SDK.** `UserService.getUserByEmail` → `DefaultUserService:59` → `userRepository.findByEmail(email)` (`repository/UserRepository.java:20`). Returns `Optional<User>` from Postgres. This tells you **existence** and gives the `User` row, but not the live Firebase provider list.
2. **The persisted provider column is garbage.** In `DefaultFirebaseAuthService.getOrCreateUser` (`:100-101`) the provider is captured as:
   ```java
   var providerId = Optional.ofNullable(claims.get("firebase")).map(Object::toString).orElse("");
   ```
   i.e. the **`firebase` claim Map's `toString()`** (e.g. `{sign_in_provider=password, identities=...}`), and that is what gets stored into `User.registeredWithProvider` (`:114`, and again at creation `:144`). **This is a known bug** (`issues.md` 2026-06-03 "`registeredWithProvider` stored as the firebase-claim Map's `toString()`"). A reset design must **not** read `registeredWithProvider` for the provider check.
3. **The one correct provider read is token-bound, not email-bound.** `DefaultFirebaseAuthService.extractSignInProvider(token)` (`:179-188`) correctly pulls the nested `firebase.sign_in_provider` claim off a verified `FirebaseToken`. But that requires a **live ID token** — which a password-reset request (unauthenticated, email-only) does not have. So this method is unusable for reset.

**Conclusion:** for an unauthenticated, email-only reset request, the *only* reliable server-side provider source is the **Admin SDK `getUserByEmail(email).getProviderData()`** — new wiring, available in 9.9.0.

### What happens when `getUserByEmail` is called for an email with no account?

- **DB version (today's code):** `userRepository.findByEmail` → `Optional.empty()`. No exception. This is exactly how `resendVerification` branches the no-leak no-op (`AuthController:235`).
- **Admin SDK version (would be new):** `FirebaseAuth.getInstance().getUserByEmail(email)` **throws `FirebaseAuthException`** for an unknown email, with `getAuthErrorCode() == AuthErrorCode.USER_NOT_FOUND`. This is the same `USER_NOT_FOUND` `AuthErrorCode` the project already switches on in `DefaultFirebaseUserService` (`:46`, `:60`, `:72`) for `deleteUser`/`getUser`/`revokeRefreshTokens`. So the existing idempotent-on-USER_NOT_FOUND pattern is the template for the no-leak branch.

### Can the backend reliably tell apart (a) unknown email, (b) password account, (c) social-only account?

**Yes, server-side, via the Admin SDK — once wired:**
- **(a) unknown email** → `getUserByEmail` throws `FirebaseAuthException` / `USER_NOT_FOUND`.
- **(b) password account** → `getUserByEmail` returns a `UserRecord` whose `getProviderData()` contains a `UserInfo` with `getProviderId() == "password"`.
- **(c) social-only account** → `UserRecord` whose provider list contains only social ids (e.g. `google.com`) and no `password`.

The decision (send a reset link only for accounts that actually have a password provider) is therefore fully server-derivable from the email alone, with no client-trusted value. **The DB alone cannot do this** (existence yes, provider no — column is garbage). This is the single new dependency a reset endpoint introduces.

---

## 4. The email send machinery (the building blocks to reuse)

### `EmailService.sendHtml`

- **Interface:** `service/EmailService.java` — `sendHtml(String to, String subject, String htmlBody, String textFallback)` and `sendPlainText(String to, String subject, String body)`. Throws `org.springframework.mail.MailException` on SMTP failure (documented contract; callers like resend surface it).
- **Impl:** `service/impl/DefaultEmailService.java`. `sendHtml` (`:44-59`) builds a `MimeMessage` via `MimeMessageHelper(message, true /*multipart*/, UTF-8)`, sets From/From-name/Reply-To from `EmailProperties` (never client-supplied), `setTo(to)`, `setSubject`, and `helper.setText(textFallback, htmlBody)` — multipart/alternative so non-HTML clients get the fallback. Dispatch via injected `JavaMailSender mailSender`. On `MailException` it logs `subject` (newline-stripped via `Sanitizer.stripNewlines`) — **never the recipient** — and rethrows (`:67-74`).

### Branded `EmailLayout` shell

- `service/impl/EmailLayout.java`, `@Component`, method `wrap(String innerBodyHtml)` (`:37-78`). Returns a full HTML doc: table-based outer layout, inline CSS only (no `<style>`/external CSS), light shell (`#f4f4f5` bg, white 600px card, 8px radius), logo header, and a footer with `Oglasino` + a `mailto:` to the configured Reply-To.
- **Logo:** R2 brand asset `public/brand/light-oglasino-full.png`, URL = `imageProperties.getCdnBaseUrl() + "/" + ImagePaths.PUBLIC_BRAND + LOGO_FILENAME` (per-env CDN base, dev `https://cdn-staging.oglasino.com`).
- **i18n-agnostic:** the shell adds only brand chrome; callers pass already-localized inner HTML.

### Plain-text fallback assembly

Each sender builds its own plain text. Pattern from `VerificationEmailSender.buildPlainText` (`:123-134`): concatenate the same translated strings (heading, body, link, note, regards, team) with `\n` separators. The HTML inner body is built by `buildInnerHtml` (`:97-121`) with a Java text-block + `.formatted(...)`, then wrapped via `emailLayout.wrap(...)`. **No templating engine** — bodies are assembled in plain Java (the interface Javadoc states this is deliberate).

### Recipient language resolution / passing

- **For a known user:** language comes from the **server-side user row** — `recipient.getPreferredLanguage().getCode()` (`VerificationEmailSender:49`), backed by `users.preferred_language_id`. Never a client value.
- **For a recipient with no user row** (the reblock email, an analog for a reset on a non-existent/social account if a design ever mails those): language is resolved from the request `X-Lang` header, normalized to one of the seeded locales, defaulting to Serbian — `AuthController.resolveReblockLanguage` (`:300-309`) with `EMAIL_LANGS = {en, sr, ru, cnr}` and `DEFAULT_EMAIL_LANG = "sr"` (`:89-90`). (Password-reset's no-leak design will need to decide language for the unknown-email branch the same way — but the unknown branch sends *no* mail, so this only matters for the real-send path, which has a user row.)
- **Translation lookup:** `TranslationService.getBackendTranslation(key, langCode)` (`DefaultTranslationService:106-110`):
  ```java
  return Optional.ofNullable(backendTranslations.get(code)).map(m -> m.get(langCode)).orElseThrow();
  ```
  Backed by an in-memory index `backendTranslations` (`Map<key, Map<langCode, value>>`) built at `@PostConstruct` from `TranslationNamespace.BACKEND_TRANSLATIONS` rows and rebuilt on admin edits (`indexBackendTranslations`, `:135-146`).
  - **`.orElseThrow()` missing-key behavior:** a missing **key** OR a missing **langCode** for an existing key throws **`NoSuchElementException`** (bare `orElseThrow()`). There is no fallback to another locale and no default string — a gap in any of the four locales for a used key is a hard runtime failure on send. So any new password-reset key MUST be seeded in all four locales.

### `BACKEND_TRANSLATIONS` seed pattern

- **Files (4, one per locale):** `src/main/resources/data/translations/0001-data-web-translations-{EN,RS,RU,CNR}.sql`.
- **Row shape:** `(<id>, <langId>, 'BACKEND_TRANSLATIONS', '<key>', '<value>', CURRENT_TIMESTAMP),`. Single-quotes in values are SQL-escaped by doubling (`didn''t`).
- **Per-file langId + ID block (BACKEND_TRANSLATIONS group):**
  | File | langId | BACKEND_TRANSLATIONS id range (current) |
  | --- | --- | --- |
  | EN | 3 | 2100 → 2166 |
  | RS | 1 | 4200 → … |
  | RU | 4 | 6300 → … |
  | CNR | 2 | 0 → … |
  Each locale file uses its own ID offset for the same namespace; the same key gets a different numeric id in each file.
- **Inline-append shape:** new keys are appended at the **end of the BACKEND_TRANSLATIONS group** in each of the four files (the group currently tails at `email.welcome.*`, EN ids 2163-2166). Per project convention (conventions Part 6 Rule 3) and the operator's standing guidance, this is an **inline append into the existing 0001 files** — not a new dedicated file (the dedicated-file exception is strictly multi-namespace).
- **Next available ID:** take the highest id within that file's BACKEND_TRANSLATIONS group and add one (EN → 2167, etc.). Per the operator's standing rule, **translation IDs are disposable** — on a collision just pick the next free id and continue; the operator renumbers post-feature. No STOP/coordinate needed for an ID collision.
- **Existing email key families to mirror (all four locales seeded):** `email.common.signoff.{regards,team}`, `email.verification.{subject,heading,body,cta,note}`, `email.deletion.*`, `email.banned.*`, `email.restored.*`, `email.reblock.*`, `email.review.*`, `email.welcome.*`. A password-reset family would naturally be `email.password.reset.{subject,heading,body,cta,note}` (+ reuse the shared `email.common.signoff.*`).

---

## 5. The existing-leak posture (CRITICAL — informs the no-leak design)

### Does the backend own register/login?

**No.** Account creation and login happen on the **client via the Firebase SDK**. The backend's only auth entry point is `POST /api/auth/firebase-sync` (`AuthController.firebaseSync`, `:92-158`), which the client calls **with an already-valid Firebase ID token** (`Authorization: Bearer <idToken>`) to mint/sync the local `User` row. So `firebase-sync` is **not** an unauthenticated email-enumeration surface — you must already hold a valid token for the account.

### What `firebase-sync` reveals (response shapes)

`firebase-sync` verifies the token, reads the email, then branches:

| Condition | HTTP | Body |
| --- | --- | --- |
| Email is banned (hash match, `isEmailBanned`) | **403** | `ProductErrorResponse` code `EMAIL_BANNED`, translationKey `email.banned`. Side effects: best-effort `deleteFirebaseUser(uid)` and a once-per-ban reblock email (`:109-118`). |
| Existing user is admin-disabled (`user.isDisabled()`) | **403** | `ProductErrorResponse` code `USER_BANNED`, translationKey `user.banned` (`:131-133`). |
| Sync ok (new OR existing user) | **200** | `AuthUserDTO` (ModelMapper from `User`) + header `X-Account-Restored: true` if a pending-deletion was auto-cancelled (`:147-157`). |
| `syncResult`/user null | **500** | empty body (`:122-124`). |

**`wasRegister` distinguishes create-vs-match** (`AuthUserDTO.setWasRegister(syncResult.wasRegister())`, `:151`) — but only on a 200, i.e. only for a caller already holding that account's valid token (web GA4 reads it as the sign_up-vs-login discriminator). It is **not** an oracle an unauthenticated attacker can hit with an arbitrary email.

### Does any backend response distinguish account-exists from account-absent (to an unauthenticated probe)?

- **`firebase-sync`:** gated behind a valid Firebase token, so not an enumeration oracle for an attacker who doesn't already control the account.
- **`resend-verification`:** **deliberately does NOT** distinguish — unknown email, already-verified, and real-send all return identical 200 `VERIFICATION_EMAIL_SENT`, with both rate limits consumed identically so timing can't probe either (§1).
- **`GET /api/auth/firebase/{firebaseId}`** (`AuthController:260-267`) is keyed by **firebaseId**, not email, and returns `UserInfoDTO` via the facade — not an email-enumeration surface.
- **`UserInfoDTO.verified`** is the only live reader of `emailVerifiedExternal` (display only; `issues.md` 2026-06-03).

**Net:** the backend today exposes **no unauthenticated email→exists oracle**. A password-reset endpoint must preserve this — return an identical success-shaped body whether the email is unknown, social-only, or a real password account, and consume any rate limits identically across all branches (mirror §1 exactly).

### Banned-email re-registration (`EMAIL_BANNED`)

Covered above: `firebase-sync` returns **403 `EMAIL_BANNED`** (`email.banned`) for a hash-matched banned email, deletes the freshly-created Firebase user (best-effort), and sends the reblock email once per ban. This is the only place `EMAIL_BANNED` is emitted (grep). For password-reset, a banned email should presumably fall into the no-leak no-op (no reset link for a banned account) rather than emit a distinguishing code — a design decision, flagged here, not made.

---

## 6. Firebase project account-linking setting ("one account per email" vs "multiple")

**Not determinable from this repo.** This is a **Firebase Console** setting (Authentication → Settings → "User account linking": *"Link accounts that use the same email"* vs *"Create multiple accounts for each identity provider"*). It is not represented in any committed config — `FirebaseConfig` only loads the service-account credentials and calls `initializeApp`; there is no `FirebaseOptions` field, env var, or yaml key for the account-linking mode. Nothing in `application-*.yaml`, `pom.xml`, or the Firebase init code encodes it.

**→ Igor must confirm this in the Firebase console** for each project (`oglasino-stage-49abb`, `oglasino-prod-7e5db`). It directly determines whether `getUserByEmail(email).getProviderData()` can return *multiple* `UserInfo` entries for one email (multiple-accounts mode → an email could have both a `password` and a `google.com` record as separate accounts, complicating the (b)/(c) distinction in §3) or whether it is one record per email. The audit cannot answer it; the §3 provider logic should be designed against the confirmed setting.

> Per the brief: this audit reports the *setting* only and does not touch the account-linking *fix* (separate future feature).

---

## 7. Re-registration-blocked + ban email flows (sanity check on the email-hash machinery)

The reset provider check operates on the same "look up by email before any user-row context" basis as the ban machinery, so confirming that machinery:

- **`isEmailBanned(email)`** — `DefaultUserAuditService:36-41`: `bannedUserAuditRepository.findByEmailHashAndRetentionUntilAfter(hashEmail(email), Instant.now()).isPresent()`. Looks up by **email hash** within the retention window; the raw email is never stored.
- **`hashEmail(email)`** — `:26-28`: `sha256Hex(email.trim().toLowerCase())` — SHA-256 hex of the trimmed, lowercased email. So the lookup key is stable across casing/whitespace, matching the lowercasing the resend/reset endpoints already apply.
- **Repository:** `BannedUserAuditRepository` (`repository/BannedUserAuditRepository.java`) — `findByEmailHashAndRetentionUntilAfter(emailHash, now)` and `findByEmailHash(emailHash)`. Entity `BannedUserAudit` carries `emailHash`, `banReason`, `bannedAt`, `bannedByAdminId`, `retentionUntil`, `reblockNotifiedAt`.
- **Reblock send-once guard:** `isReblockNotificationPending` (`:92-99`, true only when an audit row exists with `reblockNotifiedAt == null`) + `markReblockNotified` (`:101-111`). The once-per-ban flag (`reblock_notified_at`) is itself the abuse guard (`AuthController.notifyReblockOnce`, `:283-298`); a re-ban replaces the audit row with a fresh unstamped one.

**Relevance to reset:** the same `hashEmail` + `BannedUserAuditRepository` lookup is available to a reset endpoint if the design wants to suppress reset links for banned emails — it operates purely on the email, pre-user-row, exactly as needed. The hash is one-way, so a reset endpoint can check "is this email banned?" without the email ever being stored.

---

## Trust-boundary summary (conventions Part 11)

For a future `POST /api/auth/{password-reset}` request, the only client-supplied value is **email**. Everything that drives a decision is server-derivable:

| Decision | Source | Trusted from client? |
| --- | --- | --- |
| Does an account exist? | Admin SDK `getUserByEmail` (throws `USER_NOT_FOUND`) — or DB `userRepository.findByEmail` for existence-only | No — server-derived |
| Which provider(s)? | Admin SDK `getUserByEmail(...).getProviderData()[].getProviderId()` (**new wiring**) | No — server-derived. ⚠️ Do NOT use `User.registeredWithProvider` (garbage, `issues.md` 2026-06-03) |
| Is the email banned? | `UserAuditService.isEmailBanned(email)` (hash lookup) | No — server-derived |
| Recipient language | `user.getPreferredLanguage()` (DB) for real sends; `X-Lang`→default for no-row branches | No — server-derived/DB |
| Whether to send & which link | Server logic over the above; link minted by Admin SDK `generatePasswordResetLink` | No — server-derived |

**Nothing forces trusting the client.** The single new dependency is the Admin SDK `getUserByEmail`/`getProviderData` call for provider detection — available in firebase-admin 9.9.0, unused today, and the *only* reliable server-side provider source for an unauthenticated email-only request. The persisted provider column must not be used. Two console-level unknowns sit outside the code: the account-linking setting (§6, Igor to confirm) and the implication it has for multi-provider emails in §3.

---

## Appendix — exact files read

- `controller/AuthController.java`
- `dto/ResendVerificationRequest.java`, `dto/VerificationResendResult.java`, `dto/VerificationResendCooldownResponse.java`
- `service/impl/VerificationEmailSender.java`, `service/impl/EmailLayout.java`, `service/impl/WebLocales.java`
- `service/EmailService.java`, `service/impl/DefaultEmailService.java`
- `service/impl/DefaultTranslationService.java`
- `security/service/impl/DefaultFirebaseAuthService.java`
- `admin/service/FirebaseUserService.java`, `admin/service/impl/DefaultFirebaseUserService.java`
- `security/config/FirebaseConfig.java`
- `service/impl/DefaultUserAuditService.java`, `repository/BannedUserAuditRepository.java`
- `properties/WebProperties.java`, `application-dev.yaml` (web base-url, cdn base-url)
- `src/main/resources/data/translations/0001-data-web-translations-{EN,RS,RU,CNR}.sql` (BACKEND_TRANSLATIONS rows)
- `pom.xml` (firebase-admin 9.9.0)
- `security/config/SecurityConfig.java` (`/api/auth/**` permitAll + filter skip)
- Grep sweeps confirming **absence** of `generatePasswordResetLink`, Admin-SDK `getUserByEmail`, `getProviderData`, `com.google.firebase.auth.UserInfo` usage.
</content>
</invoke>
