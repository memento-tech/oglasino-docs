# Audit: External-Client Timeout Hardening — Fix Shape

**Repo:** `oglasino-backend` · **Branch:** `dev` · **Type:** read-only fix-shape analysis
**Date:** 2026-06-04

Scope: for the five confirmed timeout-less external clients (reCAPTCHA, SMTP, OpenAI, R2/S3, Cloudflare KV), determine construction shape, the correct fix mechanism, sane timeout values tied to the slowest legitimate operation, and restructure blast radius. The findings' *existence* is already confirmed; this is HOW to fix, not WHETHER.

---

## Lead table

| Client | Construct shape | Fix mechanism | Connect | Read / op | Slow-op risk |
| --- | --- | --- | --- | --- | --- |
| reCAPTCHA | field-init `new RestTemplate()` (`DefaultReCaptchaService:13`) | inject the existing bounded `restTemplate` bean | 3s | 3s | none — quick verify POST |
| SMTP | auto-configured `JavaMailSender` from `spring.mail.*` (no `@Bean`) | add `mail.smtp.*timeout` yaml keys (×3 envs) | 5s | 5s read / 5s write | low — STARTTLS relay; per-op, not total |
| OpenAI | local `new` per call (`HttpClients.createDefault()`, `DefaultOpenAIService:42`) | configure the Apache client with `RequestConfig` (connect + response timeout) | 5s | **60s response** | **HIGH** — gpt-5-nano LLM completion; a 3s read would break it |
| R2 / S3 | `@Bean` (`R2Config:18`, `UrlConnectionHttpClient.builder().build()`) | add `connectionTimeout` + `socketTimeout` on the `UrlConnectionHttpClient` builder | 5s | 10s socket | low — **deletes/lists/head only, NO uploads** |
| Cloudflare KV | field-init `new RestTemplate()` (`DefaultCloudflareKvService:28`) — legacy client coexisting with an already-injected bounded bean (`:37`) | drop the field-init client, repoint legacy methods at the injected bounded bean (keep the two error-policy helpers distinct) | 3s | 3s | none — quick toggle/read |

---

## 1. The existing bounded pattern

There is **exactly one** bounded HTTP client bean in the codebase:

`ApplicationConfig.java:57-63`
```java
@Bean
public RestTemplate restTemplate() {
  var factory = new SimpleClientHttpRequestFactory();
  factory.setConnectTimeout(3000);
  factory.setReadTimeout(3000);
  return new RestTemplate(factory);
}
```
3s connect / 3s read, via `SimpleClientHttpRequestFactory`. This is the single bounded HTTP client pattern — the `grep` for `setConnectTimeout|setReadTimeout|connectionTimeout|socketTimeout|apiCallTimeout|RequestConfig|Duration.of*` surfaced no other bounded HTTP client (the other `Duration.of*` hits are Redis TTLs, rate-limit bandwidths, and resend cooldowns — not HTTP clients).

**Places that already consume the bounded bean correctly (the pattern to copy):**

- `DefaultBaseCurrencyService.java:38` — `@Autowired private RestTemplate restTemplate;` (field injection)
- `DefaultWebRevalidationService.java:23` — `@Autowired private RestTemplate restTemplate;` (field injection)
- `DefaultExpoPushService.java:36,39` — constructor injection (`final RestTemplate restTemplate`)
- `DefaultCloudflareKvService.java:37` — `@Autowired private RestTemplate boundedRestTemplate;` (injected by type; used only on the DB-overload auto-trip path)
- `TelegramAlertService.java:24` — javadoc explicitly records the convention: *"on … Never a no-timeout `new RestTemplate()`."*

Because there is exactly one `RestTemplate` bean, by-type `@Autowired` is unambiguous everywhere — no `@Qualifier` is needed for the reCAPTCHA / KV fixes.

The R2/S3 client and the OpenAI Apache client are separate stacks (AWS SDK `UrlConnectionHttpClient`, Apache HttpClient 5) and cannot reuse the `RestTemplate` bean — they need their own builder-level timeout config (below). SMTP is the Spring-Boot-auto-configured `JavaMailSender` and is bounded via yaml, not a bean edit.

