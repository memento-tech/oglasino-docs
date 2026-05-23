# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-22
**Task:** Brevo SMTP integration (initial wiring + smoke endpoint)

## Implemented

- Added per-environment Brevo SMTP config to `application-dev.yaml`, `application-stage.yaml`, and `application-prod.yaml` — `spring.mail.host/port/username/password` plus `mail.smtp.auth=true`, `mail.smtp.starttls.enable=true`, `mail.smtp.starttls.required=true`, all driven by `OGLASINO_EMAIL_SMTP_*` env vars with no literal defaults.
- Added `oglasino.email.*` envelope block (from-address, from-name, reply-to) to all three per-env YAMLs, also fully env-var driven.
- Introduced `EmailProperties` (`@ConfigurationProperties("oglasino.email")`) carrying From / From-name / Reply-To with no baked-in defaults — missing env var fails fast at startup.
- Introduced `EmailService` interface and `DefaultEmailService` impl with the single `sendPlainText(to, subject, body)` method. Uses `JavaMailSender.createMimeMessage()` + `MimeMessageHelper` to set From + From-name + Reply-To from `EmailProperties` and the caller-supplied To/Subject/plain-text body; `MessagingException`/`UnsupportedEncodingException` re-thrown as `MailPreparationException` (a `MailException` subtype) so the smoke-endpoint catch reaches them too.
- Added throwaway `POST /api/secure/admin/email/test` controller (`EmailTestController` in the existing `admin/controller` package) gated by `@PreAuthorize("hasRole('ADMIN')")`. Body `{to, subject, body}` validated with `@NotBlank + @Email`; returns 200 on enqueue, 500 with `{exception, message}` body on `MailException`. Controller class-level Javadoc explicitly notes it is removed by the next email-using-feature brief.
- Added `DefaultEmailServiceTest` — spied `JavaMailSenderImpl` with `send(MimeMessage)` stubbed to no-op + `ArgumentCaptor<MimeMessage>` to assert From address, From-name, Reply-To, To, Subject, and plain-text Body on the captured `MimeMessage`.

## Files touched

- pom.xml — read-only; `spring-boot-starter-mail` was already present (lines 67-70) but had zero consumers. See "Obsoleted by this session."
- src/main/resources/application-dev.yaml (+25 / -1)
- src/main/resources/application-stage.yaml (+25 / -1)
- src/main/resources/application-prod.yaml (+25 / -1)
- src/main/java/com/memento/tech/oglasino/properties/EmailProperties.java (new, +52 / -0)
- src/main/java/com/memento/tech/oglasino/service/EmailService.java (new, +22 / -0)
- src/main/java/com/memento/tech/oglasino/service/impl/DefaultEmailService.java (new, +34 / -0)
- src/main/java/com/memento/tech/oglasino/admin/controller/EmailTestController.java (new, +52 / -0)
- src/test/java/com/memento/tech/oglasino/service/impl/DefaultEmailServiceTest.java (new, +73 / -0)

## Tests

- Ran: `./mvnw spotless:check` and `./mvnw test`
- Result: spotless clean; **549 passed, 0 failed, 0 skipped** (prior baseline 548 + 1 new test).
- New tests added: `DefaultEmailServiceTest.sendPlainText_stampsEnvelopeAndContent`.

## Cleanup performed

- None needed. No commented-out code, no `System.out.println`, no ad-hoc logging, no unused imports, no `TODO`/`FIXME`. The class-level Javadoc on `EmailTestController` documents its throwaway status as prose, not as a `TODO` comment, so no entry is needed in "Known gaps / TODOs" for it — but it is listed there for the next-brief deletion hook per the brief's DoD.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** draft entry below in "For Mastermind" titled "Brevo SMTP integration shipped; pattern for transactional email send." Docs/QA applies.
- **state.md:** no change. This is infra wiring, not a feature with a `features/<slug>.md` spec.
- **issues.md:** no change.
- **`oglasino-docs/infra/overview/secret-inventory.md`:** draft addition below in "For Mastermind" — seven new `OGLASINO_EMAIL_*` rows. Docs/QA applies.

