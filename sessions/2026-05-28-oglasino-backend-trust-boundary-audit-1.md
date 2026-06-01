# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** Audit brief — report-submit trust-boundary + baseSite tenant scoping (read-only)

## Implemented

Read-only audit of two `issues.md` entries:

1. **Report-submit endpoint trust-boundary** — traced `POST /api/secure/report/add` end-to-end. Reporter identity is server-derived (Part 11 compliant). Target identity (`reportedUserId` / `reportedProductId`) is taken verbatim from the client body with no application-level validation, no self-report block, no participation check. DB-level FK is the only protection on `reportedUserId`; `reportedProductId` has no FK at all. **Verdict: violation.**

2. **baseSite tenant scoping** — traced `BaseSiteFilter` → `BaseSiteCacheService` → `BaseSiteContext` → `BaseSiteQueryGenerator`. The `X-Base-Site` header is validated against `baseSiteRepository.findActiveBaseSiteCodes()` (unknown codes throw); a client cannot inject a fake base site. The header is used only as an Elasticsearch query-scoping filter on public/portal data — never as input to a moderation, authorization, or state-transition decision. The `Product.baseSite` persisted at create time is `user.baseSite` (server-derived), not the header. The Part 11 `OglasinoAuthentication.baseSiteId` has exactly one downstream consumer, and it uses the field as a "user setup complete" gate. **Verdict: clean.**

---

## Part 1 — Report-submit endpoint trust-boundary

### Files