---

## 2 + 3 + 4. Per-client: construction, mechanism, slow-op, blast radius

### reCAPTCHA — `DefaultReCaptchaService`

- **Construction:** field-init. `DefaultReCaptchaService.java:13` — `private final RestTemplate rest = new RestTemplate();`. Class is a `@Component` using **field injection** (`@Value` fields for `secret`/`verifyUrl`, no constructor).
- **Minimal fix:** inject the existing bounded `restTemplate` bean. Match the established field-injection pattern verbatim (`DefaultBaseCurrencyService` / `DefaultWebRevalidationService`):
  ```java
  @Autowired private RestTemplate restTemplate;   // replaces `new RestTemplate()`
  ```
  then rename the one usage at `:25` (`rest.postForObject`). No `@Qualifier` (single bean).
- **Slow-op:** none. The one call is a single `POST` to Google's siteverify (`verify(token)`, `:22-30`). 3s/3s is correct.
- **Blast radius:** trivial. One field + one usage rename, no constructor change required — the class already uses field injection, so injecting the bean matches the surrounding style. Optionally convert to constructor injection, but that is a style change beyond the minimal fix.

### SMTP — `JavaMailSender` (`DefaultEmailService`)

- **Construction:** no `@Bean` — Spring Boot auto-configures `JavaMailSender` from the `spring.mail.*` block. `DefaultEmailService.java:24` injects it (`@Autowired private JavaMailSender mailSender;`). The mail block exists in all three env yamls (`application-dev.yaml:65`, `application-stage.yaml:82`, `application-prod.yaml:79`) with `properties.mail.smtp.{auth,starttls}` only — **no timeout keys**.
- **Minimal fix:** config-only. Add under each env's `spring.mail.properties.mail.smtp`:
  ```yaml
  connectiontimeout: 5000   # ms — TCP connect to smtp-relay.brevo.com
  timeout: 5000             # ms — socket read timeout (per I/O operation)
  writetimeout: 5000        # ms — socket write timeout (per I/O operation)
  ```
  (JavaMail property names are literally `mail.smtp.connectiontimeout` / `mail.smtp.timeout` / `mail.smtp.writetimeout`.) Apply to **all three** yamls — dev/stage share the Brevo stage key, prod has its own; all relay through `smtp-relay.brevo.com:587` STARTTLS.
- **Slow-op / sync-vs-async (decision-critical confirmation):** the **same** `JavaMailSender` bean is used on **both** paths, so a yaml timeout bounds both:
  - **Synchronous, on the request thread** — these surface `EMAIL_SEND_FAILED` (HTTP 502) to the caller, so a hung SMTP socket directly stalls a user request:
    - `AuthController.resendVerification` → `verificationEmailSender.send(...)` (`:263`, propagates → `:270` 502)
    - `AuthController` password-reset → `passwordResetService.process(email)` (`:349`, `PasswordResetSendException` → `:358` 502)
    - `AuthController` reblock → `reblockEmailSender.send(...)` (`:405`)
  - **`@Async`** (off-thread, failure must not disturb the triggering action): `UserLifecycleEmailEventListener` (`:55,68,76,84,92`), `RegistrationEmailEventListener` (`:52`, welcome), `ReviewApprovedEmailEventListener` (`:49`), `AlertService` (`:75`, DB-overload operator alert).
  - **No legitimate reason for a mail send to need >5s.** These timeouts are JavaMail **per-socket-operation** values (not a total-send budget), and the Brevo STARTTLS conversation is a sequence of fast round-trips (EHLO → STARTTLS → AUTH → MAIL FROM → RCPT → DATA → body → QUIT). 5s per operation is generous headroom; the bodies are small transactional HTML emails. If a relay hiccup under load is a concern, 10s is a safe upper bound — but 5s is the recommendation.
- **Blast radius:** none structural — three yaml edits, no Java change.

### OpenAI — `DefaultOpenAIService` ⚠️ most likely to be set wrong

