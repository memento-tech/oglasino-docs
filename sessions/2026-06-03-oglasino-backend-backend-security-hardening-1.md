# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-03
**Task:** H1 default-deny flip (`anyRequest().authenticated()` + explicit re-allow list) in `SecurityConfig`, plus `SuggestionController` path normalization, with per-path authorization verification.

## Implemented

- Flipped the terminal authorization rule from `anyRequest().permitAll()` (allow-by-default) to `anyRequest().authenticated()` (default-deny) in `security/config/SecurityConfig.java`, and made the full public surface explicit and ordered: `OPTIONS /**` permit, `/actuator/health/**` permit (boot-loop guard), `/error` permit, `/health` permit, `/actuator/prometheus` + `/actuator/info` **denyAll**, then `/api/public/**` `/api/auth/**` `/internal/**` permit, `/api/secure/**` authenticated, everything else denied. Matcher order is top-to-bottom first-match: the `denyAll` for prometheus/info sits before the broad app-surface permits; there is no `/actuator/**` blanket permit anywhere.
- Normalized `admin/controller/SuggestionController.java`: dropped the bare class `@RequestMapping("/api")` and moved each method to its explicit full path (`@PostMapping("/api/public/suggestion")`, `@PostMapping("/api/secure/admin/suggestion")`). Resolved paths are byte-for-byte unchanged; removed the now-unused `RequestMapping` import. This removes the default-deny foot-gun where a future method on the class could land at `/api/<x>` outside all four prefixes.
- Added `security/config/SecurityConfigAuthorizationTest.java` (14 tests) that boots the **real** `SecurityConfig` filter chain (so the matcher order and the default `AuthenticationEntryPoint` are production-identical) with pass-through stubs for the five custom filters, and asserts the no-credentials status per path — including the made-up `/api/totally-new-endpoint` proving the flip denies the unknown surface, and the `@PreAuthorize` admin gate proving method security is independent of the URL flip.

## Part 1 — verified surface inventory (reproduced in code, not trusted from the audit)

Every claim below is my own Read + an independent grep this session. **The inventory matches `sessions/audit-backend-security-hardening.md` Part 1 exactly — no divergence.**

- **Authorization block (pre-change), verbatim:** `SecurityConfig.java:77-86` ended with `.anyRequest().permitAll()`; only `/api/secure/**` required auth. CONFIRMED allow-by-default.
- **Non-prefixed app paths:** grepped all `@RequestMapping` across 43 `@RestController` classes; computed class-prefix + method. **The only application path outside the four prefixes is `GET /health`** (`health/HealthController.java:10`). The two `@RestController`-substring hits with no class mapping are `@RestControllerAdvice` exception handlers (`GlobalExceptionHandler`, `images/.../ImageExceptionHandler`), not path handlers. `SuggestionController` was the bare-`/api` foot-gun (now normalized). No other non-prefixed app path exists.
- **Actuator, per env:** prod (`application-prod.yaml:111-130`) and stage (`application-stage.yaml:117-140`) both expose `health, info, prometheus` under base-path `/actuator`; readiness group is dependency-aware (`readinessState,db,redis,elasticsearch`); liveness enabled. **dev has no `management:` block** (Spring defaults → only `/actuator/health` web-exposed). `/actuator/health/readiness` resolves at exactly `/actuator/health/readiness`.
- **`/error`:** Spring default `BasicErrorController`; no custom `ErrorController` (grep: none).
- **`/media/**`:** no backend handler — single grep hit is the dead `WHITE_LIST_URL` constant (`SecurityConfig.java:32`); no `addResourceHandlers`/`WebMvcConfigurer`, no `static`/`public` dir. Correctly **NOT** added to the re-allow list.
- **Boot-loop consumers:** docker healthcheck `wget …/actuator/health/readiness` confirmed at `infra/docker-compose.yml:96` (prod) and `infra/docker-compose-stage.yml:115` (stage); edge probe `fetch(${BACKEND_ORIGIN}/actuator/health/readiness)` confirmed at `../oglasino-router/src/index.ts:146` with `if (!probe.ok) backendDown = true` at `:149`. Both unauthenticated; both covered by `/actuator/health/**` permit.

## Files touched

- src/main/java/com/memento/tech/oglasino/security/config/SecurityConfig.java (+26 / -7)
- src/main/java/com/memento/tech/oglasino/admin/controller/SuggestionController.java (+2 / -3)
- src/test/java/com/memento/tech/oglasino/security/config/SecurityConfigAuthorizationTest.java (new, +250)

