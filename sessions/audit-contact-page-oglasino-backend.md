# Audit — oglasino-backend — contact/support feature

Read-only. Code is the source of truth. Each claim below was verified against the
actual file (Read) and located via `rg`. There is **no existing contact/support
endpoint, entity, or DTO** in this repo — the only `Contact*` symbol is
`ContactAnalyzer` (`moderation/analyzer/ContactAnalyzer.java`), a product-listing
moderation analyzer that detects inline contact info in ads. Unrelated.

---

## 1. Email send path

### Public send API (Brevo SMTP via `JavaMailSender`)

`EmailService` (`src/main/java/com/memento/tech/oglasino/service/EmailService.java`)
is the outbound port. Two methods only:

- `void sendPlainText(String to, String subject, String body)` — line 22
- `void sendHtml(String to, String subject, String htmlBody, String textFallback)` — line 33

`DefaultEmailService` (`service/impl/DefaultEmailService.java`) is the single impl:

- `sendPlainText` (lines 27–42): builds a non-multipart `MimeMessageHelper`,
  `helper.setText(body, false)`.
- `sendHtml` (lines 44–59): multipart `MimeMessageHelper(message, true, …)`,
  `helper.setText(textFallback, htmlBody)` — `multipart/alternative`, plain-text
  fallback first.
- Both call private `send(message, subject)` (lines 67–74), which dispatches via
  `mailSender.send(message)` and **rethrows** `MailException` after logging the
  newline-stripped subject only (never the recipient).

The transport is Spring's `JavaMailSender` (Brevo SMTP relay). Template rendering
is explicitly out of scope (EmailService.java lines 9–11): HTML bodies are
assembled in plain Java by the layout helper.

**Implication for contact form:** the port already supports exactly what a
support email needs — a branded HTML body + plain-text fallback to a fixed
recipient (the support inbox). No new send primitive is required.

### Branded-email shell — reusable HTML wrapper exists

`EmailLayout` (`service/impl/EmailLayout.java`) is a `@Component` with one method:

- `String wrap(String innerBodyHtml)` (lines 37–78): wraps caller-supplied inner
  HTML in the branded shell — logo header (R2 brand asset
  `public/brand/light-oglasino-full.png` resolved through
  `ImageProperties.getCdnBaseUrl()`, lines 38–39) + a footer that renders the
  support contact as a `mailto:` link taken from `EmailProperties.getReplyTo()`
  (line 40, used at lines 66–67). Inline CSS, table layout, no `<style>` block.

It is i18n-agnostic — callers pass already-localized inner HTML (javadoc lines
15–17). This is a genuine reusable wrapper, not plain-text-only.

**Precedent caller to copy:** `TransactionalEmailSender`
(`service/impl/TransactionalEmailSender.java`) is the closest model. It:
- resolves localized strings via `TranslationService.getBackendTranslation`,
- builds inner HTML (`buildInnerHtml`, lines 66–118) and a parallel plain-text
  body (`buildPlainText`, lines 120–144),
- wraps the HTML with `emailLayout.wrap(...)` (line 59),
- dispatches via `emailService.sendHtml(email, subject, html, text)` (line 62),
- reads recipient address + language from a server-side `User` record, **never a
  client value** (lines 52–55; javadoc lines 24–27).

`VerificationEmailSender` / `WelcomeEmailSender` follow the same shape.

### From / Reply-To handling and where the addresses live

Envelope is set server-side in `DefaultEmailService` from `EmailProperties`,
never client-supplied:
- `helper.setFrom(emailProperties.getFromAddress(), emailProperties.getFromName())`
  — DefaultEmailService.java lines 33, 50
- `helper.setReplyTo(emailProperties.getReplyTo())` — lines 34, 51

`EmailProperties` (`properties/EmailProperties.java`) binds `oglasino.email.*`
(lines 19–21) with three fields: `fromAddress`, `fromName`, `replyTo`. No baked-in
defaults — a missing var fails startup (javadoc lines 11–14).