- Controller: `src/main/java/com/memento/tech/oglasino/controller/ReportController.java` (29 lines)
- Facade: `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultReportFacade.java` (47 lines)
- Service: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultReportService.java` (58 lines)
- Request DTO: `src/main/java/com/memento/tech/oglasino/dto/ReportRequestDTO.java`
- Entity: `src/main/java/com/memento/tech/oglasino/entity/Report.java`
- Schema: `src/main/resources/db/migration/V1__init_schema.sql:460-478` (table), `:1371-1376`, `:1715-1720`, `:1967-1975` (FKs)

### Request DTO

`ReportRequestDTO` carries five client-supplied fields with **no validation annotations**:

| Field                | Type           | Server validates? |
| -------------------- | -------------- | ----------------- |
| `reportedUserId`     | `Long`         | DB FK only (see below) |
| `reportedProductId`  | `Long`         | **No** — column is a plain `bigint` with no FK |
| `description`        | `String`       | DB `NOT NULL` only → 500 on null |
| `reportOption`       | `ReportOption` | Jackson deserialise + DB enum check |
| `reportType`         | `ReportType`   | Jackson deserialise + DB enum check |

`ReportController.createReport` accepts the body without `@Valid`; the only DTO-shape enforcement is whatever `IllegalStateException` the service throws (which surfaces as a 500, not a 4xx with a code per Part 7).

### Identity chain (Part 11)

| Identity | Source | Compliant? |
| -------- | ------ | ---------- |
| Reporter | `currentUserService.getCurrentUserIdStrict()` → `SecurityContextHolder` → `OglasinoAuthentication.userId` (set by `FirebaseAuthFilter`) | ✓ server-derived |
| Reported user | Verbatim from `reportRequest.getReportedUserId()`; passed to `entityManager.getReference(User.class, id)` | ✗ client-supplied, application-trusted |
| Reported product | Verbatim from `reportRequest.getReportedProductId()`; written straight to the `Long` column | ✗ client-supplied, application-trusted |

### (a) Does the target id resolve to a real user/product?

**Reported user — DB FK only.** `DefaultReportService:40` calls `entityManager.getReference(User.class, reportedUserId)`, which returns a lazy proxy without hitting the DB. The reference is attached to `report.reportedUser` and saved. The `report.reported_user_id` column has FK `fk2fm8nu7yscahr6sbhhgw082mp` to `users(id)` (`V1__init_schema.sql:1371-1376`), so an INSERT with a non-existent id is rejected at flush time by Postgres. That rejection surfaces as an uncaught `DataIntegrityViolationException` — a 500 to the client, not a coded 4xx. Application code does not look up the user or check `deletion_status`, `disabled`, or any other state.

**Reported product — no enforcement at all.** `Report.reportedProductId` is mapped as `@Column private Long reportedProductId` (no `@ManyToOne`, no FK). `V1__init_schema.sql:468` declares the column as `reported_product_id bigint` with no constraint. Any client-supplied long is persisted verbatim. The application does not check whether a product with that id exists.

There is also a duplicate `if` block at `DefaultReportService:34-37` that re-checks the same `ReportType.PRODUCT && reportedProductId == null` condition already tested at `:26-28` — dead code, no functional effect.

### (b) Is the reporter authorized to report this target?

**No authorization check of any kind.** The service writes the report row if (i) the reporter is authenticated and (ii) the DTO's required-field shape passes. There is no participation check (e.g., must share a chat), no viewer check, no self-report block (a user can submit a report with `reportedUserId = currentUserId`), no check that the reporter is not banned (banned reporters would fail `FirebaseAuthFilter` first, so this is academic), and no check that the reported user is not soft-deleted (`deletion_status = 'PENDING_DELETION'`).

### Rate limiting / duplicate guard

`DefaultReportFacade:29-43` implements a Redis-backed dedupe:

- Key: `report:user:<currentUserId>` (the **reporter**'s id only — not `(reporter, target)`)
- TTL: 24 hours
- If present → `createReport` returns `false` → `ReportController:27` emits **406 Not Acceptable**

This is what the web `reportService.alreadyReported` semantics map to. Two consequences:

1. A reporter can submit **one** report every 24 hours, full stop — across all targets. If they report user A on Monday, they cannot report a genuinely different bad user B until Tuesday.
2. The web label "already reported" is **misleading**: the key is keyed on the reporter, not on `(reporter, target)`. A user who clicks Report on user B after reporting user A in the same 24-hour window sees "already reported" against B, even though they never reported B.

`RateLimitFilter` does not list `/secure/report/add` as a categorised endpoint — consistent with [[feedback_rate_limiting_scope]] (narrow on purpose; no default catch-all).

### Trust-boundary verdict: **violation**

Per conventions Part 11, values used in state-transition decisions (a `Report` row is a state transition — it enters the moderation queue and can drive admin actions including bans) must be derived from the authenticated identity, read from the server's database, or be foreign keys the server validates against its own data. `reportedUserId` is a client-supplied foreign key with **no application-side validation**, and `reportedProductId` is not even DB-validated. The reporter identity is correctly server-derived (`currentUserService.getCurrentUserIdStrict()`), so the (b)-side of the trust check is **half-clean**: the actor is trustworthy, but no policy gates which target the actor may act against.

**Concrete abuse vector.** Registration is open and Firebase-Auth-only — there is no server-side throttle on account creation. An attacker can spin up N Firebase accounts and from each file one false report against the same target within their respective 24-hour windows. The 24-hour per-reporter cache does not stop this: it bounds one account's rate, not the population's. The moderation queue receives N legitimate-looking reports; whether that produces a wrongful ban depends on downstream moderation policy (out of audit scope).

### Recommended fix shape

Single backend brief, ≤1 session:

1. **Application-side existence checks** before persisting:
   - `userRepository.findById(reportedUserId).orElseThrow(...)` (USER type)
   - `productRepository.findById(reportedProductId).orElseThrow(...)` (PRODUCT type)
2. **Self-report block**: reject if `reporter.id == reportedUserId` for USER type, and `product.owner.id == reporter.id` for PRODUCT type.
3. **Deletion-state guard**: reject if the target user has `deletion_status = 'PENDING_DELETION'` (no business value in queueing a report against an account that's about to be hard-deleted).
4. **Per-target dedupe key**: switch the Redis key to `report:user:<reporterId>:target:<type>:<targetId>` so the cache semantics line up with the web's "already reported" UI and so a reporter can still report distinct bad actors within 24 hours.
5. **DTO validation** (Part 7 codes-only): add `@Valid` on the controller, `@NotNull` on `reportType` / `reportOption`, `@NotBlank` + `@Size` on `description`, conditional `@NotNull` on the target id per type. Replace `IllegalStateException` with `ProductValidationException`-style typed errors (or a sibling exception family) so the response is a coded 422 instead of a 500.
6. **Delete the dead `if` block** at `DefaultReportService:34-37`.

**Blast radius:**
- One backend session: ReportController + DefaultReportFacade + DefaultReportService + ReportRequestDTO + new `ReportErrorCode` enum.
- New translation seeds (Part 6 Rule 3): per-field error keys under the `ERRORS` namespace for the new codes. EN/RS/RU/CNR, inline-append to existing 0001 SQL files.
- No DTO wire-shape change (fields stay the same; only validation is added on existing fields).
- One follow-up `oglasino-web` brief: map new error codes to user-facing copy in `reportService`'s catch path; consider whether the "already reported" surface should switch to per-target semantics now that the cache key is per-target.
- One Firestore Rules-side check: not applicable (reports live in Postgres only).

---

## Part 2 — baseSite tenant scoping

### Files

- Context: `src/main/java/com/memento/tech/oglasino/context/BaseSiteContext.java` (request-scoped holder)
- Filter: `src/main/java/com/memento/tech/oglasino/filter/BaseSiteFilter.java` (reads header, populates context)
- Cache service: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteCacheService.java` (DB validation seam)
- DB service: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteService.java` (Redis-cached, sourced from `baseSiteRepository.findActiveBaseSiteCodes()`)
- Query generator: `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/BaseSiteQueryGenerator.java`
- Filter-query builder: `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java`
- Search service: `src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/DefaultProductsSearchService.java`
- Product create: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:108`, `:124`, `:205-215`
- Part 11 source: `src/main/java/com/memento/tech/oglasino/security/auth/OglasinoAuthentication.java:14`, populated by `FirebaseAuthFilter.java:103`

