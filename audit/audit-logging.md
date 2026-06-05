# Logging Implementation Brief — oglasino-backend (console only)

**Date:** 2026-06-03
**Branch:** dev
**Scope:** Add missing log lines to the backend. **Console only — logs go to stdout, exactly as today. No file storage, no volume, no rotation, no aggregator.** Where logs end up and how long they live is deliberately out of scope. This brief is only about the missing log *lines* in code.

This replaces the earlier logging audit. The persistence sections (Loki, file appenders, retention) are dropped on purpose.

---

## What you're doing

The backend already logs one line per HTTP request, and some areas log well. But a set of important events — auth failures, admin actions, integration failures, background jobs — currently log **nothing**, so they're invisible in production. You're filling those gaps.

Work in 4 batches (Part 6). One batch per session. After each batch run `./mvnw spotless:apply` and `./mvnw test` for touched modules, then summarize and stop.

---

## The pattern to match (already in the codebase — don't reinvent it)

- **Logger:** `private static final Logger log = LoggerFactory.getLogger(X.class);` (SLF4J). If a class has no logger, add this field.
- **Per-request context is free.** `logging/RequestLoggingFilter` already puts `requestId`, `method`, `path`, `clientIp` into MDC, and `FirebaseAuthFilter` adds `userId` after successful auth. The prod/stage console pattern prints `[req=… usr=… ip=…]` on every line. So any line you log *inside a request* already carries who/what/where — you don't need to repeat it.
- **Background jobs are different.** `@Scheduled`, `@Async`, and event listeners run **outside** any request — no request line, no MDC. A job that logs nothing is completely invisible. So jobs must log their own start/summary lines.
- **Levels:** root is INFO in prod (never DEBUG — it turns on per-SQL Hibernate logging). Use **INFO** for normal notable events, **WARN** for recoverable/degraded paths, **ERROR** for real failures.
- **Admin state-changes already write a DB audit row** (`DefaultUserAuditService`, `DefaultProductAuditService`). The DB row stays the permanent record. Your new log line is the *real-time* mirror so the action shows up in the live log too.

---

## Rules for every line you write (read these first)

These five rules matter more than any individual line. Follow them everywhere.

1. **Never log secrets or raw user content.** No tokens, no passwords, no email bodies, no Cloudflare/KV credentials, no raw moderated text. Log codes, ids, field names, and `e.toString()` instead.

2. **Don't log raw emails or full IP addresses — log a userId instead.**
   - When you have an authenticated user, log `userId=<id>`. That's the useful handle anyway.
   - For the rate-limit and internal-token lines (anonymous, no userId): log the **caller key the rate limiter already uses**, not a freshly-read raw IP. If you must log something IP-derived and have no userId, that's the only acceptable form.
   - Never put a raw email address in a log line. Use the userId; if there's genuinely no userId (e.g. a pre-registration email), log the field name, not the address.
   - *Why:* the privacy policy commits to short IP retention and treats emails as personal data. Keeping them out of logs avoids the conflict entirely.

3. **Strip newlines from any user-controlled string before logging it.** `path`, an email `subject`, anything a user can influence — a newline in those lets someone forge fake log lines. Replace `\r` and `\n` with a placeholder (e.g. `_`) before logging.

4. **On a real failure (ERROR), pass the exception object, not `e.toString()`.** `log.error("KV write failed: key={}", key, e)` gives you the full stack trace. Use `e.toString()` (message only) **only** on the auth-token-failure WARN, where the full detail could leak token info.

5. **No DEBUG that fires per request in prod.** Root is INFO. DEBUG triggers Hibernate per-SQL spam. Optional success lines on hot paths are DEBUG and stay off.

---

## Two findings that are bugs, not just missing logs

These two get a log line in this work **and** a separate `issues.md` entry, because the log only makes the problem *visible* — it doesn't fix it. Flag both for a separate decision; do not silently fix the behavior in this logging pass.

- **`images/job/ChatImagesRemovalJob`** — the `finally`-block bulk flush can throw uncaught. That's a correctness bug. Log it (Batch 4) and flag it.
- **`redis/service/impl/DefaultScheduledRedisFlushService`** — the per-product catch swallows a DB write failure and re-adds the delta to Redis, which causes view counts to drift. That's a data bug. Log it (Batch 4) and flag it.

---

## Out of scope (do not touch in this work)

- **No file storage, volume, rotation, or log aggregator.** Console only.
- **`DefaultCloudflareKvService` no-timeout `RestTemplate` (line 23).** Real issue, separate `issues.md` entry, but it's a behavior change, not a log line. Leave it alone here even though you're editing the same file.
- **Moderation rejection verdict logging** — user-triggered and high-volume; it would bury the useful signal. Skip.
- **Indexer / images-removed listener success lines** — already log failures via `@Recover`; a success line is just noise. Skip.

---

## Findings to implement

Line numbers were current as of 2026-06-03 — **re-confirm them in the code before editing**, they drift.

### Batch 1 — Security & auth filters (highest value, do first)