YAML wiring (all three profiles, env-var indirection — actual values live in GH
Secrets / deploy env, **not committed**):
- `application-prod.yaml` lines 316–319: `from-address: ${OGLASINO_EMAIL_FROM_ADDRESS}`,
  `from-name: ${OGLASINO_EMAIL_FROM_NAME}`, `reply-to: ${OGLASINO_EMAIL_REPLY_TO}`
- `application-stage.yaml` lines 330–333 — same keys.
- `application-dev.yaml` lines 305–308 — same keys.

Documented address semantics (EmailProperties javadoc + YAML comments):
- **`noreply@oglasino.com`** — the prod From (`noreply-stage@oglasino.com` in
  stage/dev). Brevo-verified sender.
- **`support@oglasino.com`** — the Reply-To target on every environment, "the
  monitored support inbox" (EmailProperties.java javadoc; also surfaced to users
  in translation seeds, e.g. `banned.dialog.body.*`, `user.locked.from.deletion`).
- **`privacy@`** — **does not exist anywhere** in the codebase. If the contact
  feature needs a privacy-routed destination, it is net-new config.

**Note:** there is no config property holding a *destination* support address as
a value the backend sends *to*. `replyTo` (support@) is set as the Reply-To
header on outbound mail; it is not currently used as a `to:` recipient. A contact
form that emails the support inbox would need a `to:` target — either reuse the
`replyTo` value or introduce a dedicated `oglasino.email.contact-to` property.

---

## 2. Suggestion table — is it generic, or category-specific?

### Surface

- Entity: `Suggestion` (`admin/entity/Suggestion.java`) extends `BaseEntity`.
  Columns: `userId` (Long, **nullable** — line 12), `suggestion` (String,
  `nullable=false` — lines 14–15), `suggestionType` (enum, `@Enumerated(STRING)`,
  no `nullable=false` — lines 17–18).
- Enum: `SuggestionType` (`admin/entity/SuggestionType.java`) — exactly two
  values: `CATEGORY_SUGGESTION`, `FEATURE_BUG_SUGGESTION` (lines 4–5).
- Table (`db/migration/V1__init_schema.sql`, suggestion block):
  ```sql
  CREATE TABLE public.suggestion (
      created_at timestamp(6) without time zone,
      id bigint NOT NULL,
      updated_at timestamp(6) without time zone,
      user_id bigint,
      suggestion character varying(255) NOT NULL,
      suggestion_type character varying(255),
      CONSTRAINT suggestion_suggestion_type_check CHECK (
        ((suggestion_type)::text = ANY ((ARRAY['CATEGORY_SUGGESTION'::character varying,
        'FEATURE_BUG_SUGGESTION'::character varying])::text[])))
  );
  ```
  `suggestion` is `varchar(255) NOT NULL`; `user_id` and `suggestion_type` are
  nullable; the CHECK constraint pins `suggestion_type` to the two enum values.
- Repository: `SuggestionRepository` (`admin/repository/SuggestionRepository.java`)
  — `JpaRepository` + `JpaSpecificationExecutor`, plus `deleteByUserId` (lines
  12–14, used by user-deletion cascade).
- DTOs:
  - `SuggestionRequestDTO` (`admin/dto/SuggestionRequestDTO.java`) — `suggestion`
    (`@NotBlank @Size(max=100)`, lines 9–11), `suggestionType` (`@NotNull`, line
    13). **See finding in §5 — these constraints are not enforced.**
  - `SuggestionDTO` (record, admin/dto) — `id, suggestionType, userId, suggestion,
    createdAt`.
  - `SuggestionsFilterRequestDTO` (admin/dto) — admin filter by `userId` /
    `suggestionType` via JPA `Specification`.
- Controller: `SuggestionController` (`admin/controller/SuggestionController.java`)
  - `POST /api/public/suggestion` (lines 22–29) — public submit. Calls
    `suggestionService.saveSuggestion(type, suggestion)`.
  - `POST /api/secure/admin/suggestion` (lines 31–50, `@PreAuthorize hasRole ADMIN`)
    — admin paged read.
