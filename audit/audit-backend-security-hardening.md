# Audit — backend-security-hardening (revalidation)

**Repo:** oglasino-backend · **Branch:** dev · **Date:** 2026-06-03 · **Mode:** READ-ONLY
**Method:** every `file:line` below was opened with `view` (Read) **and** cross-checked with an independent `grep`/`rg`. No disagreement between reads was observed on any cited line; where a prior-audit claim diverges from the code, the divergence is called out. No code was changed.

**One-line bottom line:** The single highest-value finding (Part 1) is confirmed — the chain is allow-by-default (`anyRequest().permitAll()`), and the complete non-prefixed surface is small. Several prior-audit severities are over-stated and are recalibrated below (Parts 2, 3, 6). Two findings are genuinely live latent risk (Part 1 default-deny + Part 5 subscription-coupled identity role).

---

## Part 1 — Authorization default + complete non-prefixed path inventory

### 1a. The `authorizeHttpRequests` block — verbatim

`security/config/SecurityConfig.java:73-82`:

```java
.authorizeHttpRequests(
    auth ->
        auth.requestMatchers(HttpMethod.OPTIONS, "/**")
            .permitAll()
            .requestMatchers("/api/public/**", "/api/auth/**", "/internal/**")
            .permitAll()
            .requestMatchers("/api/secure/**")
            .authenticated()
            .anyRequest()
            .permitAll())
```

**Verdict: CONFIRMED.** The chain is allow-by-default. Only `/api/secure/**` requires authentication. Everything outside `/api/secure/`, `/api/public/`, `/api/auth/`, `/internal/` (and OPTIONS preflight) falls to `anyRequest().permitAll()` and is publicly reachable. An H1 default-deny flip (`anyRequest().authenticated()` or `.denyAll()`) must explicitly re-allow the non-prefixed surface inventoried below or it will break infra + error dispatch.

Note: `/api/secure/**` enforces only `authenticated()`. Admin authorization is layered on top via method security — `@EnableMethodSecurity` (`SecurityConfig:23`) + `@PreAuthorize("hasRole('ADMIN')")` on the admin controllers (16 admin files carry `@PreAuthorize`; `admin/controller/AdminController.java:13` is class-level `hasRole('ADMIN')`). This matters for Part 5 (a lost `ROLE_ADMIN` authority defeats those `@PreAuthorize` gates).

### 1b. Complete inventory of HTTP paths NOT under the four known prefixes

I computed each controller's full path (class `@RequestMapping` + method mapping) for all 43 controller classes. **All application-controller paths resolve INSIDE the four prefixes except one.** The complete non-prefixed surface is:

| Full path | Served by | Notes |
|---|---|---|
| `GET /health` | `health/HealthController.java:10-15` (`@RequestMapping("/health")`, `@GetMapping`) | The only application controller outside the four prefixes. Returns `{"status":"ok"}`. |
| `/actuator`, `/actuator/health`, `/actuator/health/readiness`, `/actuator/health/liveness`, `/actuator/info`, `/actuator/prometheus` | Spring Boot Actuator | base-path `/actuator`; exposure per env (see 1d). |
| `/error` | Spring Boot default `BasicErrorController` | No custom `ErrorController` exists (grep: none). This is the framework error-dispatch path. **Must stay permitAll** or error responses 401/403 themselves. |
| `OPTIONS /**` | already `permitAll` (`SecurityConfig:75-76`) | CORS preflight. |

**Two paths that LOOK non-prefixed but are not** (verified, to forestall a wrong inventory):

- `admin/controller/SuggestionController.java:19` maps the class to `@RequestMapping("/api")`, with methods `/public/suggestion` (`:24`) and `/secure/admin/suggestion` (`:33`). Concatenated these are `/api/public/suggestion` and `/api/secure/admin/suggestion` — both land **inside** the prefixes and are matched correctly. Fragile pattern (a future method directly under `/api` would escape), worth a flag, but not a current hole.
- `controller/test/TestCreateJSON.java:11` maps `/api/public/filter_gen/init` — inside `/api/public`, and additionally `@Profile("dev")` (`:12`), so it does not exist in prod/stage at all.

**No `/favicon.ico`, no static-resource handler, no `/h2-console` actually served** — see 1c.

### 1c. What serves `/media/**`

