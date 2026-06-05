# Security Audit — Spring Security Setup

**Repo:** `oglasino-backend`
**Date:** 2026-06-03
**Scope:** Spring Security configuration, authentication/authorization filters, trust-boundary handling, and adjacent injection/exposure surfaces. This is a review only — no code was changed.

---

## Summary

The security model is fundamentally sound: stateless bearer-token auth via Firebase, a respected server-side trust boundary (conventions §11), method-level role enforcement on every admin endpoint, distributed rate limiting, and externalized secrets in prod. The weaknesses are mostly **structural defaults** (fail-open authorization), a **privilege-assignment shortcut** (admin-by-email), and **a few filters that do less than they appear to**.

Counts: **3 high**, **5 medium**, **8 low/hardening**, plus a "what's good — keep it" list.

---

## Request pipeline (as built)

```
InputSanitizationFilter (servlet-level, /*)          ← @Component, auto-registered
  └─ springSecurityFilterChain
       ├─ InternalTokenFilter        (only /internal/**, X-INTERNAL-TOKEN)
       ├─ FirebaseAuthFilter         (verifies token, loads cached AuthenticatedUserDTO)
       └─ RateLimitFilter            (Bucket4j/Redis, after auth so SecurityContext is set)
  └─ BotFilter (servlet-level)
  └─ DispatcherServlet → controllers (@PreAuthorize on admin)
```

Identity is derived from a verified Firebase ID token; `OglasinoAuthentication` carries `userId`, `firebaseUid`, `baseSiteId`, `preferredLanguage`, and authorities, and is read by trust decisions — never the raw token claims (except the live `email_verified` gate, which is correct).

---

## High severity

### H1 — Fail-open authorization default

`SecurityConfig.java:73-82`

```java
auth.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
    .requestMatchers("/api/public/**", "/api/auth/**", "/internal/**").permitAll()
    .requestMatchers("/api/secure/**").authenticated()
    .anyRequest().permitAll();   // ← everything else is public
```

Authorization is allow-by-default. Only `/api/secure/**` requires authentication; **any path that does not start with `/api/secure/`, `/api/public/`, `/api/auth/`, or `/internal/` is public**. This includes:

- `/actuator/**` — `/actuator/prometheus` is publicly scrapeable (internal metrics, route names, queue depths, JVM internals leak), and `/actuator/health` / `/info`. `RequestLoggingFilter.java:50` even special-cases `/actuator/` as unauthenticated infra traffic, confirming it's reachable without auth.
- `/media/**`.
- **Any new endpoint a future engineer mounts outside the four known prefixes** — silently unauthenticated.

**Impact:** Information disclosure today (Prometheus); latent auth bypass for any future misplaced endpoint.

**Recommendation:** Default-deny. Replace the trailing rule with `.anyRequest().authenticated()` (or `denyAll()`), then explicitly enumerate the public surface. Move `/actuator/**` to either `denyAll` (scrape over the Docker network only) or an authenticated/internal-token rule. This single change also contains M5 and L4/L6 below.

### H2 — Admin role assigned by a hardcoded email string

`DefaultFirebaseAuthService.java:151-156`

```java
if (token.getEmail().equals("admin@oglasino.com")) {
  newUser.setDisplayName("Admin User");
  newUser.setUserRole(UserRole.ROLE_ADMIN);
} else {
  newUser.setUserRole(UserRole.ROLE_BASIC);
}
```

Admin privilege is granted to whoever registers with the email `admin@oglasino.com`. The email is taken from a _verified_ Firebase token, so an attacker must control that Firebase account — but the safety of the entire admin surface now rests on a single assumption: **that this exact email can never be registered by anyone but the operator.** If that Firebase account does not already exist and registration is open, an attacker who can create/verify it self-assigns admin. If it does exist, its credentials become a single point of total compromise.

**Impact:** Privilege escalation to full admin (all `/api/secure/admin/**` controllers) under a recoverable precondition.