- Service: `DefaultSuggestionService` (`admin/service/impl/DefaultSuggestionService.java`)
  - `saveSuggestion` (lines 20–30): derives `userId` from
    `currentUserService.getCurrentUserId().orElse(null)` (line 22) — **server-side,
    not from the client** — then saves type + userId + suggestion. This is the
    exact "logged-in → attributed; anonymous → null userId" pattern the contact
    feature wants (see §3, §6).

### Is it generic or category-specific?

It is **semi-generic, not category-only**. The name and one enum value say
"category suggestion," but the second enum value `FEATURE_BUG_SUGGESTION` already
makes it a two-purpose free-text user-message store. The schema carries no
category foreign key, no email column, no subject, no status/triage column — just
`{userId?, suggestion(text ≤255), suggestionType(enum), timestamps}`.

### Could a "contact request" row fit without semantic contortion?

**No — recommend against reusing this table.** Reasoning:

1. **Missing fields.** A contact request from an anonymous user needs a
   reply-to **email** and almost certainly a **subject/topic**. The suggestion
   table has neither, and `userId` is null for anonymous submitters — so an
   anonymous contact row would have no way to reach the sender back. Adding an
   email column to `suggestion` pollutes a moderation/admin table with PII and a
   reply channel it was never shaped for.
2. **Length.** `suggestion` is `varchar(255)` (and the DTO claims `≤100`). A
   contact/support message body wants more room than 255 chars.
3. **Enum semantics.** Routing contact requests through a third `SuggestionType`
   value works mechanically but overloads an admin-suggestions surface with a
   user-support concern, and the admin read endpoint
   (`/api/secure/admin/suggestion`) would start returning support messages mixed
   with category/bug suggestions.
4. **What it *does* model well — reuse the *pattern*, not the *table*.** The
   server-derived-`userId`-or-null attribution in `saveSuggestion` is exactly the
   right model for the contact feature. Copy that pattern into a dedicated
   `contact_request` entity/table carrying `{userId?, email, subject?, message,
   createdAt}` rather than bending `suggestion`.

---

## 3. Auth / identity

### How a controller reads the authenticated user today

`OglasinoAuthentication` (`security/auth/OglasinoAuthentication.java`) is the
principal placed in `SecurityContextHolder` by `FirebaseAuthFilter`. It carries:
`userId`, `firebaseUid`, `baseSiteId`, `preferredLanguage`, `authorities`, and a
stashed verified `FirebaseToken` in `credentials` (lines 12–24). **It does NOT
carry email.**

`AuthenticatedUserDTO` (`security/auth/AuthenticatedUserDTO.java`, the Redis-cached
snapshot, lines 20–29) carries `userId, firebaseUid, disabled, userRole,
subscriptionType, subscriptionActive, baseSiteId, preferredLanguage,
deletionStatus` — **also no email.**

Controllers read identity through `CurrentUserService`
(`security/service/impl/DefaultCurrentUserService.java`):
- `getCurrentUserId()` → `Optional<Long>` (lines 31–38) — empty when anonymous.
- `getCurrentUserIdStrict()` (lines 49–52) — `orElseThrow`.
- `getCurrentUserStrict()` (lines 54–57) → loads the full `User` entity via
  `userRepository.findById(userId)`.
- `isCurrentUserAdmin()`, `getCurrentUserBaseSiteId()`.

### Is the authenticated user's EMAIL available server-side?

**Not directly from the auth principal — only `userId`.** Email is available
server-side via the DB:
- `getCurrentUserStrict()` loads the `User`, and `User.getEmail()` is the address.
- Or `UserRepository.findByEmail` / `UserService.getUserByEmail`
  (`repository/UserRepository.java:20`, `service/UserService.java:23`) exist for
  email lookups.
- The stashed `FirebaseToken` in `OglasinoAuthentication.credentials` also exposes
  `decoded.getEmail()`, but that is a token claim, not the canonical store. The
  DB-loaded `User.email` is the authoritative, trust-boundary-correct source.