**Nothing in this backend.** `grep -rn "/media"` returns exactly one hit: the dead `WHITE_LIST_URL` constant (`SecurityConfig:31`). There is no `@RequestMapping`/`@GetMapping` for `/media`, no `WebMvcConfigurer`/`addResourceHandlers`/`ResourceHandlerRegistry` (grep: none), and no `src/main/resources/static` or `/public` directory (`ls`: absent). `/media/**` is served by the edge (Cloudflare worker / CDN), not the origin. The `WHITE_LIST_URL` entry for it is doubly dead (the constant is never referenced — see L2/L6).

### 1d. Actuator base path + exposed endpoints, per env

`management.endpoints.web.exposure.include` and `base-path` checked in **every** `application*.yaml`:

- **prod** (`application-prod.yaml:111-116`): `include: health, info, prometheus`; `base-path: /actuator`. Health probes enabled (`:120-121`), `show-details: never` (`:119`). Readiness group is dependency-aware: `readiness: include: readinessState,db,redis,elasticsearch` (`:126-127`).
- **stage** (`application-stage.yaml:117-122`): identical exposure (`health, info, prometheus`) + base-path `/actuator`. Diff from prod: `show-details: when-authorized` (`:129`). Same dependency-aware readiness group (`:136-137`).
- **dev** (`application-dev.yaml`): **no `management:` block at all.** So dev uses Spring Boot defaults — only `/actuator/health` is web-exposed; `/actuator/info` and `/actuator/prometheus` are **not** exposed in dev.

Reachability summary (prod/stage):
- `/actuator/health` ✅ (rollup) · `/actuator/health/readiness` ✅ (db+redis+es aware) · `/actuator/health/liveness` ✅ (livenessstate enabled, `:129`/`:139`) · `/actuator/info` ✅ · `/actuator/prometheus` ✅
- All under `/actuator`. All currently public (outside the four prefixes → `anyRequest().permitAll()`).

### 1e. CRITICAL — boot-loop interaction (docker-compose healthchecks)

Exact `test:` lines:

- `infra/docker-compose.yml:93` (backend service, **prod**, `SPRING_PROFILES_ACTIVE: prod` at `:9`):
  ```yaml
  test: ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health/readiness || exit 1"]
  ```
- `infra/docker-compose-stage.yml:108-114` (backend service, **stage**, `SPRING_PROFILES_ACTIVE: stage` at `:18`):
  ```yaml
  test:
    [
      "CMD-SHELL",
      "wget -qO- http://localhost:8080/actuator/health/readiness || exit 1",
    ]
  ```
- `infra/docker-compose.local.yml` and `infra/docker-compose.es.yml` have **no backend healthcheck** — they only run Postgres/Redis/ES sidecars (those healthchecks hit `pg_isready` / `redis-cli ping` / `:9200/_cluster/health`, not the backend).

**Exact URL hit + from where:** `http://localhost:8080/actuator/health/readiness`, **container-internal** (`localhost` inside the backend container, via `wget`). This is not public traffic.

**Minimum actuator surface that MUST stay reachable after a default-deny flip:** `/actuator/health/readiness` (the readiness probe). If a default-deny flip makes `/actuator/**` require auth, the container-internal `wget` (which carries no Firebase token) gets 401/403, the healthcheck fails, Docker marks the container unhealthy, and the deploy boot-loops. **The H1 fix must explicitly `permitAll` `/actuator/health/**`** (readiness + liveness) at minimum; `/actuator/health`, `/actuator/info`, `/actuator/prometheus` should be triaged separately (prometheus/info ideally locked down, but readiness/liveness must stay open to the container). `/error` must also stay `permitAll`.

### 1f. `RequestLoggingFilter` special-cases `/actuator/`

**Verdict: CONFIRMED** (prior claim cited `:50`). `logging/RequestLoggingFilter.java:49-53`:

```java
String path = req.getRequestURI();
if (path.startsWith("/actuator/")) {
  chain.doFilter(req, resp);
  return;
}
```

It skips MDC seeding + access-log emission for `/actuator/*` (the 30s liveness pings would dominate logs). It is an early-return pass-through, not an auth bypass — it does not touch the security decision. Note it is wired at `HIGHEST_PRECEDENCE` via `LoggingConfig` (its own `FilterRegistrationBean`, `logging/LoggingConfig.java:21-23`).

---

## Part 2 — Admin-by-email

**Verdict: CONFIRMED at code level; severity recalibrated to latent (effectively dead for the live admin).**

`security/service/impl/DefaultFirebaseAuthService.java:151-156`, inside `createUserSynchronized(...)`:

```java
if (token.getEmail().equals("admin@oglasino.com")) {
  newUser.setDisplayName("Admin User");
  newUser.setUserRole(UserRole.ROLE_ADMIN);
} else {
  newUser.setUserRole(UserRole.ROLE_BASIC);
}
```

- **(a) Literal email compared:** `"admin@oglasino.com"` (`:151`), exact `.equals`.
- **(b) Branch lives ONLY in the create-new-user path:** `createUserSynchronized` is reached only via `getOrCreateUser` (`:106-109`) `.orElseGet(() -> createUserSynchronized(...))` — i.e. only when `getUser(token)` returns empty (no existing row). Inside the synchronized block there is a second `getUser` re-check (`:126-127`) that also returns `new SyncResult(existing, false)` for an existing row. The role assignment cannot re-fire for an existing user row.
- **(c) Existing user's role on subsequent logins:** read from the DB, never re-assigned. The filter's hot path reads `authData.userRole()` from the `redisUserAuth` cache backed by `userRepository.findAuthDataByFirebaseUid` (`FirebaseAuthFilter.java:87` → `DefaultFirebaseAuthService.getCachedAuthData:85-92`). `getOrCreateUser`'s existing-user branch (`:107-108`) returns the row as-is.

**Effectively dead for the live admin today:** Igor controls the `admin@oglasino.com` Firebase account **and** a manually-created DB row already exists for it, and he owns `oglasino.com`. Because the row exists, the create-path never re-runs for that email, so the hardcoded grant never fires for the live admin. This is **not a live open hole** — it is a **re-arming trap**: it would re-fire if the DB row were deleted-and-recreated, or in any fresh environment seeded from Firebase before the admin row is inserted (the next account to register with that exact email would be granted `ROLE_ADMIN`). In-scope as latent hardening (move admin grant to a DB seed / explicit provisioning step rather than an email literal in the create path).

**Registration openness (context):** Any valid Firebase account that completes `firebaseSync` and has no existing row gets a `ROLE_BASIC` row — `createUserSynchronized` sets `UserRole.ROLE_BASIC` (`:155`), `SUBSCRIPTION_FREE` + `subscriptionActive=true` (`:145-146`). So registration is open to any Firebase identity; the `ROLE_ADMIN` branch could only ever fire for a *different* account if that account registered with the literal `admin@oglasino.com` address while no row for it existed. Since Igor owns the domain, an attacker cannot mint a Firebase account on that address — the practical reachability is "fresh-env seed ordering" only.

---

## Part 3 — Duplicate filter registration (contradiction resolved)

**Verdict: the 2026-06-03 security-audit claim is REFUTED; the `issues.md` 2026-06-03 entry is CORRECT.** Severity is cleanliness, not perf-doubling.

- **Base class (the deciding fact):** both extend `OncePerRequestFilter`.
  - `security/filter/FirebaseAuthFilter.java:27` — `public class FirebaseAuthFilter extends OncePerRequestFilter`
  - `security/filter/InternalTokenFilter.java:11` — `public class InternalTokenFilter extends OncePerRequestFilter`
  `OncePerRequestFilter` guards `doFilterInternal` with a per-request `…FILTERED` request attribute keyed on the filter instance, so each filter's **body runs exactly once per request** even when the same bean is wired into the chain twice. The security audit's "two full Firebase verifications + two `redisUserAuth` lookups per request" is **wrong** — there is one verify + one cache lookup per request.

- **Each is a plain `@Bean` filter AND added to the security chain:**
  - `@Bean`: `ApplicationConfig.java:22-30` (`firebaseAuthFilter`), `:32-35` (`internalTokenFilter`).
  - Security chain: `SecurityConfig.java:83` (`addFilterBefore(internalTokenFilter, …)`), `:84` (`addFilterBefore(firebaseAuthFilter, …)`).
  Because they are `@Bean Filter`s with no disabling `FilterRegistrationBean`, Spring Boot **also** auto-registers each as a standalone servlet filter (default `LOWEST_PRECEDENCE`) — so the wiring is duplicated (chain + servlet). The dedupe guard means only the chain invocation does work; the servlet-level invocation no-ops.