**Recommendation:** Do not derive roles from an email literal in code. Assign the admin role out-of-band — a seeded DB row, a Firebase custom claim provisioned by the operator, or an explicit admin-bootstrap step — and have registration always create `ROLE_BASIC`. At minimum, confirm `admin@oglasino.com` is a pre-provisioned, locked-down Firebase account and document that dependency.

### H3 — Firebase service-account key committed to git

Tracked already in project memory (`firebase-key-in-git`). Prod loads the key correctly from `/run/secrets/firebase.json` (`application-prod.yaml`, `FirebaseConfig.java:42-63`), so the runtime path is clean — the liability is the historical committed copy.

**Recommendation:** Rotate the service-account key after prod is verified (consistent with the existing "secret rotation deferred" decision), and ensure the file is git-ignored / purged from history at that point.

---

## Medium severity

### M1 — Input-sanitization filter does not cover JSON request bodies, and is the wrong layer anyway

`InputSanitizationFilter.java:24-90`, `Sanitizer.java`

The filter overrides `getReader()`, `getParameter()`, and `getParameterValues()`. But Spring/Jackson read `@RequestBody` JSON from **`getInputStream()`**, which is _not_ overridden (`AbstractJackson2HttpMessageConverter` → `HttpInputMessage.getBody()` → `servletRequest.getInputStream()`). Therefore **JSON bodies — the primary write surface — pass through unsanitized** despite the filter's appearance. Only query/form parameters are touched.

Two problems:

1. **False sense of protection.** The filter looks like global XSS defense but doesn't apply to the requests that matter.
2. **Wrong technique even where it does apply.** HTML-encoding on _input_ (`Encode.forHtmlContent`) corrupts stored data (`Ben & Jerry` → `Ben &amp; Jerry`), double-encodes across round-trips, and the body path reads the entire request into memory and silently degrades on non-JSON. XSS should be handled by **output/context encoding at render time**, owned by the consuming client/SSR layer.

**Recommendation:** Decide one way. Either remove input sanitization entirely and rely on output encoding (preferred), or — if a defense-in-depth input filter is truly wanted — also wrap `getInputStream()` and switch from HTML-encoding to genuine input _validation_ (reject/strip control chars), not encoding. Do not leave it in its current half-working state.

### M2 — `FirebaseAuthFilter` runs twice per request (duplicate registration)

`ApplicationConfig.java:21-35`, `SecurityConfig.java:84`

`FirebaseAuthFilter` and `InternalTokenFilter` are plain `@Bean`s of type `Filter`. Spring Boot **auto-registers every `Filter` bean with the servlet container for `/*`** in addition to their placement in the security chain. `RateLimitConfig.java:61-67` deliberately suppresses exactly this with a disabled `FilterRegistrationBean` (and documents why) — but the same guard is missing for these two filters.

Net effect: `FirebaseAuthFilter` executes **twice per request** — two full Firebase token verifications and two `redisUserAuth` lookups — once in the security chain and once as the auto-registered servlet filter. `InternalTokenFilter` likewise (bounded to `/internal/**` by its `shouldNotFilter`).

**Impact:** Doubled per-request auth cost (Firebase verify + Redis) on the hot path; redundant `SecurityContext` writes.

**Recommendation:** Add disabled `FilterRegistrationBean<FirebaseAuthFilter>` and `FilterRegistrationBean<InternalTokenFilter>` beans (mirroring `RateLimitConfig`) so each runs only inside the security chain. Verify with a startup log of registered filters.

### M3 — Authorities are coupled to subscription state; an inactive subscription strips the user's role

`AuthUtils.java:23-33`

```java
if (!subscriptionActive || subscriptionType == null) {
  return List.of();                       // ← no authorities at all
}
var subscriptionAuthority = new SimpleGrantedAuthority("ROLE_" + subscriptionType.name());
var roleAuthority        = new SimpleGrantedAuthority(userRole.name());
return List.of(subscriptionAuthority, roleAuthority);
```