## Tests

- Ran: `./mvnw test` (full suite — `SecurityConfig` is a core file)
- Result: **890 passed, 0 failed, 0 errors** (was 876 before; +14 new)
- New test: `SecurityConfigAuthorizationTest` — per-path no-credentials authorization + the `@PreAuthorize` admin gate.
- `./mvnw spotless:check`: green (ran `spotless:apply` once for two long-comment/long-statement wraps in the new test, then re-checked clean).

### Verification table — actual observed behavior (no credentials)

| Path | Brief expectation | **Observed** | Verdict |
|---|---|---|---|
| `GET /actuator/health/readiness` | 200 | **200** | ✅ boot-loop guard holds |
| `GET /actuator/health/liveness` | 200 | **200** | ✅ |
| `GET /actuator/health` | 200 | **200** | ✅ |
| `GET /error` | not 401/403 | **not 401/403** | ✅ reachable |
| `GET /health` | 200 | **200** | ✅ |
| `GET /actuator/prometheus` | 403 | **403** (denyAll) | ✅ locked down |
| `GET /actuator/info` | 403 | **403** (denyAll) | ✅ locked down |
| `POST /api/public/suggestion` | reaches controller | **200** | ✅ normalized path, public |
| `POST /api/secure/admin/suggestion` (no token) | 401 | **403** | denied — see Brief-vs-reality |
| `GET /api/secure/probe` (no token) | 401 | **403** | denied — see Brief-vs-reality |
| `GET /api/totally-new-endpoint` (no token) | 401 | **403** | ✅ **default-deny denies the unknown surface** |
| `POST /api/secure/admin/suggestion` (non-admin token) | 403 | **403** | ✅ `@PreAuthorize` independent + intact |
| `POST /api/secure/admin/suggestion` (admin token) | 200 | **200** | ✅ |

## Brief vs reality

I implemented the flip as specified; this is a **non-blocking** observation on the verification table, not a challenge that stopped work (the security invariant — denied vs allowed — is correct for every path; the flip is safe and shipped).

1. **Anonymous denials return 403, not 401**
   - Brief Part 5 table says: `/api/secure/**` and the unknown `/api/...` path → **401** without a token; prometheus/info → 403.
   - Code says: **every** unauthenticated denial returns **403** (empirically pinned in the test). With `formLogin` and `httpBasic` disabled (`SecurityConfig.java:73-74`) and **no custom `AuthenticationEntryPoint`** configured, Spring Security's default entry point is `Http403ForbiddenEntryPoint` → 403 for all anonymous denials, whether the rule is `authenticated()` or `denyAll()`.
   - Why this matters: it does **not** matter for the security goal — denied is denied, and this is **pre-existing** behavior for `/api/secure/**` (that rule is unchanged by the flip, so no regression). It matters only as an accuracy note so the spec/brief table isn't read as "must be 401." The 401 path in `FirebaseAuthFilter.java:167-168` fires only for a **present-but-invalid** token; a **no-token** request never hits it and falls to the 403 entry point.
   - Resolution: I assert the invariant `denied ∈ {401,403}` in the test (with a javadoc note) rather than hard-coding 401, and documented the observed 403 here. No code change to the entry point (out of scope: brief forbids touching filters/entry point). Flagging for the spec table to be corrected before later briefs build on a "401" assumption.

## Cleanup performed

- Removed the unused `org.springframework.web.bind.annotation.RequestMapping` import from `SuggestionController.java` (the class-level annotation it served was dropped).
- (The dead `WHITE_LIST_URL` constant, dead `/media` entry, and swagger/h2 entries in `SecurityConfig` were **left in place** — they are L2/L6 cleanup items scoped to a later brief in this feature, not this one. Flagged below.)

## Config-file impact

- conventions.md: no change.
- decisions.md: **new entry DRAFTED below** (default-deny posture) — batched for feature close per the brief; do not apply now.
- state.md: **H1 status flip DRAFTED below** — batched for feature close per the brief; do not apply now.
- issues.md: no change.

## Obsoleted by this session

- The `anyRequest().permitAll()` allow-by-default posture — replaced by default-deny, deleted in this session.
- The bare `@RequestMapping("/api")` class mapping on `SuggestionController` — replaced by explicit method paths, deleted in this session.
- Nothing else (no stale tests, no now-dead code introduced).

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no debug logging, unused import removed, spotless + full test green.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): flagged in "For Mastermind."
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundaries) — confirmed: the flip strengthens the boundary (unknown surface now denied); no client-supplied value enters an authorization decision. Part 8 (the Cloudflare worker is the edge boundary; backend liveness must stay reachable) — confirmed via the `/actuator/health/**` permit.