- **No disabled `FilterRegistrationBean` for either** (prior claim correct). `grep FilterRegistrationBean` across `src/main` returns only `RateLimitConfig` and `LoggingConfig`. `RateLimitConfig.java:61-67` is the correct pattern and is present:
  ```java
  FilterRegistrationBean<RateLimitFilter> reg = new FilterRegistrationBean<>(rateLimitFilter);
  reg.setEnabled(false);
  ```
  `LoggingConfig` uses the same pattern for `RequestLoggingFilter`. Neither `FirebaseAuthFilter` nor `InternalTokenFilter` has one.

- **Corrected per-request body execution count:** `FirebaseAuthFilter` body = **1×**, `InternalTokenFilter` body = **1×** (and `InternalTokenFilter` additionally `shouldNotFilter`s everything outside `/internal/`, `InternalTokenFilter.java:17-19`, so its body is 0× for non-`/internal` requests). Fix is cosmetic/correctness hygiene (add the two disabling `FilterRegistrationBean`s mirroring `RateLimitConfig`), not a latency fix.

---

## Part 4 — Input sanitization filter (M1)

**Verdict: CONFIRMED.** `getInputStream()` is NOT overridden, so JSON `@RequestBody` bypasses the filter; and `Sanitizer` HTML-encodes (wrong layer).

- **Overridden methods** (`security/filter/InputSanitizationFilter.java`, anonymous `HttpServletRequestWrapper`): `getParameter` (`:35`), `getParameterValues` (`:40`), `getReader` (`:48`). **`getInputStream()` is NOT among them** (grep on the file confirms — no `getInputStream` token). Spring's Jackson message converter reads `@RequestBody` from `ServletServerHttpRequest.getBody()` → `request.getInputStream()`, **not** `getReader()`. Therefore the `getReader()` override (`:48-62`), which is the only place JSON is sanitized, is **never exercised for `@RequestBody` deserialization** — it sanitizes only callers that explicitly use the reader (none of the JSON controllers do). Net: the JSON write surface passes through unsanitized.

- **What `Sanitizer.sanitize` does** (`util/Sanitizer.java:6-9`):
  ```java
  public static String sanitize(String input) {
    if (input == null) return null;
    return Encode.forHtmlContent(input.trim());
  }
  ```
  It `trim()`s **and** HTML-entity-encodes (`org.owasp.encoder.Encode.forHtmlContent`). This is output-encoding applied at the input layer — it persists `&lt;`/`&amp;`-style entities into the DB and ES, corrupting data for any non-HTML consumer (mobile, JSON APIs, search tokenization) and double-encoding when the client re-renders. Wrong layer, as claimed. (See trust-boundary notes for the M1 "is it safe to remove" question.)

