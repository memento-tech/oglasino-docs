# Backend Security Hardening

**Status:** built / pending verification
**Repos:** oglasino-backend (primary), oglasino-web (one paired fix)
**Branches:** backend `dev`, web `dev`
**Origin:** a backend security audit (2026-06-03) surfaced 3 high / 5 medium / 8 low findings. Two read-only Phase-2 revalidation audits (`sessions/audit-backend-security-hardening.md`, `sessions/audit-web-output-encoding.md`) confirmed, refuted, or recalibrated each against current code. This spec reflects the **revalidated** reality, not the original audit's severities.

A pre-regression hardening pass. Goal: close the live and latent authentication/authorization weaknesses before the regression milestone (1–2 weeks out), without breaking the infra healthcheck contract or the edge backend-availability gate.

---

## Delivery (2026-06-04)

All six Phase-5 briefs landed on `dev` (backend + web), uncommitted pending Igor's commit. Status is `built / pending verification` — code-complete but not yet stage-smoked. The remaining work is the operational close-out checklist below (§8) and Igor's stage regression smoke.

1. **Brief 1 — H1 default-deny** + `SuggestionController` normalization. `SecurityConfig` now terminates in `anyRequest().authenticated()`; the public surface is explicit and small (`OPTIONS /**`, `/actuator/health/**`, `/error`, `/health`, `/api/public/**`, `/api/auth/**`, `/internal/**`); `/actuator/prometheus` + `/actuator/info` are `denyAll`. Verified per-path by `SecurityConfigAuthorizationTest`.
2. **Brief 2 — admin out-of-band.** Admin is now a per-env seeded DB row (`data/admin/data-admin-{dev,stage,prod}.sql`) keyed on the env's Firebase UID, idempotent via `ON CONFLICT (firebase_uid) DO NOTHING`; registration is unconditionally `ROLE_BASIC`; the email-literal grant and the JSON admin seed are removed.
3. **Brief 3 — web JsonLd escape** (`oglasino-web`). `JsonLd` escapes serialized JSON-LD for the `<script>` context as its single output path (`serializeJsonLd`), closing the stored-XSS sink. Verified with the literal `</script>` attack string.
4. **Brief 4 — backend input-sanitization filter removed.** `InputSanitizationFilter` + `Sanitizer.sanitize` + the OWASP encoder dependency removed; email-render gate passed (only the listing title reaches an HTML email body, and it is `HtmlUtils.htmlEscape`'d). `Sanitizer.stripNewlines` retained (live log-injection guard).
5. **Brief 5 — M3 + M4.** `AuthUtils` always emits the identity role, gating only the subscription-tier authority on an active subscription. The client sort field is validated against the seeded `OrderType` catalog before reaching ES `Sort.by`; an unrecognized field degrades to the existing default sort.
6. **Brief 6 — cleanup.** L3 Postman impersonation path removed (incl. the `postman.testing.active` seed row); M2 duplicate filter registration disabled via `FilterRegistrationBean(false)`; M5 H2 → `test` scope; L1 constant-time internal-token compare; L2/L6 dead `WHITE_LIST_URL` deleted; L5 CORS (`UPDATE`→`PATCH`); L7/L8 no-action (L7 done by Brief 4).

---

## 1. What was revalidated (and what changed from the original audit)

The original 2026-06-03 audit over-stated four severities and mischaracterized two mechanisms. The revalidation corrected them. The spec carries the **corrected** severities:

- **M2 (duplicate filter registration)** — original audit: "two Firebase verifications per request." **Refuted.** Both `FirebaseAuthFilter` and `InternalTokenFilter` extend `OncePerRequestFilter` (`FirebaseAuthFilter.java:27`, `InternalTokenFilter.java:11`), whose dedupe guard runs each body once. The `issues.md` 2026-06-03 entry was correct. Severity: **cleanliness**, not perf.
- **M4 (client sort field)** — original: implied arbitrary-field injection. **Nuanced.** The raw client `field` does reach `Sort.by` with no allow-list (`DefaultProductsFilterQueryBuilder.java:155-159`), but it is an **Elasticsearch** sort (no SQLi), and a seeded `OrderType` catalog already exists — the request path just ignores it. Severity: **defense-in-depth / trust-boundary cleanup**.
- **H2 (admin-by-email)** — original: "admin granted to whoever registers with `admin@oglasino.com`." **Recalibrated to latent.** The branch lives only in the create-new-user path (`DefaultFirebaseAuthService.java:151-156`) and cannot re-fire for an existing row. The operator controls the `admin@oglasino.com` Firebase account, the DB row already exists, and the operator owns `oglasino.com`. So it is a **re-arming trap** (fresh-env seed, or delete-and-recreate), not a live open hole. Still fixed — no exceptions.
- **M5 (H2 on prod classpath)** — original: implied an exposed H2 console. **Nuanced.** H2 is `runtime`-scoped (ships on prod), but `spring.h2.console.enabled` is unset, so the console is **not** exposed. Live risk: unnecessary jar on the prod image, not an open console.
- **L3 (Postman impersonation)** — **elevated to medium.** The triple gate's only real prod guard is the `dev`-profile check; the `postman.testing.active` Configuration row is seeded `'true'` (`data-configuration.sql:4`), so two of three gates are already satisfied. One mis-set `SPRING_PROFILES_ACTIVE` from live user-impersonation.

## 2. The headline cross-repo seam (M1) — read this first

The backend's `InputSanitizationFilter`/`Sanitizer` HTML-encodes input (`Encode.forHtmlContent`). This is the wrong layer (corrupts non-HTML consumers, double-encodes) AND half-broken (it overrides `getReader()`/`getParameter()` but **not** `getInputStream()`, so JSON `@RequestBody` — the dominant write surface, 43 occurrences across 22 controllers — is never sanitized anyway).

But it **cannot be removed in isolation.** Two render paths currently depend on the backend `<`-encoding to stay XSS-safe:

1. **Web `JsonLd` sink (`oglasino-web/src/components/server/seo/JsonLd.tsx:14`).** Structured data is injected into a `<script type="application/ld+json">` via `JSON.stringify`, which does **not** escape `</script>`/`<`/`>`. The product, user, and catalog pages feed it raw `product.name`, `product.description`, `user.displayName`, `product.title`. A field containing `</script><script>…</script>` breaks out and executes — **stored XSS on public, unauthenticated pages**, masked today only by the backend filter.
2. **Backend email bodies (Brevo SMTP).** The backend itself emits user-supplied strings into HTML email. Those must escape at their own render step before the global input filter goes.

**Hard sequencing constraint:** the web `JsonLd` fix (escape serialized JSON for the script context) must land **before or together with** the backend filter removal. Mobile needs no action — React Native escapes by default and has no DOM.

## 3. Findings and fixes (the work)

### H1 — Default-deny authorization
`SecurityConfig.java:73-82` ends in `anyRequest().permitAll()` — allow-by-default. Only `/api/secure/**` requires auth. Everything outside the four prefixes (`/api/secure/`, `/api/public/`, `/api/auth/`, `/internal/`) is public.

Fix: flip the trailing rule to `anyRequest().authenticated()` (or `denyAll()`), then **explicitly re-allow the complete non-prefixed public surface**:

- `/actuator/health/**` (readiness + liveness) — **MUST stay open.** The docker-compose healthchecks (`infra/docker-compose.yml:93`, `infra/docker-compose-stage.yml:108-114`) hit container-internal `http://localhost:8080/actuator/health/readiness` with no token; the edge mobile-liveness probe (maintenance-split feature) hits the same. Gate this and the deploy boot-loops AND the edge backend-availability gate breaks.
- `/error` — Spring default dispatch (`BasicErrorController`, no custom `ErrorController`). Gate it and error responses deny themselves.
- `OPTIONS /**` — already permitAll; keep.
- `/health` — the only non-prefixed application controller (`HealthController.java`). Re-allow it, **or delete it** as redundant with `/actuator/health` (engineer's call; lean delete).
- `/actuator/prometheus`, `/actuator/info` — currently public, **lock down** (deny / internal-only). Do not blanket-permit `/actuator/**`.

Verify the invariant **per path** — confirm each re-allowed path responds and each newly-denied path is denied (**403**, not 401 — with `formLogin`/`httpBasic` disabled and no custom `AuthenticationEntryPoint`, Spring's default `Http403ForbiddenEntryPoint` returns 403 on anonymous denial; denied-is-denied) — not merely that the flip happened (barrier-review lesson, decisions.md 2026-05-28). Fold in the `SuggestionController` `/api` class-mapping normalization here (see §3 cleanup) — a class mapped to `/api` relying on method paths landing inside the prefixes is most dangerous exactly under default-deny.

**Cost of being wrong: highest in the feature.** A missed path either boot-loops the container or silently re-opens a hole.

### H2 — Admin role out-of-band (remove the email literal)
`DefaultFirebaseAuthService.java:151-156` grants `ROLE_ADMIN` to the literal `admin@oglasino.com` in the create path. Move the admin grant out of code — DB seed / explicit provisioning step — and have registration always create `ROLE_BASIC`. **Order: provision the admin role out-of-band first, then remove the email branch** (removing it first risks locking the admin out). Latent today (the live admin row exists, so the branch is dead for it); fixed as a re-arming trap.

### H3 — Firebase key rotation (NOT a code brief)
Tracked as `firebase-key-in-git`. Runtime path is clean (prod loads from `/run/secrets/firebase.json`); the liability is the historical committed copy. Operational rotation after prod is verified. No engineering brief — close-out checklist item only.

### M1 — Remove input HTML-sanitization, fix the render sinks (coordinated, see §2)
- **Web (gates the backend removal):** escape serialized JSON for the script context in `JsonLd` — `JSON.stringify(item).replace(/</g, '\\u003c')` or equivalent, or change the structured-data delivery off a raw `<script>`.
- **Backend:** confirm email templates escape user-supplied interpolations at their own render step (the email-notifications spec already calls for escaping the listing title — verify it is actually done), then remove `InputSanitizationFilter` + `Sanitizer`. If any input hygiene is still wanted, make it semantic (trim / normalize / strip control chars), **not** HTML-encoding.

### M2 — Disable duplicate filter registration (cleanliness)
`FirebaseAuthFilter` and `InternalTokenFilter` are `@Bean Filter`s added to the security chain AND auto-registered as servlet filters. The `OncePerRequestFilter` guard means bodies run once, so this is correctness hygiene, not perf. Add disabled `FilterRegistrationBean`s mirroring `RateLimitConfig.java:61-67`.

### M3 — Decouple identity role from subscription state
`AuthUtils.java:23-33` returns `List.of()` (zero authorities, including the identity `ROLE_*`) when `!subscriptionActive || subscriptionType == null` — so a lapsed/null-subscription admin loses `ROLE_ADMIN` and is locked out of every `@PreAuthorize` admin endpoint. Fix: emit the identity authority (`ROLE_ADMIN`/`ROLE_BASIC`) **unconditionally**; gate only the subscription-tier authority (`ROLE_<TIER>`) on `subscriptionActive`. Source values are server-derived (Part 11 clean); the bug is the coupling.

**Reachability is DB-dependent and currently unknown.** New users get an active free subscription, so it works for them by accident. The live `admin@oglasino.com` row was created by hand — if its `subscription_active=false` or `subscription_type=null`, the admin is **locked out of the admin panel right now**. The M3 brief includes a read-only step to check that one row (read-only `psql`, local, per conventions). If lapsed/null → this is a live admin lockout, fix urgently; if active → hardening. The fix is worth doing either way (it removes the dependency on that DB state entirely).

### M4 — Whitelist the sort field via the seeded catalog
`orderBy.getField()` is a free-form client `String` (`OrderTypeDTO.java:8`, no validation) reaching ES `Sort.by` with no allow-list. A seeded `OrderType` catalog exists but is ignored on the request path. Fix: resolve the client's order against the seeded catalog server-side and use the catalog's `field`/`direction`, ignoring the client `field` string. ES sort, not SQL — defense-in-depth + a trust-boundary cleanup (same class as the historical `oldName` bug, Part 11).

### Cleanup (M5 + L1–L8 + adjacent)
- **L3 (medium) — remove the Postman impersonation path entirely.** Delete `FirebaseAuthFilter.validateForPostmanTesting` and its call site; remove the `postman.testing.active` config row. (Operator decision: removed, not kept-and-hardened — the dev workflow does not depend on it.)
- **M5** — move `com.h2database:h2` from `runtime` to `test` scope (`pom.xml:183-187`).
- **L1** — `InternalTokenFilter.java:28` constant-time compare via `MessageDigest.isEqual` on UTF-8 bytes.
- **L2/L6** — delete the dead, unreferenced `WHITE_LIST_URL` constant (`SecurityConfig.java:26-35`); it lists h2-console + swagger paths for dependencies not on the classpath.
- **L5** — CORS (`SecurityConfig.java:61-62`): drop the non-existent method `"UPDATE"`, add `PATCH`; confirm prod `ALLOWED_CORS` holds exact `oglasino.com` origins (not broad patterns) given `allowCredentials:true`.
- **L7** — `Sanitizer.sanitize` trims silently (moot once M1 removes it).
- **L8** — `BotFilter` is UA-substring matching; document it as noise-reduction, not a security control. No code change required.
- **Adjacent** — normalize `SuggestionController`'s `/api` class mapping to explicit `/api/public/...` / `/api/secure/...` (folded into H1, §3). `controller/test/TestCreateJSON.java` is `@Profile("dev")` — harmless, leave or note.

## 4. Engineering brief sequence (Phase 5)

Six briefs, grouped by what changes together. Brief 3 **gates** Brief 4.

1. **Brief 1 — H1 default-deny** + `SuggestionController` normalization. Per-path verification. Highest stakes.
2. **Brief 2 — H2 admin provisioning** (out-of-band grant first, remove email branch second).
3. **Brief 3 — web `JsonLd` script-context escape** (`oglasino-web`). **Gates Brief 4.**
4. **Brief 4 — backend M1 removal** (verify email escaping, then remove the filter). Runs only after Brief 3 lands.
5. **Brief 5 — M3 + M4** (decouple role from subscription — includes the read-only admin-row subscription check; whitelist sort via the seeded catalog).
6. **Brief 6 — cleanup** (M2, M5, L1–L8, L3-removal).

H3 is not a brief — operational rotation, close-out checklist item.

## 5. Trust boundary (Part 11)

Auth identity is server-derived and clean: `FirebaseAuthFilter` verifies the token, loads `AuthenticatedUserDTO` from `redisUserAuth` (DB-backed), populates `OglasinoAuthentication`; subscription/role/baseSite come from the DB, not token claims; the email-verification gate reads the live token claim. Two findings are trust-boundary smells of the `oldName` class: **M4** (client `field` used in a sort decision without server re-derivation) and the general principle behind **M1** (push XSS encoding to the actual HTML render points, not the input layer).

## 6. Definition of done

- All six briefs APPROVED; backend `./mvnw spotless:check` + `./mvnw test` green; web lint/tsc/test green.
- H1 verified per-path: each re-allowed path responds, each newly-denied path is denied (**403** via `Http403ForbiddenEntryPoint`, not 401), the docker healthcheck does not boot-loop (Igor smoke against a local stack).
- M1: web fix landed before/with backend removal; no raw-HTML sink carries backend strings unescaped.
- M3: admin-row subscription columns checked; if lapsed/null, admin-panel access re-verified after the fix.
- L3 impersonation path removed; `postman.testing.active` row gone.
- Docs in sync; config-file edits applied by Docs/QA.

## 7. Out of scope

The `MarkdownViewer` unsanitized `rehype-raw` surface (independent of this feature — operator-controlled GitHub source, never touches the backend; tracked separately if Igor wants it in `issues.md`); the `DefaultCloudflareKvService` no-timeout `RestTemplate` (already in `issues.md` 2026-06-03). Firebase key rotation (H3) is operational follow-up, not out of scope — it moves to the close-out checklist below (§8).

## 8. Close-out checklist (operational, not code)

These are **not** code and were never briefs. They survive feature close as operator follow-ups — each is Igor's to verify, post-deploy or pre-prod. None blocks the `built / pending verification` → verified transition except as noted.

1. **H3 — Firebase service-account key rotation.** The runtime path is clean (prod loads from `/run/secrets/firebase.json`); rotate the historical committed key after prod is verified. This is the pre-existing `firebase-key-in-git` item — track/close it there, not as a duplicate.
2. **Router origin-bypass firewall check.** Confirm the backend origin host (`api-origin.oglasino.com` / `-stage`) is reachable **only** through Cloudflare — the droplet firewall accepts traffic only from Cloudflare IP ranges (or the origin is orange-clouded and IP-locked). If the origin host is publicly resolvable and not IP-locked, the entire edge (rate-limit, `CF-Connecting-IP`, and this feature's default-deny work) can be bypassed. An infra/DNS check, not code; surfaced by the router edge-reachability audit.
3. **CORS `ALLOWED_CORS` value.** Confirm the prod `ALLOWED_CORS` env value holds exact `oglasino.com` origins (not broad `https://*`-style patterns), given `allowCredentials: true`.
4. **Stage regression smoke across all six briefs — with the M1 deployment-ordering gate.** The web JsonLd fix (Brief 3) must be live in the target environment **before or with** the backend filter removal (Brief 4), or stored XSS reopens for the gap. Smoke specifics from the briefs — notably **M4: click each sort option (newest, price asc/desc) and confirm results actually reorder**, not merely that search returns without a 500 (guards against a web/backend sort-field name mismatch, e.g. the app posting `price` vs the seeded `basePrice`).