- **Construction:** **local `new` per call.** `DefaultOpenAIService.java:42` — `try (CloseableHttpClient client = HttpClients.createDefault())`. A fresh default Apache HttpClient 5 client is built (and closed) on every `sendRequest`. No `RequestConfig`, so no connect/response timeout.
- **Minimal fix:** configure the Apache client with a `RequestConfig` carrying a **short connect** and a **long response** timeout. Either set it on the client builder or per-request:
  ```java
  RequestConfig cfg = RequestConfig.custom()
      .setConnectTimeout(Timeout.ofSeconds(5))
      .setResponseTimeout(Timeout.ofSeconds(60))   // LLM completion budget
      .build();
  // HttpClients.custom().setDefaultRequestConfig(cfg).build()  — or  post.setConfig(cfg)
  ```
  Cannot use the `RestTemplate` bean (different stack — Apache HttpClient 5).
- **Slow-op (the one that breaks if set short):** this is an **LLM completion**, not a quick API call. What's called:
  - Endpoint: `${app.openai.api-url}` (the OpenAI Responses API), model **`gpt-5-nano`** (`DEFAULT_MODEL`, `:25`; overridable via `openai.default_model` config), `reasoning.effort` default `MINIMAL`.
  - Two operations via `DefaultOpenAIFacade`: (a) **product description suggestion** — `getOpenAIDescriptionSuggestion(productName, lang)` (`:37`), single-language generation, called from `OpenAiController` / `DashboardProductController`; (b) **translation** — `translateTextWithPrompt(...)` (`:31`) generating all languages in one prompt (used by `DefaultProductService`, `DefaultReviewTranslationsService`, `DefaultUserTranslationsService`).
  - gpt-5-series reasoning completions legitimately take **tens of seconds**. **A 3s read timeout would break every AI generation.** Connect must stay short (TCP+TLS to `api.openai.com` is sub-second), but the **response/read budget must be tens of seconds**. Recommend **connect 5s, response 60s.** (60s comfortably covers minimal-effort short generations with headroom; if generations are observed running longer, this is the one value worth promoting to config — but a single 60s constant is fine today per Part 4a.)
- **Blast radius:** contained to `sendRequest` — the per-call `try (… createDefault())` becomes a per-call `try (… custom with RequestConfig)`. No caller change. **Optional** improvement (not required for the fix): promote the configured client to a reused `@Bean` / field instead of rebuilding per call — but that is a perf/structure change beyond the timeout fix; the minimal fix just adds `RequestConfig` to the existing per-call construction.

### R2 / S3 — `R2Config` / `DefaultR2Service`

- **Construction:** `@Bean`. `R2Config.java:18-31` builds the `S3Client` with `.httpClient(UrlConnectionHttpClient.builder().build())` (`:29`) — no timeout override on the http client and no `overrideConfiguration` on the S3 client.
- **Minimal fix:** add timeouts to the `UrlConnectionHttpClient` builder:
  ```java
  .httpClient(UrlConnectionHttpClient.builder()
      .connectionTimeout(Duration.ofSeconds(5))
      .socketTimeout(Duration.ofSeconds(10))
      .build())
  ```
  Optionally also bound total call time via `.overrideConfiguration(c -> c.apiCallAttemptTimeout(Duration.ofSeconds(10)).apiCallTimeout(Duration.ofSeconds(30)))` on the `S3Client.builder()`. The http-client-level connect+socket timeouts are the essential fix; the `overrideConfiguration` is belt-and-suspenders.
- **Slow-op (decision-critical confirmation):** **the client does NOT do uploads.** Per Part 8 (direct-to-R2), image bytes never proxy through the backend. Every operation through `r2Client` (`DefaultR2Service`) is a **metadata operation**:
  - `deleteObject` (`:39`), `deleteObjects` (batched ≤1000 keys/request, `:74`), `listObjectsV2` (paginated, `:106/136/159`), `headObject` (`:181`).
  - All are single fast HTTP round-trips (the bulk delete and list paginate over multiple requests, but each request is small). **No large-upload budget is needed** — so a moderate 10s socket timeout cannot cap a legitimate upload (there are none). Connect 5s / socket 10s is safe; even the 1000-key bulk delete response is small.