The user's identity role (`ROLE_ADMIN` / `ROLE_BASIC`) is only granted when the subscription is active. If an admin's subscription lapses or `subscriptionType` is null, they receive **zero authorities** and lose `ROLE_ADMIN` → locked out of the entire admin surface. This is fail-_closed_ (not an escalation), but it conflates two orthogonal concerns — entitlement (subscription) and identity (role) — in one mapping.

**Impact:** Authorization fragility and surprising lockouts; any future `@PreAuthorize("hasRole('BASIC')")` would also silently fail for users whose subscription is inactive.

**Recommendation:** Always emit the identity role authority regardless of subscription state, and emit the subscription authority only when active. Keep the two independent.

### M4 — Client-controlled sort field passed straight into the Elasticsearch query (no whitelist)

`DefaultProductsFilterQueryBuilder.java:147-167`, field source `OrderTypeDTO.java:8`

```java
Sort.by(new Sort.Order(orderBy.getDirection(), orderBy.getField(), DEFAULT_NULL_HANDLING))
```

`orderBy.getField()` is a free-form `String` bound from the client JSON (`SearchProductsDTO → ProductsFilterDTO → OrderTypeDTO`) and handed to `Sort.by` with no allow-list. The direction is enum-constrained, but the field is not.

**Impact:** Bounded by Elasticsearch (no SQLi-style exfiltration), but a client can sort on arbitrary indexed fields — enabling order-based inference of non-displayed field values, and DoS/error via sorting on non-`doc_values` or unmapped fields.

**Recommendation:** Validate `orderBy.getField()` against a server-side whitelist of sortable fields (or map an enum of allowed sort keys) before building the sort.

### M5 — H2 driver on the production classpath + dead H2-console whitelist

`pom.xml:184-185` (H2 at `runtime` scope, not `test`), `SecurityConfig.java:26-35` (`WHITE_LIST_URL` lists `/h2-console`, `/h2-console/**`)

H2 ships to prod (runtime scope). Prod uses Postgres so the console is not active by default, but combined with H1's `anyRequest().permitAll()` and a whitelist constant that explicitly names `/h2-console`, any accidental enablement (`spring.h2.console.enabled`) would expose an unauthenticated DB console.

**Recommendation:** Move the H2 dependency to `test` scope. Remove the H2-console entries (see L2 — the whole `WHITE_LIST_URL` constant is dead).

---

## Low severity / hardening

- **L1 — Non-constant-time internal-token comparison.** `InternalTokenFilter.java:28` uses `token.equals(internalToken)` — timing-attackable. Use `MessageDigest.isEqual(...)` on UTF-8 bytes.

- **L2 — Dead, misleading `WHITE_LIST_URL` constant.** `SecurityConfig.java:26-35` is defined but never referenced (the chain uses hardcoded matchers). It lists `/h2-console`, `/swagger-ui`, `/v3/api-docs` as if controlled, implying intent that doesn't exist. Remove it.

- **L3 — Dev Postman impersonation backdoor.** `FirebaseAuthFilter.validateForPostmanTesting` (lines 195-232) lets a request impersonate any `userId` via `X-User` when `dev` profile + `postman.testing.active` config + `X-Postman: true`. Triple-gated and acceptable for dev, but confirm the `dev` profile can never run in prod and `postman.testing.active` cannot be flipped on in the prod configuration table.

- **L4 — Actuator/Prometheus publicly reachable.** Subset of H1, called out separately because it leaks today. Restrict `/actuator/**` to the Docker network or an internal-token rule even after the default-deny fix.