- **Registration:** `@Component` (`:18`) + `@WebFilter("/*")` (`:19`). There is **no `@ServletComponentScan`** anywhere (grep: none), so the `@WebFilter` annotation is **inert** — the filter is registered purely as a Spring bean (`@Component`), which Spring Boot auto-registers as a servlet filter mapped to `/*` at `LOWEST_PRECEDENCE`. It has **no `FilterRegistrationBean`** of its own. (Minor correction to the prior "auto-registered servlet filter" framing: it's bean-auto-registration that does it, not the `@WebFilter`.)

- **Write-surface split (`@RequestBody` vs params):** 43 `@RequestBody` occurrences across 22 controllers; 27 `@RequestParam`; 0 `@ModelAttribute`/form-binding. The dominant write surface is JSON `@RequestBody` — exactly the surface the filter does **not** touch. The filter meaningfully touches only the `@RequestParam`/query-string surface (and even there, HTML-encoding is the wrong transform). So the filter covers a minority of the write surface and does the wrong thing on it.

---

## Part 5 — Subscription-coupled authorities (M3)

**Verdict: CONFIRMED at code level — the identity role is emitted ONLY inside the active-subscription branch.** Reachability for the real admin depends on the admin row's subscription columns (DB state, not determinable from code).

`security/AuthUtils.java:23-33`:

```java
private static List<GrantedAuthority> subscriptionToAuthorities(
    UserSubscription subscriptionType, boolean subscriptionActive, UserRole userRole) {
  if (!subscriptionActive || subscriptionType == null) {
    return List.of();
  }

  var subscriptionAuthority = new SimpleGrantedAuthority("ROLE_" + subscriptionType.name());
  var roleAuthority = new SimpleGrantedAuthority(userRole.name());

  return List.of(subscriptionAuthority, roleAuthority);
}
```

- The identity-role authority (`roleAuthority`, `:30`, the `ROLE_ADMIN`/`ROLE_BASIC` from `userRole.name()`) is constructed and returned **only** inside the active-subscription branch (`:29-32`). When `!subscriptionActive || subscriptionType == null`, the method returns `List.of()` (`:26`) — **zero authorities, including the identity role.** So a user with no/lapsed subscription is authenticated (the `OglasinoAuthentication` is still set in `FirebaseAuthFilter.java:123-135`) but carries no `ROLE_*` authority. Any `@PreAuthorize("hasRole('ADMIN')")` admin endpoint (Part 1b) would then deny — a lapsed-subscription admin is locked out of the entire admin surface.

  Note also `userRole.name()` is used raw (`ROLE_ADMIN` already includes the `ROLE_` prefix), while the subscription authority is `"ROLE_" + subscriptionType.name()`. Both feed `hasRole(...)`/`hasAuthority(...)` decisions; the identity role string is `ROLE_ADMIN` (correct for `hasRole('ADMIN')`).

- **Origin of `subscriptionActive`/`subscriptionType` (Part 11):** server-derived. They are read from the DB into `AuthenticatedUserDTO` via `userRepository.findAuthDataByFirebaseUid` (cached as `redisUserAuth`), then passed to `AuthUtils.subscriptionToAuthorities(AuthenticatedUserDTO)` (`AuthUtils.java:18-21`) from `FirebaseAuthFilter.java:129`. Never client-supplied. Good on trust boundary; the bug is the coupling, not the source.

- **Reachability for the real admin:** New users are created with `SUBSCRIPTION_FREE` + `subscriptionActive=true` (`DefaultFirebaseAuthService.java:145-146`), which would emit authorities normally. The live admin is a **manually-created DB row** (per the brief) — its `subscription_type`/`subscription_active` columns are not set by the create path and are not determinable from source code. **If** that row has `subscription_active=false` or `subscription_type=null`, the admin is locked out of all `@PreAuthorize` admin endpoints; **if** it carries an active subscription, the lockout is theoretical. This is a read-only audit with no DB writes — I did not query the prod admin row. The code-level coupling is real regardless; the fix (emit the identity `ROLE_*` unconditionally, gate only the subscription-tier `ROLE_<TIER>` authority on `subscriptionActive`) removes the dependency on that DB state entirely and is worth doing as hardening independent of current reachability.

---

## Part 6 — Client-controlled sort field (M4)

**Verdict: NUANCED — CONFIRMED that the raw client `field` reaches `Sort.by` with no allow-list; the "selects a seeded catalog" hypothesis is REFUTED for the request path; blast radius is Elasticsearch-sort, not SQL injection.**

- **Binding path — free-form client String, not a server catalog lookup:**
  - `dto/SearchProductsDTO.java:8` → `private ProductsFilterDTO productsFilter;` (the search endpoint takes `@RequestBody SearchProductsDTO`).
  - `dto/ProductsFilterDTO.java:17` → `private OrderTypeDTO orderBy;` (settable, `:99`).
  - `dto/OrderTypeDTO.java:8` → `private String field;` with a public setter (`:24-26`) and **no validation annotation** (the whole DTO has no `@Pattern`/`@NotNull`/etc — full file read). So `field` is bound straight from client JSON.
  A seeded catalog *does* exist (`data/core/data-order-types.sql` seeds 4 rows; `entity/OrderType.java`; `OrderTypeRepository`; `converter/OrderConverter.java:18-21` maps `OrderType→OrderTypeDTO` including `field`, served to the client inside `CatalogDTO`). But that catalog is **outbound only** — the request path does **not** resolve the client's `orderBy` against it. The server trusts whatever `field` string the client posts. Legit clients echo a seeded `field` (`createdAt` or `basePrice` — the only two seeded fields), but nothing enforces that.

- **`DefaultProductsFilterQueryBuilder` hands `field` to `Sort.by` with no whitelist — CONFIRMED:** `elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java:155-159`:
  ```java
  orderBy ->
      nativeQueryBuilder.withSort(
          Sort.by(
              new Sort.Order(
                  orderBy.getDirection(), orderBy.getField(), DEFAULT_NULL_HANDLING))),
  ```
  `orderBy.getField()` (client String) and `orderBy.getDirection()` (client `Sort.Direction` enum) flow directly into an Elasticsearch `NativeQueryBuilder.withSort`. No allow-list, no catalog re-resolution.

- **Real exposure:** This is an **Elasticsearch sort field**, not a JPA/SQL `Sort` — there is **no SQL-injection vector**. The realistic worst case is sorting on an arbitrary ES field name or a field-not-found → an ES error (likely surfaced as 500) or an unexpected ordering; not data exfiltration. Lower than "arbitrary field in a SQL ORDER BY." A whitelist is still worth adding as defense-in-depth (and to fix the convention of trusting client input for an ordering decision): resolve the client `orderBy.code` against the seeded `OrderType` catalog server-side and use the catalog's `field`/`direction`, ignoring any client-supplied `field`. Recommend NUANCED → keep on the M-list as a defense-in-depth / trust-boundary cleanup, downgrade the implied severity from "arbitrary field injection."

---

## Part 7 — Low / hardening items

- **L1 — `InternalTokenFilter` non-constant-time compare: CONFIRMED.** `security/filter/InternalTokenFilter.java:28` — `if (token == null || !token.equals(internalToken))`. `String.equals` is non-constant-time (early-exits on first mismatch). Theoretical timing side-channel on the internal token; low because `/internal/**` is not edge-exposed and the token is high-entropy. Use `MessageDigest.isEqual` / `java.security.MessageDigest`.

- **L2/L6 — `WHITE_LIST_URL` dead + lists h2/swagger + no springdoc: CONFIRMED.** `SecurityConfig.java:26-35` defines `WHITE_LIST_URL` (includes `/h2-console`, `/h2-console/**`, `/v3/api-docs/**`, `/swagger-ui/**`, `/swagger-ui.html`, `/media/**`). `grep "WHITE_LIST_URL"` returns **only the declaration** — it is never referenced in `authorizeHttpRequests` (which uses inline literals). Dead constant. No `springdoc`/`swagger`/`openapi` dependency in `pom.xml` (grep: none), so the swagger/api-docs entries are doubly dead. Delete the constant.

- **L3 — `validateForPostmanTesting` triple gate: CONFIRMED; cannot fire in prod via the profile gate, but the config row defaults to `true`.** `FirebaseAuthFilter.java:202-204`:
  ```java
  if (activeProfiles.equals("dev")
      && "true".equals(header)            // header X-Postman
      && configurationService.getBooleanConfig("postman.testing.active")) {
  ```
  Gate 1 — `activeProfiles.equals("dev")`: `activeProfiles` is `${spring.profiles.active:default}` (`:50-51`). Prod runs `SPRING_PROFILES_ACTIVE: prod` (`infra/docker-compose.yml:9`), stage runs `stage` (`infra/docker-compose-stage.yml:18`). **The `dev` profile is NOT active in prod/stage**, so this branch is unreachable in prod. Gate 2 — header `X-Postman: true`. Gate 3 — `postman.testing.active` **is a Configuration-table row, seeded to `'true'`** (`data/configuration/data-configuration.sql:4`: `(1, 'postman.testing.active', 'true', …)`). So in prod the **only** thing standing between an attacker and impersonating user id 2 (default, `:206`) / any `X-User` id is the profile-name check — gates 2 and 3 are attacker-controllable/already-true. **Defense-in-depth gap:** the impersonation path is one mis-set `SPRING_PROFILES_ACTIVE` away from live, and the config row offers no protection (it's `true`). Recommend removing the impersonation path entirely, or at minimum flipping the seed to `false` and gating on something not named `dev`.

- **L5 — CORS: CONFIRMED.** `SecurityConfig.java:55-66`: `setAllowedHeaders(List.of("*"))` (`:63`) + `setAllowCredentials(true)` (`:65`); `setAllowedMethods(Arrays.asList("GET","POST","PUT","UPDATE","DELETE","OPTIONS"))` (`:61-62`) — includes the **non-existent HTTP method `"UPDATE"`** (no-op; the real verb `PATCH` is **omitted**, so any future `@PatchMapping` would fail CORS preflight). Origins are `setAllowedOriginPatterns(corsProperties.getCors())` (`:60`), sourced from `app.security.allowed.cors`. Prod value is env-driven `${ALLOWED_CORS}` (`application-prod.yaml:221`, comment "only oglasino.com origins"); stage `${ALLOWED_CORS}` (`application-stage.yaml:232`); dev is a hardcoded localhost list (`application-dev.yaml:199-205`). The exact prod origin list is not in the repo (it's an env var) — confirm `ALLOWED_CORS` contains exact origins, not broad `*`-style patterns, since `allowCredentials:true` + a wildcard origin pattern would be a credential-exposure footgun. (`allowedHeaders:"*"` with credentials is permitted by Spring's pattern handling but broad; tighten if feasible.)

- **L7 — `Sanitizer.sanitize` trims: CONFIRMED.** `util/Sanitizer.java:8` — `Encode.forHtmlContent(input.trim())`. (Same line proves the HTML-encode in Part 4.)

- **L8 — `BotFilter` UA substring matching: CONFIRMED.** `security/filter/BotFilter.java:56-62`: lowercases the `User-Agent` and `return BOT_KEYWORDS.stream().anyMatch(k -> lower.contains(k.toLowerCase()))`. Substring contains-match on a hardcoded keyword list (`:17-32`, includes `PostmanRuntime`, `python-requests`). Trivially bypassed (any UA omitting the keywords) and prone to false positives; blocks with 204 (`:49`). Low. Note it has a dev/Postman bypass (`shouldNotFilter`, `:38-40`).

- **M5 — H2 scope + console: CONFIRMED (with nuance).** `pom.xml:183-187` declares `com.h2database:h2` with `<scope>runtime</scope>` (not `test`) — so H2 ships on the production runtime classpath. **However** `spring.h2.console.enabled` is **not set anywhere** (grep over `src/`: the only `h2-console` hits are the dead `WHITE_LIST_URL` entries), so Spring Boot's default (`false`) applies and the H2 console servlet is **not** registered. The runtime-scope H2 jar is unnecessary weight/attack-surface on the prod image (DB is Postgres) and should be moved to `<scope>test</scope>`; the console is not actually exposed today.

---

## Trust-boundary and seam notes

- **Part 11 — auth identity is server-derived (good):** `FirebaseAuthFilter` verifies the Firebase ID token (`:82`), loads `AuthenticatedUserDTO` from the `redisUserAuth` cache backed by `userRepository.findAuthDataByFirebaseUid` (`:87`), and populates `OglasinoAuthentication` with `userId`, `baseSiteId`, `preferredLanguage`, and authorities derived from server-side subscription/role (`:123-135`). `subscriptionActive`/`subscriptionType`/`userRole` all originate from the DB, not token claims. The email-verification gate correctly reads the **live** token claim `decoded.isEmailVerified()` rather than the stale `emailVerifiedExternal` column (`:99-110`, comment documents this). No client-supplied trust value was found in the auth path.

- **Part 6 is a trust-boundary smell:** `orderBy.field` is a client value used in a (sort) decision without server re-derivation. Per Part 11 the server should derive the sort `field` from its own seeded `OrderType` catalog keyed by the client's `code`, not trust the client's `field`. Low blast radius (ES sort) but it's the same class of issue as the historical `oldName` trust bug.

- **M1 removal safety (the key seam question — is it safe to remove `InputSanitizationFilter`/`Sanitizer`?):** This backend's contract is **codes-not-messages** (conventions Part 7) and **the client renders all user-visible strings** (web = Next.js/React, mobile = React Native). React and React Native **escape interpolated strings by default** — the XSS protection belongs on the render side, which already has it. HTML-entity-encoding at the backend input layer is therefore both the wrong layer *and* actively harmful: it persists `&lt;`/`&amp;` entities that the clients then render literally or double-escape, corrupting product names/descriptions/messages for every consumer (and ES tokenizes the encoded form). The one caller class to check before removal is anywhere the backend emits a string into an HTML context itself — **email bodies** (Brevo SMTP) and any server-rendered HTML. A spec/Phase-4 task should confirm the email templates escape at their own render step (or apply targeted encoding there) so removing the global input filter does not open an email-HTML-injection seam. Net recommendation for the spec: **remove the global HTML-encoding filter; if any input sanitization is wanted, make it semantic (trim/normalize/strip-control-chars), not HTML-encoding; push XSS encoding to the actual HTML render points (email).** This is a cross-repo seam (web/mobile render behavior) — flag for Mastermind seam analysis, not a unilateral backend call.

- **Boot-loop seam (Part 1e):** the H1 default-deny flip intersects the infra healthcheck contract (`/actuator/health/readiness`, container-internal) and the edge worker's mobile-liveness probe (per the maintenance-split feature, which re-pointed the edge probe at `/actuator/health/readiness`). The spec must keep `/actuator/health/**` reachable to unauthenticated container-internal + edge callers, or it breaks both the Docker healthcheck and the edge backend-availability gate.

---

## Corrections to the prior 2026-06-03 audit

1. **Part 3 (severity over-stated):** The security audit claimed `FirebaseAuthFilter` + `InternalTokenFilter` run **twice per request** (two Firebase verifications, two `redisUserAuth` lookups). **REFUTED.** Both extend `OncePerRequestFilter` (`FirebaseAuthFilter.java:27`, `InternalTokenFilter.java:11`); the dedupe guard means each body runs once. The `issues.md` 2026-06-03 entry ("body runs once, wiring duplicated") is the correct reading. Severity: cleanliness, not perf-doubling.

2. **Part 6 (severity over-stated / mechanism mischaracterized):** Prior claim implied an arbitrary-field injection risk. The `field` is indeed free-form client input reaching `Sort.by` with no allow-list (CONFIRMED), **but** it is an **Elasticsearch** sort, not SQL — no SQLi. And a seeded `OrderType` catalog exists (the prior audit's framing of "free-form String … no allow-list" is literally true but omits that the request path simply ignores the catalog rather than there being no catalog). Recalibrate to defense-in-depth / trust-boundary cleanup, lower blast radius.

3. **Part 2 (severity recalibrated):** Prior claim "admin role is granted to whoever registers with `admin@oglasino.com`." True for a *fresh* row, but **the branch is in the create-path only and the live admin row already exists**, so it is effectively dead for the live admin — a re-arming trap, not a live open hole. (Code claim CONFIRMED; severity recalibrated to latent.)

4. **Part 4 (mechanism refinement):** Prior claim "auto-registered servlet filter" via `@WebFilter`. The `@WebFilter("/*")` annotation is **inert** — there is no `@ServletComponentScan`. Registration happens because the class is a `@Component`, which Spring Boot bean-auto-registers as a servlet filter. The core claim (no `getInputStream` override → `@RequestBody` bypassed; HTML-encode is wrong layer) is **CONFIRMED**.

5. **Part 1 (`/media/**` clarified):** It is not served by any backend handler at all (no controller, no resource handler, no static dir) — it is an edge/CDN path. The `WHITE_LIST_URL` `/media/**` entry is dead (the whole constant is unreferenced).

6. **L2/L6 (confirmed dead, doubly):** `WHITE_LIST_URL` is unused *and* the swagger/api-docs entries reference dependencies that aren't on the classpath.

7. **M5 (nuance added):** H2 is `runtime`-scoped (CONFIRMED, ships on prod), **but** `spring.h2.console.enabled` is unset, so the console is **not** actually exposed today — the live risk is "unnecessary jar on the prod image," not "open H2 console."

8. **L3 (nuance added):** The triple gate is real and the `dev`-profile gate keeps it dead in prod — but the `postman.testing.active` Configuration row is **seeded `true`**, so two of the three gates are already satisfied/attacker-controllable in prod; only the profile name protects it.

---

## Adjacent observations (Part 4b — flagged, not fixed)

- **`controller/test/TestCreateJSON.java`** ships a `@Profile("dev")` test/dev-tooling controller in `src/main` (`/api/public/filter_gen/init`). Harmless in prod (profile-gated) but it's production source carrying a dev-only endpoint. Low.
- **CORS `allowedMethods` lists `"UPDATE"`** (not a real HTTP method) and **omits `PATCH`** (`SecurityConfig.java:61-62`). Cosmetic today (no `@PatchMapping` exists), but a future PATCH endpoint would silently fail preflight. Low.
- **`SuggestionController` maps the class to `@RequestMapping("/api")`** and relies on method paths landing inside `/public/` or `/secure/` (`SuggestionController.java:19,24,33`). Works today; fragile — a method mapped directly under the class would sit at `/api/<x>` outside all four security prefixes. Low/medium for a default-deny world (worth normalizing to explicit `/api/public/...` / `/api/secure/...` class mappings).
- **`DefaultCloudflareKvService` no-timeout `RestTemplate`** — already logged in `issues.md` 2026-06-03; not re-investigated here, noted for completeness.

---

## Notes on tool reliability (standing rule)

Every cited `file:line` was opened with Read and independently confirmed with `grep`/`rg` on the same token. No instance of disagreeing reads (the phantom-content failure mode in state.md Risk Watch) was encountered this session. There is **no base `application.yaml`/`.properties`** — only `application-{dev,stage,prod}.yaml` (`ls` confirmed); dev has no `management:` block, which is why dev's actuator exposure differs from prod/stage. This was verified, not assumed.