**Implication for contact form:** for a logged-in submitter, do **not** trust a
client-supplied email — derive it from `userId` →
`userRepository.findById(...).getEmail()` (or `getCurrentUserStrict()`).

### Public vs secure split, and how a both-modes endpoint is wired

Matcher (`security/config/SecurityConfig.java`, `authorizeHttpRequests`,
lines 66–100):
- `/api/public/**`, `/api/auth/**`, `/internal/**` → `permitAll()` (lines 92–93)
- `/api/secure/**` → `authenticated()` (lines 95–96)
- `anyRequest().authenticated()` — default-deny (lines 99–100)

**How an endpoint serves BOTH authenticated and anonymous callers:** put it under
`/api/public/**` (permitAll), and rely on `FirebaseAuthFilter` to *opportunistically*
populate the security context when a token is present:

- `FirebaseAuthFilter` (`security/filter/FirebaseAuthFilter.java`) runs on every
  non-OPTIONS request (lines 51–63). If a token is present and valid, it sets the
  `OglasinoAuthentication` regardless of whether the path is public or secure
  (lines 108–120).
- If **no token** is present, it falls straight through anonymously (the `if` at
  line 63 is skipped).
- If a **bad token** hits a permitAll path (`/api/public`, `/api/auth`,
  `/internal`), the 401 is suppressed and the request proceeds anonymously
  (lines 146–153).

So a controller under `/api/public/**` can call
`currentUserService.getCurrentUserId()` and get the userId when logged in or
`Optional.empty()` when anonymous — **with no extra wiring**. This is precisely
what `POST /api/public/suggestion` →
`DefaultSuggestionService.saveSuggestion` already does (§2). The contact endpoint
should follow the same model: live under `/api/public/`, derive identity if
present, fall back to the client-supplied email if anonymous.

---

## 4. Rate limiting / abuse

### `RateLimitFilter` keying

`RateLimitFilter` (`security/filter/RateLimitFilter.java`), registered in the
Spring Security chain *after* `FirebaseAuthFilter` (SecurityConfig.java line 103),
so the context is populated when it runs.

Keying — `callerKey(request)` (lines 152–173), in priority order:
1. `user:<userId>` when an `OglasinoAuthentication` is present (lines 154–155).
2. else `device:<X-Device-Id>` header when present (lines 161–164) — mobile
   per-installation buckets; client-supplied/spoofable but only resets the
   attacker's own bucket.
3. else `ip:<CF-Connecting-IP>` (Cloudflare-set), falling back to
   `request.getRemoteAddr()` when that header is absent (lines 166–172).

Bucket key: `"rl:" + caller + ":" + category` (line 64), Bucket4j-over-Redis
(`RateLimitConfig.java`). 429 body is the unified error envelope with code
`RATE_LIMITED` (lines 34–39), plus `Retry-After` (line 87).

### How a new public endpoint opts into throttling

Path-based routing in `categorize(path, method)` (lines 101–150). To throttle a
new endpoint you add a branch here. **The public suggestion endpoint already opts
in** — lines 132–138:
```java
if ("POST".equals(method)) {
  ...
  if (path.equals("/api/public/verify-recaptcha") || path.equals("/api/public/suggestion")) {
    return RateLimitCategory.PUBLIC_WRITE;
  }
}
```
`PUBLIC_WRITE` = 20 requests / 1 minute (`RateLimitCategory.java` line 16). A new
`POST /api/public/contact` would add itself to that `||` chain (or get its own
category if a tighter cap is wanted). For an anonymous spam target a tighter
dedicated category is worth considering, since the IP-keyed fallback is the only
key for anonymous callers and 20/min/IP is generous for a contact form.

`RequestThrottleFilter` (`security/filter/RequestThrottleFilter.java`) is a
*separate* concern — global DB-overload shedding (503), keyed on nothing
client-side; not a per-caller abuse control. Not relevant to contact throttling.

### Existing per-email throttle pattern — directly applicable