- **`security/filter/FirebaseAuthFilter.java:139`** — the `catch (Exception e)` swallows token-verification failures. Logger exists (line 29).
  → **WARN:** `"Firebase token verification failed for path={}: {}"` with `path` and `e.toString()`. **Never log the token.** (high)
- **`security/filter/FirebaseAuthFilter.java:87`** — a *verified* token that maps to no user record throws and gets swallowed by the same catch. This means cache/DB are out of sync.
  → **ERROR** (distinct line): `"Verified Firebase uid={} has no auth record (cache/DB desync)"`. (high)
- **`security/filter/RateLimitFilter.java:73`** — a 429 is returned with no log. No logger in the class — add one.
  → **WARN:** `"Rate limit exceeded: caller={} category={} retryAfterSec={}"`. Use the caller key the limiter already computes (not a raw IP read). (high)
- **`security/filter/InternalTokenFilter.java:28`** — a `/internal/*` request with a bad/missing internal token gets 401 with no log. No logger — add one.
  → **WARN:** `"Internal endpoint rejected: missing/invalid internal token path={}"`. (high)
- **`moderation/impl/LocalContentModerator.java:85`** — `catch (Exception ignored)` silently falls back to `EN`. No logger — add one.
  → **WARN:** `"Language detection failed, defaulting to EN: {}"`. **Important:** the existing comment notes this is also the normal out-of-request path in tests. Guard so the WARN does **not** fire on that benign case (a simple check of the known-benign condition — do **not** build a rate limiter). (medium)

### Batch 2 — Admin & moderation state-changes

Introduce a small shared helper that builds a uniform `Admin action:` message — leading token `Admin action:`, then `action=…`, `actorId=…`, `targetId=…`, `reason=…`. The helper returns the **message string**; each class logs it through **its own** logger (so the class name still shows in the log). Use **WARN** for punitive/destructive actions, **INFO** for benign ones.

- **`admin/facade/impl/DefaultUsersFacade.java:67`** — ban writes the audit row but logs nothing (unban already logs). Logger exists.
  → **WARN:** ban with `userId`, `adminId`, `reason`, `productsDeactivated` count. (high)
- **`admin/facade/impl/DefaultAdminProductFacade.java:42 / :65`** — product ban/unban. No logger — add one.
  → **WARN** on ban, **INFO** on unban, with `productId` (+ owner id). (high)
- **`admin/service/impl/DefaultAdminReviewService.java:44 / :55`** — approve/disapprove; disapprove increments reviewer penalties silently. No logger — add one.
  → **INFO** on approve, **WARN** on disapprove, with `reviewId`, reviewer id, target id, reason, and whether the penalty was incremented. (high)
- **`admin/facade/impl/DefaultAdminReportFacade.java:80`** — resolve report. No logger — add one.
  → **INFO:** `"reportId={} notifyUser={}"`. (medium)
- **`admin/controller/UsersController.java:96 / :115 / :145`** — lock / unlock / force-delete from deletion. No logger — add one.
  → **INFO** for lock/unlock, **WARN** for force-delete, with `targetUserId`, `adminId` (`auth.getUserId()`), `reason`. (high for force-delete, medium for lock/unlock)
- **`service/impl/DefaultConfigurationService.java:42`** — runtime config change with no log. Logger exists.
  → **WARN:** `"Config updated: key={} old={} new={}"`. (medium)
- **`admin/internal/controller/AppVersionAdminController.java:24 / :37`** — version ceiling/floor update. No logger — add one. A wrong floor can lock every client out.
  → **WARN:** `"App version floor/ceiling updated to {}"`. (medium)
- **`admin/controller/MaintenanceAdminController.java:24` / `service/impl/DefaultCloudflareKvService.java:35`** — maintenance toggle with no log.
  → **WARN:** `"Maintenance toggled -> {}"`. **Put this in the service** so it covers both callers. (medium)

### Batch 3 — External-integration boundaries (failures that vanish)

- **`service/impl/DefaultCloudflareKvService.java`** — the KV **PUT** (`setMaintenanceCloudflareValue`, ~line 69) has no try/catch and no log; `readMaintenanceFlag` (~line 56) only handles NotFound, anything else propagates raw.
  → PUT failure → **ERROR:** `"Cloudflare KV write failed: key={} value={}"` (pass the exception). Non-NotFound read failure → **WARN** + degrade gracefully: `"Cloudflare KV read failed for key={}, treating as off"`. (high PUT, medium read)
- **`images/service/impl/DefaultR2Service.java:35`** — single-object `delete(key)` has no try/catch, no log → silent orphaned file. (`deleteBulk` is fine, leave it.)
  → **WARN:** `"Failed to delete R2 object key={}"`. (medium)
- **`service/impl/DefaultEmailService.java:35`** — `mailSender.send` is unwrapped; an SMTP failure throws with no log.
  → **ERROR:** `"Failed to send email to=userId=<id> subject=<field>"` (pass the exception; **log the userId, not the address**; strip newlines from subject). Confirm catch-vs-rethrow with the email-notifications design before finalizing. (medium)
- **`notifications/service/impl/DefaultNotificationsService.java:69`** — already logs the Firestore write failure at ERROR but without the userId, so you can't tell whose notification was lost.
  → **Amend** the existing ERROR to include `userId` (and notification type). (low)