## Obsoleted by this session

- `spring-boot-starter-mail` in `pom.xml` (lines 67-70) had been a load-bearing-to-nothing dependency — present in the project, zero consumers (grep across `src/` for `JavaMailSender`, `MimeMessage`, `SimpleMailMessage`, `spring.mail`, `smtp`, `brevo`, `sendinblue` all returned nothing email-sending-related). This session wires up the first consumers, so the previously-orphan starter is now load-bearing. Nothing was deleted — flagged in "For Mastermind" per the brief's "Out of scope, but flag if you notice" item on a previously-abandoned SMTP provider.

## Conventions check

- **Part 4 (cleanliness):** confirmed.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** one finding flagged in "For Mastermind" (the previously-orphan `spring-boot-starter-mail`). Nothing else surfaced in files touched.
- **Part 6 (translations):** N/A — brief explicitly excludes `BACKEND_TRANSLATIONS` integration this session.
- **Part 7 (error contract):** the throwaway smoke endpoint deliberately deviates from the conventions Part 7 codes-only contract — it returns `{exception, message}` JSON with 500 on `MailException`. This is an explicit brief-level override scoped to the throwaway endpoint (the brief says: "Returns 200 on enqueue success, 500 with the Spring mail exception class + message on failure"). The endpoint is removed by the next email-using-feature brief, so the contract deviation is bounded to the smoke-endpoint lifetime. Permanent feature emails will not introduce this shape.
- **Part 11 (trust boundaries):** confirmed. The smoke endpoint's `to`/`subject`/`body` are caller-controlled; the From address, From-name, and Reply-To are server-derived from `EmailProperties` (never client-supplied). Endpoint is `@PreAuthorize("hasRole('ADMIN')")`-gated. `EmailService.sendPlainText` itself is internal — only backend callers reach it; to/subject/body are caller-controlled but trusted because callers are server-side.
- **Other parts touched:** Part 4a structured evidence below.

## Known gaps / TODOs

- **`EmailTestController` and its `EmailTestRequest` record are throwaway and must be deleted by the next brief** that wires real feature email content (User Deletion confirmations, password reset, ban notifications, etc.). The class-level Javadoc states this explicitly; this section is the second pointer per the brief's DoD.
- HTML email, multi-part messages, templated email, async/queued sending, retry logic, dead-letter handling, bounce-handling, suppression-list integration, Brevo webhooks, per-target validation, `BACKEND_TRANSLATIONS` integration — all deliberately deferred per brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - `EmailProperties` — three values vary per environment (From address differs prod-vs-stage-vs-dev; From-name and Reply-To happen to be constant today but are envelope-level data that may move per-environment later, e.g. distinct Reply-To inboxes per locale). Config vs constant — textbook config case.
    - `EmailService` interface (one impl today, `DefaultEmailService`). Earns its place: (a) per-feature briefs that follow this one (User Deletion confirmations, password reset, ban notifications) all need `sendPlainText`-like methods and will mock the interface in their per-feature unit tests rather than reaching for Mockito on the concrete class — every existing service in this repo follows the same `Service`/`DefaultService` pattern (`UserService`/`DefaultUserService`, `WebRevalidationService`/`DefaultWebRevalidationService`, etc.). (b) Brevo's free-tier quota (300/day, shared with stage) is plausibly insufficient once real features ship — a second impl swap (e.g., Postmark or self-hosted Postal) is concrete near-term work, not hypothetical. Inlining would have to be undone in the very next brief.
    - `DefaultEmailService` — single concrete implementation; thin (`MimeMessage` construction + send). No retry, no async, no template engine — all out of scope.
    - `EmailTestController` — throwaway; flagged for deletion in "Known gaps / TODOs."
    - Controller-local `EmailTestRequest` record — DTO for `@Valid @RequestBody`, defined as a nested record inside the controller (no separate `dto/` file because it lives and dies with the throwaway controller).
  - **Considered and rejected:**
    - A `MailConfig` `@Configuration` class — Spring Boot's `MailSenderAutoConfiguration` builds `JavaMailSender` directly from `spring.mail.*` properties; zero bean wiring needed. Inlined into the YAML config per the brief's explicit guidance ("only if more than 3 lines of bean wiring; otherwise inline").
    - Inlining `DefaultEmailService` into the throwaway controller — would have to be undone in the next brief, so the cost is paid either way; cheaper to ship the long-lived shape now.
    - Per-target validation/allowlist on the smoke endpoint — brief explicitly rules this out for the throwaway surface.
    - HTML body / multi-part / template engine / retry wrapper / async queue — all out of scope per brief.
    - A separate `AdminEmailFacade` between controller and service — facade adds nothing for a single sendPlainText call; controller calls service directly, matching the `CacheAdminController → CacheManager` and `EsIndexerController → IndexerService` shape elsewhere in `admin/controller`.
  - **Simplified or removed:** nothing.