`AuthController` (`controller/AuthController.java`) implements the
60s-cooldown + N-per-day pattern the brief references, twice
(resend-verification and password-reset), using plain `StringRedisTemplate`
primitives — **no Bucket4j**:

- Constants (lines 72–86): `RESEND_COOLDOWN = 60s`, `RESEND_DAILY_LIMIT = 4`,
  `RESEND_DAILY_WINDOW = 24h`; password-reset mirror on separate key prefixes.
- `resendVerification` (lines 203–268) and `requestPasswordReset` (lines 297–358)
  share the mechanism:
  - normalize email to lowercase (lines 211, 305);
  - **60s gap** via SET-NX-EX: `redisTemplate.opsForValue().setIfAbsent(cooldownKey,
    "1", COOLDOWN)` — on miss, return 429 with `retryAfterSeconds` from the key's
    TTL (lines 218–230, 312–322);
  - **daily cap** via `INCR` + 24h TTL stamped on first use; over the limit,
    release the cooldown slot and return a distinct 429 code (lines 235–246,
    327–338);
  - on a real send failure, **release both limits** so a transient SMTP error
    doesn't lock the user out (lines 259–265, 349–353);
  - the limits are consumed identically on the no-leak no-op so timing can't probe
    account existence (lines 248–254).

**Applicability to contact form:** this is the right tool for an anonymous
contact form keyed on the submitter's email — it caps per-address volume on top
of the per-IP `RateLimitFilter` URL bucket, protecting the shared Brevo daily
budget. The pattern is copy-ready; the contact endpoint would key on the
client-supplied (anonymous) or DB-derived (logged-in) email. Both layers stack:
URL/IP bucket (RateLimitFilter) + per-email Redis cooldown (AuthController-style).

---

## 5. Error contract

Confirmed (`exception/GlobalExceptionHandler.java`, `@RestControllerAdvice`,
applies to all non-image controllers — javadoc lines 24–30):

- **Envelope** `{errors:[{field, code, translationKey}]}` via `ProductErrorResponse`.
  `field` is null for object-level violations (e.g. AuthController.errorResponse
  lines 419–425 builds `FieldError(null, code, translationKey)`).
- **400 Jakarta** — `MethodArgumentNotValidException` (lines 83–96) and
  `HandlerMethodValidationException` (lines 108–121) → `ResponseEntity.badRequest()`.
  Both map each constraint message (an error-code name) to its translation key.
- **422 business** — `ProductValidationException` (lines 129–134), with a
  severity ladder `403 > 422 > 400` (lines 253–267). `ReportValidationException` /
  `AppVersionValidationException` mirror this. A contact feature wanting business
  rules (e.g. message-too-long-after-trim, banned content) would throw a
  validation exception carrying its own error-code enum and get 422.
- **403** — `AuthorizationDeniedException` / `AccessDeniedException` (lines 55–64),
  emitting `NOT_AUTHENTICATED` vs `ACCESS_DENIED`.
- **429** — written directly by `RateLimitFilter` (pre-DispatcherServlet),
  `RATE_LIMITED` (RateLimitFilter.java lines 34–39).
- **500** — `INTERNAL_ERROR` catch-all (lines 205–213); never a validation path.

**Finding — public suggestion endpoint does not enforce its DTO constraints
(directly relevant to how the contact endpoint must be wired).**
`SuggestionController.suggestCategory` declares `@RequestBody SuggestionRequestDTO`
with **no `@Valid`** (SuggestionController.java line 24). The
`@NotBlank @Size(max=100)` / `@NotNull` on `SuggestionRequestDTO` (lines 9–13) are
therefore **never triggered** — no 400 is ever produced for a blank/oversized
suggestion or a null type. A blank/null `suggestion` would instead reach
`save(...)` and hit the `NOT NULL` DB constraint → 500 (an error-contract
violation per conventions Part 7). For the contact endpoint to emit proper 400s,
the handler **must** annotate the body `@Valid` — do not copy the suggestion
controller's omission. Flagged as an adjacent observation in the session summary
(severity medium).