- **`listeners/UserStateChangedEventListener.java:22`** — calls cache revalidation after commit with no try/catch, no log; if it throws, users see stale ban/unban state. No logger — add one.
  → **ERROR:** `"Failed to revalidate user cache for userId={}"` (pass the exception). (medium)

### Batch 4 — Background jobs & async listeners

Match the good template already in the codebase: `jobs/ProductRemovalJob` — start line, summary with counts + elapsed ms, per-item failure logged, top-level ERROR on abort.

- **`images/job/ChatImagesRemovalJob.java:42`** — `@Scheduled`, zero logging, no logger. The `finally` bulk flush can throw uncaught (**also a bug — see "Two findings that are bugs" above**). Add logger + a deleted counter.
  → **INFO** start; **INFO** summary `"Chat images removal done: deleted={} elapsedMs={}"`; **ERROR** if the stream/flush throws. (high)
- **`redis/service/impl/DefaultScheduledRedisFlushService.java:48`** — the per-product catch swallows a DB write failure and re-queues the delta silently → drifting view counts (**also a bug — see above**).
  → **WARN** in the catch: `"View flush failed for productId={}, re-queued: {}"`; optional **INFO** summary `"View flush: persisted={} productsWithErrors={}"`. (high)
- **`jobs/ProductBaseCurrencyUpdater.java:109`** — finish line has no count (a run of 0 looks identical to a run of 100k).
  → **Amend** the finish line to include the updated count. (low)

---

## Priority table

| #  | Area        | File / method (re-confirm line)                               | Level      | Severity | Batch |
| -- | ----------- | ------------------------------------------------------------- | ---------- | -------- | ----- |
| 1  | Auth        | FirebaseAuthFilter:139 swallowed verify failure               | WARN       | high     | 1     |
| 2  | Auth        | FirebaseAuthFilter:87 verified-uid/no-record desync           | ERROR      | high     | 1     |
| 3  | Abuse       | RateLimitFilter:73 429 caller+category (+logger)              | WARN       | high     | 1     |
| 4  | Security    | InternalTokenFilter:28 internal-token reject (+logger)        | WARN       | high     | 1     |
| 5  | Moderation  | LocalContentModerator:85 lang fallback (+logger, test-safe)   | WARN       | medium   | 1     |
| 6  | Admin       | DefaultUsersFacade:67 ban                                     | WARN       | high     | 2     |
| 7  | Admin       | DefaultAdminProductFacade:42/65 product ban/unban (+logger)   | WARN/INFO  | high     | 2     |
| 8  | Admin       | DefaultAdminReviewService:44/55 approve/disapprove (+logger)  | INFO/WARN  | high     | 2     |
| 9  | Admin       | DefaultAdminReportFacade:80 resolve (+logger)                 | INFO       | medium   | 2     |
| 10 | Admin       | UsersController:96/115/145 lock/unlock/force-delete (+logger) | INFO/WARN  | high/med | 2     |
| 11 | Admin       | DefaultConfigurationService:42 config update                  | WARN       | medium   | 2     |
| 12 | Admin       | AppVersionAdminController:24/37 floor/ceiling (+logger)       | WARN       | medium   | 2     |
| 13 | Admin/Integ | Maintenance toggle (put log in CloudflareKv service)          | WARN       | medium   | 2     |
| 14 | Integration | DefaultCloudflareKvService:69 KV PUT failure                  | ERROR      | high     | 3     |
| 15 | Integration | DefaultCloudflareKvService:56 non-NotFound read failure        | WARN       | medium   | 3     |
| 16 | Integration | DefaultR2Service:35 single-key delete failure                 | WARN       | medium   | 3     |
| 17 | Integration | DefaultEmailService:35 send failure (userId, not email)       | ERROR      | medium   | 3     |
| 18 | Integration | DefaultNotificationsService:69 add userId to error            | amend      | low      | 3     |
| 19 | Listener    | UserStateChangedEventListener:22 revalidation failure (+logger)| ERROR     | medium   | 3     |
| 20 | Jobs        | ChatImagesRemovalJob:42 no logging (+logger) **also a bug**    | INFO+ERROR | high     | 4     |
| 21 | Jobs        | DefaultScheduledRedisFlushService:48 swallowed failure **bug** | WARN       | high     | 4     |
| 22 | Jobs        | ProductBaseCurrencyUpdater:109 add updated count              | amend      | low      | 4     |

**22 line-items across ~18 files.** Skipped on purpose: moderation rejection logging, indexer/images success lines, the malformed-counter skip (DEBUG-only if ever wanted).

---

## Definition of done (per batch)

- All lines in the batch added, following the five rules above.
- `./mvnw spotless:apply` and `./mvnw test` clean for touched modules.
- No DEBUG that fires per request. No tokens / emails / secrets / raw user content logged. Newlines stripped from user-controlled strings.
- The two bug findings (items 20, 21) each have an `issues.md` entry drafted in the session summary, flagging that the log line makes the bug visible but does not fix it.
- Session summary written per conventions Part 5, including the Config-file impact section (the two `issues.md` drafts).