- **Adjacent observation (Part 4b) — previously-orphan `spring-boot-starter-mail`:** `pom.xml` lines 67-70 already declare `spring-boot-starter-mail` (no version pin — managed by the Spring Boot parent at 4.0.6). Grep for any user of `JavaMailSender`, `MimeMessage`, `SimpleMailMessage`, `spring.mail`, `smtp`, `brevo`, or `sendinblue` across `src/` returns zero email-sending-related matches. Either the starter was added speculatively and never wired, or a prior SMTP integration was wired and reverted leaving the starter behind. This brief makes the starter load-bearing, so no cleanup is needed — but Mastermind may want to record the prior fact-of-the-matter in a decisions entry, since the brief's "Out of scope, but flag if you notice" line asked about exactly this case.

- **Draft for `decisions.md` (Docs/QA to apply) — "Brevo SMTP integration shipped; pattern for transactional email send":**

  > ## 2026-05-22 — Brevo SMTP integration shipped; pattern for transactional email send
  >
  > Backend now sends outbound transactional email via Brevo's SMTP relay (`smtp-relay.brevo.com:587` STARTTLS) on the free tier. Wiring is identical across dev / stage / prod; the per-environment difference is which Brevo SMTP key is used (`oglasino-backend-prod` for prod; `oglasino-backend-stage` shared by stage and dev) and which Brevo-verified From address gets stamped (`noreply@oglasino.com` for prod; `noreply-stage@oglasino.com` for stage + dev). Reply-To is `support@oglasino.com` on every environment so user replies on `noreply-*` mail land in the monitored inbox.
  >
  > **The shape:**
  >
  > - `oglasino.email.*` `@ConfigurationProperties` carries the envelope (From / From-name / Reply-To). All values come from environment variables — no defaults baked in, missing variable fails fast at startup.
  > - `spring.mail.host/port/username/password` plus the `mail.smtp.auth=true` / `mail.smtp.starttls.enable=true` / `mail.smtp.starttls.required=true` block wires Spring Boot's `MailSenderAutoConfiguration` — no `MailConfig @Configuration` class needed.
  > - `EmailService.sendPlainText(to, subject, body)` is the only entry point this brief; per-feature briefs (User Deletion confirmations, password reset, ban notifications, etc.) extend the interface for their own needs.
  > - `DefaultEmailService` uses `JavaMailSender.createMimeMessage()` + `MimeMessageHelper` to build a proper-headered `MimeMessage`, sets From + From-name + Reply-To from `EmailProperties`, sends. UTF-8 throughout.
  > - The brief ships a throwaway smoke endpoint `POST /api/secure/admin/email/test` (admin-gated, returns 500-with-`{exception, message}`-body on `MailException`) so Igor can prove deliverability end-to-end. **The endpoint is deleted by the next email-using-feature brief.**
  >
  > **Trust boundary:** the From / From-name / Reply-To are server-derived from `EmailProperties` and never client-supplied. The smoke endpoint accepts `to`/`subject`/`body` from an authenticated admin but does not validate per-target (the throwaway lifetime + admin gate are the protection); permanent admin-broadcast surfaces would need their own trust-boundary work. `EmailService.sendPlainText` itself is internal — server-side callers only.
  >
  > **Free-tier quota:** Brevo free tier is 300 emails/day, shared across prod + stage (dev reuses stage's key). Per-feature briefs that ship high-volume transactional email need to either move Brevo to a paid tier or migrate to a different provider; the `EmailService` interface is the swap-out seam.
  >
  > **Alternatives considered.** A `MailConfig @Configuration` class — rejected, Spring Boot autoconfigures `JavaMailSender` from `spring.mail.*` properties; the YAML config plus `EmailProperties` is all that's needed. Inlining `DefaultEmailService` into the throwaway controller — rejected, would need to be undone the moment the next per-feature brief lands. Per-target validation / allowlist on the smoke endpoint — rejected per brief; the throwaway lifetime plus admin gate is the protection.

- **Draft for `oglasino-docs/infra/overview/secret-inventory.md` (Docs/QA to apply) — append seven rows to the inventory table:**

  | Secret Name                | Lives In                                                                  | Used By                                                                                       | Rotation Date | Notes                                                                                                                                                       |
  | -------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | OGLASINO_EMAIL_SMTP_HOST    | Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `JavaMailSender` autoconfig (`spring.mail.host`)                              | TBD           | Brevo SMTP relay host. Value is `smtp-relay.brevo.com`. Identical across all three environments; kept as an env var rather than a literal so a future provider swap edits env vars only. |
  | OGLASINO_EMAIL_SMTP_PORT    | Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `JavaMailSender` autoconfig (`spring.mail.port`)                              | TBD           | Value is `587` (STARTTLS).                                                                                                                                  |
  | OGLASINO_EMAIL_SMTP_USERNAME| Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `JavaMailSender` autoconfig (`spring.mail.username`)                          | TBD           | Brevo SMTP login. Format: `<numeric>@smtp-brevo.com`. Shared across all three environments — Brevo issues one login per account; per-env isolation is via the password (SMTP key), not the login. |
  | OGLASINO_EMAIL_SMTP_PASSWORD| Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `JavaMailSender` autoconfig (`spring.mail.password`)                          | TBD           | Brevo SMTP key. Per environment: prod uses Brevo SMTP key named `oglasino-backend-prod`; stage and dev share `oglasino-backend-stage`. Rotating one does not affect the other. |
  | OGLASINO_EMAIL_FROM_ADDRESS | Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `EmailProperties.fromAddress`                                                  | n/a           | Verified Brevo sender. Prod: `noreply@oglasino.com`. Stage + dev: `noreply-stage@oglasino.com`. Changing this string without first verifying the new address with Brevo causes all sends to fail. |
  | OGLASINO_EMAIL_FROM_NAME    | Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `EmailProperties.fromName`                                                     | n/a           | From display name. Value is `Oglasino` on every environment.                                                                                                |
  | OGLASINO_EMAIL_REPLY_TO     | Droplet `/etc/oglasino/.env` (prod, stage); local `.env` (dev)            | oglasino-backend `EmailProperties.replyTo`                                                      | n/a           | Reply-To header. Value is `support@oglasino.com` on every environment. Per-email overrides are allowed in code but the default applies when not set. |

- **Local dev verification path for Igor:** set the seven `OGLASINO_EMAIL_*` env vars in `.env` (use stage Brevo creds; `noreply-stage@oglasino.com` as the From address), start the backend, sign in as an admin user, `curl -X POST -H 'Content-Type: application/json' -H 'Authorization: Bearer <admin-firebase-id-token>' -d '{"to":"<your-inbox>","subject":"Brevo smoke","body":"Hello from local dev"}' http://localhost:8080/api/secure/admin/email/test` — should return 200 and the email should land From `noreply-stage@oglasino.com` Reply-To `support@oglasino.com`.

- **Closure gate:** every config-file edit required by this session is either drafted above for Docs/QA to apply (the `decisions.md` entry and the seven `secret-inventory.md` rows) or explicitly stated as "no change" (`conventions.md`, `state.md`, `issues.md`). No implicit dependency.
