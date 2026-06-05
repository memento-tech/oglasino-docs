# Impl: External-client timeout hardening

**Repo:** `oglasino-backend` · **Branch:** changes left uncommitted on `dev` (hard rule forbids `git checkout`; per the brief's fallback, Igor creates `external-client-timeouts` and commits the listed files). **Date:** 2026-06-04 · Build: `./mvnw spotless:check` clean, `./mvnw test` **935 passed / 0 failed**.

All five clients are now timeout-bounded with per-client failure logging. OpenAI also fires a best-effort Telegram alert. **Config decision changed mid-session at Igor's direction:** timeout VALUES do **not** all live in yaml. OpenAI timeouts moved to the DB `configuration` table (hot-tunable, no redeploy); R2 + SMTP stay in yaml as env-var-with-default (no DB hot-reload is architecturally available for them — see "Config placement" below).

---

## Config placement — why OpenAI is DB and R2/SMTP are not

The codebase already documents this exact boundary (`data-configuration.sql` rows 76–80 for `@Scheduled` cron, 194–197 for the DB-overload poller): **a value read at request/poll time can be DB config; a value read at bean-construction/startup time cannot**, because `DefaultConfigurationService` warms its cache on `ApplicationReadyEvent` — *after* all beans are built.

| Client | Timeout consumed at… | Placement | Hot-reload (no restart)? |
| --- | --- | --- | --- |
| OpenAI | per call, inside `sendRequest` | DB `configuration` table | ✅ yes (admin `ConfigController`) |
| R2/S3 | `S3Client` `@Bean` construction (startup) | yaml `${VAR:default}` | ❌ restart only (no rebuild) |
| SMTP | Spring Boot mail autoconfig (startup) | yaml `${VAR:default}` | ❌ restart only (no rebuild) |
| reCAPTCHA / KV | shared `restTemplate` bean (startup) | n/a — uses shared 3s/3s bean | n/a |

OpenAI config-in-DB is the existing pattern, not a new one: `openai.default_model` + both prompts already live in that table.

---

## Per-client changes

### 1. reCAPTCHA — `service/impl/DefaultReCaptchaService.java`
- Replaced field-init `private final RestTemplate rest = new RestTemplate();` (`:13`) with `@Autowired private RestTemplate restTemplate;` (the bounded 3s/3s `ApplicationConfig.restTemplate()` bean), matching `DefaultBaseCurrencyService`. Renamed the one usage (`rest.postForObject` → `restTemplate.postForObject`).
- Added a `Logger`; the previously-silent `catch (RestClientException)` now logs `log.warn("reCAPTCHA siteverify call failed, treating token as invalid", e)` before returning `false` (behavior unchanged: still degrades to invalid).
- No new config (shared bean).

### 2. SMTP — `application-{dev,stage,prod}.yaml`
Added under `spring.mail.properties.mail.smtp` in all three env files (env-var-with-default, matching the repo's existing `${VAR:default}` convention):
```yaml
connectiontimeout: ${MAIL_SMTP_CONNECT_TIMEOUT_MS:5000}
timeout: ${MAIL_SMTP_READ_TIMEOUT_MS:5000}
writetimeout: ${MAIL_SMTP_WRITE_TIMEOUT_MS:5000}
```
- No Java change. Send-failure logging already exists and is centralized: `DefaultEmailService.sendInternal` (`:70-72`) logs `log.error("Failed to send email subject={}", …, e)` before rethrowing — confirmed, not duplicated.

### 3. OpenAI — `openai/service/impl/DefaultOpenAIService.java` + `openai/facade/impl/DefaultOpenAIFacade.java` + DB seed
- **Timeout (DefaultOpenAIService `sendRequest`):** the per-call Apache client is now `HttpClients.custom().setDefaultRequestConfig(requestConfig).build()` with a `RequestConfig` carrying **connect SHORT / response LONG**. Values read per call from DB config via `resolveTimeoutMs(key, fallback)`:
  - `openai.timeout.connect-ms` (default/fallback `5000`)
  - `openai.timeout.response-ms` (default/fallback `60000`)
  - `resolveTimeoutMs` falls back to the hardcoded default on a **missing / blank / non-numeric / non-positive** value. The non-positive guard is essential: a `0` timeout in Apache HttpClient 5 means **infinite** — without the guard, an empty config value (`getIntConfig` returns 0) would have re-introduced the exact hang this fixes.
- **DB seed (`data/configuration/data-configuration.sql`):** appended two rows in the OpenAI section, ids `180`/`181` (next free ids; 9/10 adjacent to the existing openai 6/7/8 are taken):
  - `(180, 'openai.timeout.connect-ms', '5000', …)`
  - `(181, 'openai.timeout.response-ms', '60000', …)`
- **Failure handling (DefaultOpenAIFacade — the operation-context seam):**
  - The two facade entry points map 1:1 to the two operations, which is the only place the operation label exists (`sendRequest` sees only an `OpenAIInputDTO`). Each wraps its OpenAI call in `try/catch` → calls `handleOpenAiFailure("translation" | "description-suggestion", e)` → **re-throws**.
  - **Graceful degrade preserved exactly** by re-throwing: `description-suggestion` propagates to the controllers (`OpenAiController`, `DashboardProductController`, both `throws Exception`) just as before; `translation` is swallowed by its three callers' existing `catch (Exception e) { /* do nothing for now */ }` (`DefaultReviewTranslationsService:89`, `DefaultUserTranslationsService:63`, `DefaultProductService:~540`). The new alert changes **no** user-facing outcome.
  - **Log:** `handleOpenAiFailure` logs `log.error("OPENAI {} failed: {}", operation, e.getMessage(), e)`.
  - **Telegram alert:** fired **after** logging via the existing `TelegramAlertService.sendMessage(...)` (already the bounded `RestTemplate`; itself best-effort — blank creds skip, any failure logs WARN, **never throws**), so it cannot throw back into the request path. Direct call, mirroring `AlertService.sendTelegram`'s pattern (no redundant wrapper).
- **Per-call client construction kept** (promoting to a reused bean is out of scope, per the audit).

**Forced-failure trace (confirmed by reading the catch chain, not run against live OpenAI):** an exception out of `openAIService.sendRequest` → caught in the facade → logged with operation label → `telegramAlertService.sendMessage(...)` (cannot throw) → `throw e` → **same outcome as today** (description: 500 to caller; translation: silently swallowed, entity saved with original-language text only).

### 4. R2/S3 — `config/R2Config.java` + `properties/R2Properties.java` + yamls
- `R2Config.r2Client`: the `UrlConnectionHttpClient.builder()` now sets `.connectionTimeout(Duration.ofMillis(props.getConnectTimeoutMs()))` and `.socketTimeout(Duration.ofMillis(props.getSocketTimeoutMs()))` (added `java.time.Duration` import).
- `R2Properties` (`@ConfigurationProperties("cloudflare.r2")`): added `connectTimeoutMs` (default `5000`) + `socketTimeoutMs` (default `10000`) with getters/setters — matches the existing `@ConfigurationProperties` style rather than introducing `@Value` into the config bean.
- yamls (all three): `cloudflare.r2.connect-timeout-ms: ${R2_CONNECT_TIMEOUT_MS:5000}` / `socket-timeout-ms: ${R2_SOCKET_TIMEOUT_MS:10000}`.
- Failure logging already present on the R2 operations (`DefaultR2Service.delete:44`, `deleteBulk:81/86`, `headObject`), so no new logging — the timeout surfaces through those existing handlers. Safe per audit: metadata ops only (deletes/lists/head, **no uploads**), so a 10s socket timeout caps no legitimate operation.

### 5. Cloudflare KV — `service/impl/DefaultCloudflareKvService.java`
- Deleted the legacy field-init `private RestTemplate restTemplate = new RestTemplate();` (`:28`). Renamed the already-injected bounded bean `boundedRestTemplate` → `restTemplate` (now the single client). Repointed the two bounded helpers (`boundedReadMaintenanceFlag`, `boundedSetMaintenanceValue`) onto it.
- **The two read helpers stayed distinct** (the critical constraint): `readMaintenanceFlag` (`:121`) still degrades every non-404 failure to "off"; `boundedReadMaintenanceFlag` (`:101`) still propagates timeout/5xx/auth so the auto-trip records FAILED. Only the client is shared; the error policies are unchanged.
- Corrected the stale `:30-36` comment that claimed the bounded helpers "collapse into the legacy ones" — it now states the single shared client + the two deliberately-distinct read policies.
- No new config (shared bean). Closes `issues.md` 2026-06-03 "DefaultCloudflareKvService uses a no-timeout RestTemplate".
- **Test updated:** `DefaultCloudflareKvServiceTest` collapsed its two mock clients into the single `restTemplate` mock; removed the now-false "legacy client never touched by assert path" assertion; added `readMaintenanceFlag_nonNotFoundFailure_degradesToOff` to lock the degrade-to-off policy alongside the existing assert-path propagate-to-FAILED test (the two distinct policies are both asserted). 17/17 green.

---

## Throttle finding (OpenAI alert)
There is **no** existing dedup/cooldown/rate-limit on the alert path — confirmed by reading `TelegramAlertService` and `AlertService` end to end (the `Throttle*` classes in the `health` package are the DB-overload *request* throttle, unrelated). Per the brief, I did **not** build one. **OpenAI alert has no throttle; a sustained outage will alert per failed request — throttling is a follow-up.**

---

## Tests
- `./mvnw spotless:check` — clean.
- `./mvnw test` — **935 passed, 0 failed.**
- Config wiring is not unit-testable without the live dependencies; verified by reading the resolved bindings and the catch chains. `resolveTimeoutMs` is exercised implicitly (fallback path) and guards the 0=infinite footgun. No tests were fabricated for the live-dependency paths.

## Stopped on / open
- Branch: changes uncommitted on `dev` — Igor branches + commits the 11 files (the tree also carries unrelated security-hardening work; do not bundle it).
- `data-configuration.sql` ids 180/181 chosen as next-free; renumber if they collide with concurrent feature work.
- Config-file impact (`conventions.md`/`decisions.md`/`state.md`/`issues.md`): drafted text for `issues.md` (close the 2026-06-03 KV no-timeout entry) is in the session summary's "For Mastermind" — not applied here (Docs/QA writes).