### How the header resolves to a `BaseSiteDTO`

`BaseSiteFilter.doFilterInternal:21-36`:

1. Skip non-`/api` paths.
2. Read `X-Base-Site` header; fall back to `?baseSite=` query param.
3. If a non-blank code is present, call `baseSiteCacheService.getBaseSiteForCode(code)` and stash the result on the request-scoped `BaseSiteContext`.

`DefaultBaseSiteCacheService.getBaseSiteForCode:35-40` does:

```java
return getAllBaseSites().stream()
    .filter(bs -> bs.getCode().equals(baseSiteCode))
    .findFirst()
    .orElseThrow();
```

`getAllBaseSites()` is sourced from `baseSiteRepository.findActiveBaseSiteCodes()` (`DefaultBaseSiteService:60`), which is a DB query. The full list and per-code DTO are Redis-cached (`@Cacheable("redisBaseSiteOverviews")`, `@Cacheable("redisBaseSite", key = "#code")`), but the cache is populated from the DB — the client cannot poison it.

**Implication:** a client-supplied `X-Base-Site` code only resolves if it matches a row returned by `findActiveBaseSiteCodes()`. An unknown code raises `NoSuchElementException` (a 500 to the caller, since it is not mapped to a code-based 422). The set of values the client can successfully spoof is therefore exactly the set of known active base sites — i.e., publicly known tenants, with publicly accessible product catalogs.

### Where the resolved base site enters queries

`BaseSiteQueryGenerator.addQuery:17-24` adds a single ES filter to the bool query:

```java
queryBuilder.filter(f -> f.term(t -> t.field("baseSiteId").value(...getId())));
```

`DefaultProductsFilterQueryBuilder.buildBaseQuery:93-120` iterates `generators` — including `BaseSiteQueryGenerator` — for every search-mode path (PORTAL_SEARCH, DASHBOARD_SEARCH, ADMIN_SEARCH). The generator is purely additive: it narrows the ES result set to documents tagged with the requested `baseSiteId`. There is no auth, no moderation, no state decision wired into the generator.

`DefaultProductsSearchService.countPortalProducts:72-75` repeats the same `baseSiteId` term filter on the `_count` path — query scoping only.

### Where the header is consumed outside ES queries

`DefaultProductService` is the only non-search consumer of `BaseSiteContext`:

- **`createProduct:108`** sets `product.setBaseSite(user.getBaseSite())` — **server-derived from the loaded `User` entity**, not from the header. A client who manipulates `X-Base-Site` cannot move their product into another tenant; the persisted tenant is whatever the user's profile says.
- **`createProduct:124`** calls `getBaseSite()` to pick a catalog/filter source. `getBaseSite():205-215` chooses `currentBaseSite` (header) only if `currentBaseSite.getDomain().equals(user.baseSite.getDomain())`; otherwise it falls back to `user.baseSite`. The chosen `BaseSiteDTO` is then used to look up `getCatalog()` / `getAllowedCurrencies()` — i.e., to decide *which catalog of filters and currencies is in scope when validating this create*. The actual `product.baseSite` was already set to `user.baseSite` two lines up.

`ProductSearchController:55-69` and `PublicProductController:43-50` explicitly null-check `baseSiteContext.getCurrentBaseSite()` and emit `BASE_SITE_MISSING_OR_INVALID` (422) when absent. No auth use.

### Where the Part 11 server-trusted `baseSiteId` is consumed

`OglasinoAuthentication.baseSiteId` (set from `AuthenticatedUserDTO.baseSiteId` at `FirebaseAuthFilter:103`) is exposed by `DefaultCurrentUserService.getCurrentUserBaseSiteId()`. Grep yields **exactly one** consumer: `DashboardProductController.canUserCreateProduct:158`, which throws `USER_SETUP_INCOMPLETE` when the value is missing. This is a "user has finished onboarding" gate, not a tenant-isolation gate.

There is therefore no code path where the server-trusted `baseSiteId` is checked against the header-derived `BaseSiteContext.currentBaseSite.id` and rejected on mismatch. The two values live in independent tracks.

### Trust-boundary verdict: **clean**

Per Part 11, the audit must check whether the header could be used to drive a moderation, authorization, or state-transition decision. It is not:

1. The header **is validated** server-side against the DB-derived list of active base sites. A client cannot inject a fake or inactive tenant code; the resolver throws.
2. The set of base-site codes the client can successfully resolve is exactly the set of publicly known active tenants. Each tenant's product catalog is publicly accessible at its own web origin already, so "switching" the header lets the caller see the other tenant's public products — which they could also see by visiting that tenant's domain. The header does not unlock any non-public data.
3. The persisted `Product.baseSite` at create time is server-derived from `user.baseSite`; the header cannot move products between tenants.
4. The header does not influence which user identity is authenticated, which role is granted, which subscription tier applies, or which moderation state a product enters.
5. The server-trusted Part 11 `baseSiteId` value is used only as a "user setup complete" gate, not a tenant isolation gate. This is consistent with the platform's design: tenants are catalog-scoped, not access-scoped.

### Adjacent observations (flagged, not in scope)

- **`getBaseSiteForCode(code)` returns 500 on unknown code.** `orElseThrow()` raises `NoSuchElementException` with no error code mapping — Part 7 says validation logic should not produce 500s. Low severity, UX-only (the path is unreachable from current frontends since they only send known codes from `BaseSiteCacheService` reads, but a curl attacker would see 500).
- **`DefaultProductService.getBaseSite():205-215` prefers the header when domains match.** This means a user whose `user.baseSite.domain` is shared by another base site (if such a configuration is ever introduced) could submit a `X-Base-Site` for the sibling base site and get *its* catalog/filters applied to validate their create — even though the persisted `product.baseSite` would still be `user.baseSite`. The two would diverge. Today no base sites share a domain (each base site is keyed by domain in `baseSiteRepository.findCoreByCode`), so the branch never triggers a mismatch in practice; it's a fragile shape rather than a current bug.
- **`BaseSiteContext` is silently ignored across the entire `/api/secure/admin/**` surface.** That's intentional today — admin search scopes by header just like portal search — but worth noting that an admin who forgets to send `X-Base-Site` falls into "no filter" mode and gets cross-tenant results. Not a security issue (admins are authorised), but a UX gotcha.

---

## Files touched

- None (read-only audit, no code modified)

## Files read (for audit)