## Known gaps / TODOs

- None. No `TODO`/`FIXME` added.
- The actuator/metrics lock-down assertions verify the **authorization decision** via stub handlers; the real actuator health aggregation (db/redis/es) is the stage smoke's job (brief "Smoke" section), by design.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): the `SecurityConfigAuthorizationTest` pass-through-filter harness — earned because the five custom filters are constructor deps of the real `SecurityConfig` and must be present for the chain to build, but their token/throttle logic is orthogonal to the URL authorization decision under test; stubbing them to pass through is the minimal way to exercise the real authorization rules without booting infra (the repo has no `@SpringBootTest`).
  - Considered and rejected: (a) a full `@SpringBootTest` MockMvc boot — rejected, needs DB/Redis/ES and the repo deliberately has none; (b) duplicating the authorization rules in the test instead of importing the real `SecurityConfig` — rejected, it would test a copy, not the config; (c) tightening `assertDenied` to a hard `401` — rejected, reality is 403 and over-coupling to the entry-point default adds brittleness without value.
  - Simplified or removed: the `SuggestionController` class-level `@RequestMapping("/api")` indirection — inlined to explicit full method paths (one fewer concatenation to reason about under default-deny).
- **Adjacent observations (Part 4b):**
  - `security/filter/FirebaseAuthFilter.java:120` — the comment claims `/api/secure/** → 401` for the anonymous fall-through, but the real anonymous-denial status is **403** (default `Http403ForbiddenEntryPoint`). Stale/inaccurate comment. **Severity: low** (cosmetic; could mislead a future reader about the wire contract). Did not fix — the brief forbids touching the filters this session.
  - `SecurityConfig.java:27-36` `WHITE_LIST_URL` is dead (unreferenced; lists `/media`, `/h2-console`, swagger/api-docs entries whose deps aren't on the classpath). **Severity: low.** Did not fix — it is L2/L6, scoped to the cleanup brief later in this feature, not H1.
  - The actuator discovery index `GET /actuator` (the links page, exposed in prod/stage) now falls to `anyRequest().authenticated()` → 403 to anonymous callers (was public pre-flip). **Intended and desirable** (nothing consumes the index; it leaks the endpoint map). Noted for awareness, not a defect.
- **DRAFT for decisions.md (batched for feature close — do NOT apply now):**
  > **2026-06-03 — Backend authorization is default-deny (H1, backend-security-hardening).** `SecurityConfig.authorizeHttpRequests` now terminates in `anyRequest().authenticated()` (was `permitAll()`). The complete public surface is explicit: `OPTIONS /**`, `/actuator/health/**` (unauthenticated — docker healthcheck + edge mobile liveness probe carry no token), `/error`, `/health`, and `/api/public/**` + `/api/auth/**` + `/internal/**`; `/actuator/prometheus` + `/actuator/info` are `denyAll` (nothing scrapes them; metrics shouldn't be internet-reachable via the edge). Any new path outside these is denied by default — adding a public endpoint now requires an explicit matcher. Note: with `formLogin`/`httpBasic` disabled and no custom `AuthenticationEntryPoint`, all unauthenticated denials return **403** (Spring's default `Http403ForbiddenEntryPoint`), not 401.
- **DRAFT for state.md (Backend Security Hardening section — batched for feature close):**
  > H1 (default-deny flip + SuggestionController normalization) shipped on backend `dev` (Brief 1, 2026-06-03); 890 tests green, per-path authorization verified by `SecurityConfigAuthorizationTest`. Remaining Phase-5 briefs: H2 admin out-of-band, web JsonLd escape (gates → M1 filter removal), M3+M4, cleanup (M2, M5, L1–L8, Postman path).
- **Spec correction request:** `features/backend-security-hardening.md` §4 Brief 1 Part 5 table lists 401 for `/api/secure/**` and the unknown-path rows; actual is 403 (proven). Recommend correcting the table to 403 so H2/cleanup briefs don't assert a 401 that never occurs.
- No closure-gate violation: the only config-file impacts are the two drafts above, both explicitly batched for feature close per the brief's own "Config-file impact" instruction. No unstated config dependency.