---

## 6. Trust boundary (Part 11)

For the proposed split (logged-in → derive identity from auth; anonymous → trust
form email):

**Server-derivable (must NOT be trusted from the request when logged in):**
- `userId` — `currentUserService.getCurrentUserId()` (DefaultCurrentUserService
  lines 31–38). Authoritative, from `SecurityContextHolder`.
- The logged-in user's **email** — load from DB via `userId`
  (`getCurrentUserStrict().getEmail()` or `userRepository.findByEmail`). Email is
  **not** on `OglasinoAuthentication` / `AuthenticatedUserDTO`, so it is a DB read,
  not a context read — but it is still fully server-derived. (The stashed
  `FirebaseToken.getEmail()` is also available but the DB value is canonical.)
- `baseSiteId`, `preferredLanguage` — on the auth principal (for site attribution
  / email language), server-derived.

**Must come from the request (only the anonymous case):**
- The reply-to **email** — for an anonymous submitter there is no server identity,
  so the form email is the only source. It is **content**, not an
  authorization/moderation/state-transition input, so trusting it does not violate
  Part 11. It should be format-validated (Jakarta `@Email` + `@NotBlank`) and
  treated as untrusted display/reply data.
- The **message body** (and subject/topic if added) — always client content,
  validated for length/required, moderated as needed.

**Spoofing flag (the one thing to get right):** the split must be driven by the
**server's** view of authentication, not a client flag. Concretely:

- If a request **carries a valid token** (so `getCurrentUserId()` is present),
  attribute the contact request to that `userId` and use the **DB-derived email**.
  Do **not** also accept a client-supplied email field on this path — otherwise a
  logged-in user could submit a contact request attributed to their account while
  putting *someone else's* address in the reply-to / "from" field, spoofing
  attribution (the row says user A, the reply goes to victim B; or an outbound
  "from this user" email carries a forged address). Ignore any client email when
  `userId` is present.
- Only when `getCurrentUserId()` is **empty** (genuinely anonymous) should the
  client email be used — and then there is no account to spoof, because the row is
  attributed to no user (`userId = null`, exactly as
  `DefaultSuggestionService.saveSuggestion` stores it, line 22).
- Do not let the client choose the mode (e.g. an `anonymous: true` body flag or a
  client-sent `userId`). The presence/absence of a server-verified principal is
  the only switch. This mirrors the existing suggestion endpoint, which never
  reads identity from the body.

There is no client-supplied "before/previous" value in this feature, so the
specific `oldName` anti-pattern from Part 11 does not arise here.

---

## Summary of cross-cutting findings

1. **Reuse the email stack as-is.** `EmailService.sendHtml` + `EmailLayout.wrap` +
   a `TransactionalEmailSender`-style assembler cover the outbound support email.
   No new send primitive.
2. **Do NOT reuse the `suggestion` table.** Reuse its server-derived-userId
   *pattern*; create a dedicated `contact_request` table with `email`, a larger
   message column, and (likely) subject. (Pre-prod V1 schema fold, conventions
   Part 12 — edit `V1__init_schema.sql` in place.)
3. **Both-modes endpoint is already a solved pattern.** Put it under
   `/api/public/`; `FirebaseAuthFilter` opportunistically populates the principal,
   so `getCurrentUserId()` yields userId-or-empty with no extra wiring.
4. **Two stacking abuse controls available:** the URL/IP `RateLimitFilter` bucket
   (add the path to `categorize`, `PUBLIC_WRITE` or a tighter dedicated category)
   and the `AuthController` per-email 60s+N/day Redis cooldown (copy-ready).
5. **Email of a logged-in user is a DB read, not an auth-context field.** Derive
   it server-side; never trust a client email when a principal is present.
6. **Annotate the request body `@Valid`** — the existing public suggestion
   controller omits it (SuggestionController.java line 24), so its DTO constraints
   are dead and a malformed body 500s instead of 400s. The contact controller must
   not repeat that.
</content>
</invoke>