- `src/main/java/com/memento/tech/oglasino/controller/ReportController.java`
- `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultReportFacade.java`
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultReportService.java`
- `src/main/java/com/memento/tech/oglasino/dto/ReportRequestDTO.java`
- `src/main/java/com/memento/tech/oglasino/entity/Report.java`
- `src/main/java/com/memento/tech/oglasino/context/BaseSiteContext.java`
- `src/main/java/com/memento/tech/oglasino/filter/BaseSiteFilter.java`
- `src/main/java/com/memento/tech/oglasino/service/BaseSiteCacheService.java`
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteCacheService.java`
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteService.java`
- `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/BaseSiteQueryGenerator.java`
- `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java`
- `src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/DefaultProductsSearchService.java`
- `src/main/java/com/memento/tech/oglasino/controller/PublicProductController.java`
- `src/main/java/com/memento/tech/oglasino/controller/ProductSearchController.java`
- `src/main/java/com/memento/tech/oglasino/controller/DashboardProductController.java` (excerpt around line 158)
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java` (excerpts: create path 85-215, getBaseSite 205-215)
- `src/main/java/com/memento/tech/oglasino/security/auth/OglasinoAuthentication.java`
- `src/main/java/com/memento/tech/oglasino/security/service/impl/DefaultCurrentUserService.java`
- `src/main/resources/db/migration/V1__init_schema.sql` (report table + FKs, users table)

## Tests

- Ran: nothing (audit only)
- Result: n/a
- New tests added: none

## Cleanup performed

- None needed — read-only audit, no code changes.

## Config-file impact

- **conventions.md:** no change
- **decisions.md:** no change
- **state.md:** no change
- **issues.md:** **2 entries to amend.** Drafts in "For Mastermind" below.
  - 2026-05-16 "Report-submit endpoint trust-boundary verification unknown" → finalize severity (**medium**), append audit findings, set status to `open` with audit conclusion; recommend a fix brief.
  - 2026-05-14 "`baseSite` tenant scoping was never audited" → mark as audited; append verdict (**clean**); set status to `fixed` since the audit closes the open question.

## Obsoleted by this session

- Nothing.

## Conventions check

- **Part 4 (cleanliness):** confirmed (no code touched).
- **Part 4a (simplicity):** see structured evidence in "For Mastermind." Audit-only session — no new code, no new abstractions; fix-shape recommendations sized in the recommended-fix section above.
- **Part 4b (adjacent observations):** flagged — two `baseSite` adjacent observations recorded in "For Mastermind"; report-side dead-code `if` block at `DefaultReportService:34-37` noted in Part 1.
- **Part 6 (translations):** N/A this session (no translation work).
- **Part 7 (error contract):** flagged — `DefaultReportService` throws `IllegalStateException` (raw 500) on validation gaps instead of a coded 4xx; `BaseSiteCacheService.getBaseSiteForCode` `orElseThrow()` produces a 500 on unknown codes. Recorded in audit body.
- **Part 11 (trust boundaries):** the audit itself.
- **Other parts touched:** none.

## Known gaps / TODOs

- None — the audit is complete to the brief's "definition of done" (both parts end with an explicit Part 11 verdict and, for the violation, a recommended fix shape with blast radius).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (no code added).
  - Considered and rejected: nothing — the recommended fix in Part 1 was sized in the audit body, not built. I deliberately did **not** propose a participation/visibility check ("must have viewed the user's page" or "must share a conversation") on top of the basic existence + self-report + per-target dedupe checks: the existence/self-report/dedupe trio closes the abuse vector at the entry point, and a participation check adds friction without adding meaningful protection against the registration-pumping attack (the attacker can scroll a public profile too). Mastermind can override if the policy goal is stricter.
  - Simplified or removed: nothing.

