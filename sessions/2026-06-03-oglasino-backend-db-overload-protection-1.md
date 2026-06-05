# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-03
**Task:** READ-ONLY audit — inventory current backend state against the DB Overload Protection brief's 17 questions; write findings to `.agent/audit-db-overload-protection.md`.

## Implemented

- Read-only audit only. No source changed. Produced `.agent/audit-db-overload-protection.md` answering all 17 questions with file path + actual value, marking spec/code disagreements.
- Established ground truth on: per-env HikariCP pool size (dev 20 / stage 8 / prod 18), runtime pool readability (in-process `HikariDataSource` MXBean; no HTTP-exposed Micrometer metrics), Configuration-table seeding (single shared seed, no per-env split), filter ordering (no `@Order` anywhere; deterministic trio only inside the Security chain), the existing `DefaultCloudflareKvService` KV writer, `EmailService.sendPlainText`, `SystemErrorCode`/`ErrorCode` shape, free `incident_log` name, Tomcat threads, Hikari timing, `pg_stat_statements` (unused), `@Scheduled` resolving from YAML, the 429-body mechanism to mirror, and ERRORS-namespace next-available IDs.

## Files touched

- `.agent/audit-db-overload-protection.md` (new, audit deliverable) — the only file created.
- `.agent/2026-06-03-oglasino-backend-db-overload-protection-1.md` + `.agent/last-session.md` (this summary).
- No `src/` changes.

## Tests

- Not run — read-only audit, no code change. (`./mvnw spotless:check` / `./mvnw test` N/A: nothing compiled.)

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (A new `db-overload-protection` feature row may eventually be authored by Docs/QA when the feature enters the pipeline, but that is Mastermind/Phase-4 work, not an engineer draft — not required by this audit.)
- issues.md: no change required from me. One pre-existing adjacent observation surfaced (no-timeout `RestTemplate` in `DefaultCloudflareKvService`) — flagged below in "For Mastermind", not authored into issues.md (engineer agents draft, Docs/QA writes).

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no seed rows added; the audit *reports* the next-available ERRORS IDs for a future brief but reserves/writes nothing).
- Part 7 (error contract): confirmed-and-flagged — the as-built filter body extends `{field, code}` with `translationKey`; surfaced as a discrepancy in audit Q15.
- Part 10 (lifecycle): confirmed — this is the Phase-2 audit; the canonical spec (`features/db-overload-protection.md`) does not exist yet, which is expected.
- Part 11 (trust boundaries): explicitly checked in audit Q17 — boundary is clean; `RateLimitFilter`'s client-header bucket key flagged as the one pattern the throttle must NOT copy for its trip decision.

## Known gaps / TODOs

- Q12 (DO-side `pg_stat_statements` grant/extension state) cannot be answered from code — reported as such in the audit.
- Resolved env-var values (`DATASOURCE_*`, `ALLOWED_CORS`) are host/`.env`/compose-supplied and out of repo; pool literals (in YAML) are reported.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no abstractions/config/patterns introduced.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **The "spec" file does not exist on disk.** The brief points at `oglasino-docs/features/db-overload-protection.md`; it is absent (closest is `connection-pool-hardening.md`). Consistent with Part 10 (audit precedes spec), so I proceeded against code and the brief's self-contained questions. Flagging so you know the spec is yet to be authored.

- **Six discrepancies between the brief's assumptions and the code** (full detail in the audit's "Summary of discrepancies"):
  1. Pool size 18 is **prod-only** (dev 20, stage 8) → derive thresholds from the live `HikariDataSource`, not a literal.
  2. No HTTP-exposed HikariCP metrics (no `micrometer-registry-prometheus`; dev has no `management` block) → read the pool in-process.
  3. No per-environment Configuration-table seed mechanism exists → "default-OFF dev / opt-in prod" needs a per-profile YAML env-var flag or the app_version_config seed-neutral-then-operator-mutate pattern.
  4. The proposed 503 body omits `translationKey`; the as-built `RateLimitFilter` body includes it → mirror the filter (and reconcile against Part 7's `{field, code}`-only envelope).
  5. No filter has `@Order`; place `RequestThrottleFilter` via `SecurityConfig.addFilterAfter(..., RateLimitFilter.class)` for determinism, not as a bare `@Component`.
  6. `leak-detection-threshold` disabled in all envs; dev `connection-timeout` is the 30s default → key the throttle off `getThreadsAwaitingConnection()`.

- **Part 4b adjacent observations (out of scope — not fixed):**
  - **(medium)** `DefaultCloudflareKvService` (`service/impl/DefaultCloudflareKvService.java:23`) uses `new RestTemplate()` with **no connect/read timeout** (vs the 3s-timeout `RestTemplate` in `ApplicationConfig`). A hung Cloudflare KV call blocks the calling thread indefinitely — bad on a degraded auto-trip path that calls KV precisely when the system is already stressed. Did not fix — out of audit scope.
  - **(medium)** `FirebaseAuthFilter` and `InternalTokenFilter` are `@Bean` `OncePerRequestFilter`s with **no `FilterRegistrationBean` disabling servlet auto-registration** (`ApplicationConfig:22,33`), so they are double-registered (Security chain + standalone servlet filter). `OncePerRequestFilter` dedupe means the body runs once, but the wiring is duplicated. `RateLimitConfig:61-67` shows the correct disable pattern. Pre-existing; did not fix — out of scope.

- **No config-file edits required by this session.** Explicitly confirmed: this audit produces no `conventions.md`/`decisions.md`/`state.md`/`issues.md` dependency that must be applied before closure. The two Part 4b items above are flagged for your triage into `issues.md` if you choose; they are not my writes.