- **L5 — CORS breadth.** `SecurityConfig.java:55-67`: `allowedHeaders: "*"` with `allowCredentials: true` (acceptable with origin _patterns_, not a `*` origin); `allowedMethods` includes the non-existent method `"UPDATE"` (typo, harmless) and omits `PATCH`. `setAllowedOriginPatterns` is broader than `setAllowedOrigins` — confirm the prod `ALLOWED_CORS` value is exact origins or tightly scoped patterns, not `https://*`.

- **L6 — Swagger paths whitelisted but not served.** `WHITE_LIST_URL` and the rate-limit categorizer reference `/swagger-ui`/`/v3/api-docs`, but no `springdoc` dependency is on the classpath, so nothing is exposed. Dead config — remove with L2.

- **L7 — `Sanitizer.sanitize` silently `trim()`s all input** (`Sanitizer.java:8`), mutating values beyond "sanitize." Surprising side effect; document or drop.

- **L8 — `BotFilter` is UA-substring matching** (`BotFilter.java:56-62`) — trivially spoofed. Fine as noise reduction, not a security control; don't rely on it for anything.

---

## What's good — keep it

- **Trust boundary respected (conventions §11).** Roles, subscription, baseSite, userId come from the verified token + DB-backed `redisUserAuth` cache, never from raw client claims. `wasRegister` and provider are derived server-side.
- **Email-verification gate reads the live token claim** (`decoded.isEmailVerified()`), deliberately not the stale `emailVerifiedExternal` column — correct freshness reasoning (`FirebaseAuthFilter.java:98-110`).
- **Banned / pending-deletion short-circuit before any controller** (`FirebaseAuthFilter.java:92-121`).
- **Admin authorization is genuinely enforced.** All 16 admin controllers carry class- or method-level `@PreAuthorize("hasRole('ADMIN')")`; `UserRole.ROLE_ADMIN` maps to authority `"ROLE_ADMIN"` and `hasRole('ADMIN')` matches it. No admin endpoint relies on URL-level auth alone. (`DefaultCurrentUserService.isCurrentUserAdmin` adds a consistent secondary check.)
- **Stateless + CSRF disabled** — correct for a bearer-token API.
- **Distributed rate limiting** (Bucket4j/Redis) with sensible per-category bandwidths and user → device → IP keying; `CF-Connecting-IP` + `forward-headers-strategy: framework` resolve client IP correctly behind Cloudflare. Uncovered endpoints cost zero Redis.
- **Secrets externalized in prod** (env / docker secrets; Firebase via `/run/secrets`). Actuator restricted to `health,info,prometheus` with `show-details: never` and an explicit "never `*`" comment.
- **CORS origin-restricted in prod** (config-driven), using origin patterns with credentials rather than a wildcard origin.
- **No SQL injection, open redirect, SSRF, or unsafe deserialization found.** All `@Query`/native queries use bound parameters; ES text queries use the typed builder API; the only `Path.of`/outbound-URL construction is over config values, not user input; `ObjectMapper.readValue` is used only on DB-sourced data or to a non-polymorphic `Map`.

---

## Suggested order of work

1. **H1** — flip to default-deny authorization; lock down `/actuator/**`. (Highest impact; subsumes M5/L4.)
2. **H2** — remove admin-by-email; assign admin out-of-band.
3. **M2** — fix duplicate filter registration (perf + correctness).
4. **M1** — decide the input-sanitization strategy (likely remove, move to output encoding).
5. **M3 / M4** — decouple role from subscription; whitelist the sort field.
6. **L1–L8** — hardening and dead-config cleanup.

Items H1, H2, M1, and M3 change behavior across the whole surface and read like they each deserve a proper brief rather than an ad-hoc patch.

---

## Config-file impact

This audit surfaces no required edit to `conventions.md`, `decisions.md`, `state.md`, or `issues.md` on its own. If Igor decides to act on H1/H2/M1, several of these would be worth recording as entries in `issues.md` (backlog) and possibly `decisions.md` (e.g. the default-deny posture, the admin-bootstrap mechanism) — but that is Docs/QA's write, not mine. Flagging here for routing.