- **Part 1 verdict — drafted `issues.md` amendment** (target file: `oglasino-docs/issues.md`, target entry: `2026-05-16 — Report-submit endpoint trust-boundary verification unknown`). Suggested new severity: **medium**. Suggested addendum text:

  > **Audit (2026-05-28, session `2026-05-28-oglasino-backend-trust-boundary-audit-1`):** Verdict — **violation** per conventions Part 11. Reporter identity is server-derived (`currentUserService.getCurrentUserIdStrict()` from `SecurityContextHolder`) ✓. Target identity (`reportedUserId` / `reportedProductId`) is taken verbatim from the client body with no application-level existence check, no self-report block, no participation check. `reported_user_id` is FK-protected at the DB layer (`fk2fm8nu7yscahr6sbhhgw082mp`), so a totally fake id surfaces as an uncaught `DataIntegrityViolationException` (HTTP 500). `reported_product_id` has no FK at all — any client-supplied long is persisted verbatim. The 24-hour Redis dedupe key is `report:user:<reporterId>` (reporter-only, not per-target), which bounds one account's rate but not the population's — an attacker can spin up N Firebase accounts and from each file one report against the same target. Recommended fix shape: application-level `userRepository.findById` / `productRepository.findById` lookups, self-report block, deletion-state guard, per-target dedupe key, full DTO `@Valid` + typed error codes (delete dead `if` at `DefaultReportService:34-37`). One backend session + one web brief. Audit body has the full blast-radius treatment.

- **Part 2 verdict — drafted `issues.md` amendment** (target file: `oglasino-docs/issues.md`, target entry: `2026-05-14 — \`baseSite\` tenant scoping was never audited`). Suggested status flip to **fixed**. Suggested addendum text:

  > **Audit (2026-05-28, session `2026-05-28-oglasino-backend-trust-boundary-audit-1`):** Verdict — **clean** per conventions Part 11. The `X-Base-Site` header (with `?baseSite=` fallback) is validated against the DB via `DefaultBaseSiteCacheService.getBaseSiteForCode`, which sources from `baseSiteRepository.findActiveBaseSiteCodes()` — a client cannot inject a fake or inactive tenant code; an unknown code throws. The header is consumed only as an Elasticsearch query-scoping filter on public catalog data (`BaseSiteQueryGenerator`); it never feeds an authorization, moderation, or state-transition decision. The persisted `Product.baseSite` at create time is `user.baseSite` (server-derived from the loaded `User` entity, `DefaultProductService:108`), not the header — a client cannot move products between tenants. The server-trusted Part 11 `OglasinoAuthentication.baseSiteId` has exactly one downstream consumer (`DashboardProductController.canUserCreateProduct:158`), and it uses the value as a "user setup complete" gate, not a tenant-isolation gate. Two minor adjacent observations recorded in the audit body but not promoted to separate issues: `getBaseSiteForCode` returns a 500 on unknown codes (Part 7 — codes-only) and `DefaultProductService.getBaseSite()` prefers the header when domains match (fragile shape, no current bug since base sites are unique per domain).

- **Part 4b adjacent observations** (audit-side, not fixed):
  - **Dead `if` block at `DefaultReportService:34-37`.** Duplicate of `:26-28`. Severity: low (cosmetic). Fold into the Part 1 fix brief — no separate brief needed.
  - **`BaseSiteCacheService.getBaseSiteForCode` returns 500 on unknown code.** `orElseThrow()` with no code mapping. Part 7 violation. Severity: low (UX, not security — reachable from curl, not from current frontends). File path: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultBaseSiteCacheService.java:35-40`. I did not fix this because it is out of scope.
  - **`DefaultProductService.getBaseSite():205-215` domain-match branch.** Prefers `currentBaseSite` over `user.baseSite` when domains match. Today no base sites share a domain, so the branch effectively always returns the header when it equals the user's stored base site, and otherwise falls back to the user's. If the platform ever introduces base sites that share a domain, the persisted `product.baseSite` (user-derived) and the catalog-source `baseSite` (header-derived) could diverge — the create would validate against a different catalog than the one the product belongs to. Severity: low (latent, no current trigger). File path: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:205-215`. I did not fix this because it is out of scope; flagging only.

- **Closure gate** — Both `issues.md` amendments above are drafted, not applied. They depend on a Docs/QA session being scheduled. The session does not close with a pending config-file edit per Part 5 — the drafts are explicitly here for Docs/QA to apply. No code changed in this repo, so no engineering-side close-out is needed.