- **Blast radius:** contained to the `R2Config` bean — builder-chain edit only, no caller change. `Duration` import to add.

### Cloudflare KV — `DefaultCloudflareKvService`

- **Construction:** **two clients coexist by design.** `DefaultCloudflareKvService.java:28` — legacy `private RestTemplate restTemplate = new RestTemplate();` (field-init, **no timeout**) used by `toggleMaintenance` / `isMaintenanceActive` / `readMaintenanceFlag` (`:121`) / `setMaintenanceCloudflareValue` (`:139`). Separately, `:37` `@Autowired private RestTemplate boundedRestTemplate;` (the bounded bean) is already wired and used only by the DB-overload auto-trip path (`boundedReadMaintenanceFlag` `:101`, `boundedSetMaintenanceValue` `:114`). This is the `issues.md` 2026-06-03 finding.
- **Minimal fix:** delete the field-init `restTemplate` (`:28`) and repoint the legacy methods at the injected bounded bean (rename `boundedRestTemplate` → `restTemplate` once it is the only client). 3s/3s is correct for KV — quick toggle/read.
- **Slow-op:** none — single GET/PUT against the Cloudflare KV REST API. 3s/3s fine.
- **Blast radius / nuance ⚠️:** the class's in-code comment (`:30-36`) claims that once the legacy client is bounded, *"the bounded helpers below collapse into the legacy ones."* **A naïve full collapse would be wrong** — the two read helpers differ in **error policy, not just client**:
  - `readMaintenanceFlag` (`:121`) swallows **every** non-404 failure (timeout, 5xx, auth) to `false`/"off" — graceful degradation matching the worker's absent=up semantics.
  - `boundedReadMaintenanceFlag` (`:101`) swallows **only** 404 and **lets timeout/5xx/auth propagate** to `assertMaintenanceOn`'s catch, so the auto-trip records `FAILED` rather than mistaking an unknown KV state for "off."
  These two policies must **both survive**. The fix is: share the single bounded client (drop the duplicate field), but **keep the two helper methods distinct** because the auto-trip path requires propagate-on-failure while the admin/read path requires degrade-to-off. The set helpers (`setMaintenanceCloudflareValue` re-throws; `boundedSetMaintenanceValue` lets the caller's try/catch handle it) can converge more readily, but the read pair cannot. Recommend the implementation brief say "share the bounded client; do **not** collapse the read helpers" and correct the stale in-code comment in the same edit.

---

## Effort grouping (for the implementation brief)

**A. Config-only (yaml, no Java):**
- **SMTP** — add `mail.smtp.{connectiontimeout,timeout,writetimeout}: 5000` under `spring.mail.properties.mail.smtp` in all three env yamls.

**B. One-/few-line bean swap (use the existing bounded `RestTemplate`):**
- **reCAPTCHA** — replace field-init `new RestTemplate()` with `@Autowired private RestTemplate restTemplate;` + rename the one usage. (No constructor change; matches `DefaultBaseCurrencyService`.)
- **Cloudflare KV** — delete the legacy field-init client, repoint the four legacy methods at the already-injected bounded bean, **keep the two read helpers distinct**, correct the stale `:30-36` comment.

**C. Own-client builder config (separate HTTP stacks, no `RestTemplate` reuse):**
- **OpenAI** — add `RequestConfig` (connect 5s / **response 60s**) to the per-call Apache client. ⚠️ the value most likely to be set wrong — must keep a long response budget for LLM completions.
- **R2/S3** — add `connectionTimeout` (5s) + `socketTimeout` (10s) to the `UrlConnectionHttpClient` builder in `R2Config` (optionally `overrideConfiguration` apiCall timeouts). Safe because the client does deletes/lists/head only — no uploads.

**D. Construction restructure (constructor changes / new beans):** **none required.** reCAPTCHA stays on field injection; OpenAI's per-call client is configured in place (promoting it to a reused bean is an optional, separate improvement). No client needs a constructor signature change for the minimal timeout fix.
