# State

The single source of truth for "where are we." Mastermind reads this at the start of every planning session. Engineer agents read the relevant slice. Docs/QA keeps it current.

**Last updated:** 2026-05-19
**Target:** regression testing in 1–2 weeks, production in 1 month

---

## Feature pipeline

### Feature statuses

- `planned` — spec exists, no engineering started
- `in-progress-backend` — backend work active or required next
- `backend-stable` — backend done, contract frozen, web can adopt
- `in-progress-web` — web work active
- `web-stable` — web done, mobile can adopt
- `in-progress-mobile` — mobile work active
- `mobile-stable` — all three platforms shipped to feature branch
- `shipped` — merged to main, in production (or staging if pre-launch)
- `parked` — deliberately deferred

---

## Active feature

### Product validation

- **Spec:** [features/product-validation.md](features/product-validation.md)
- **Status:** `web-stable`
- **Branch:** `feature/validation-refactor`
- **Why active:** backend and web are complete and manually tested by Igor. Remaining work is mobile (Expo) adoption, which is a separate Mastermind chat opened after merge.

**Tasks remaining:** mobile adoption only. The mobile chat starts from the `## Platform adoption` section of [features/product-validation.md](features/product-validation.md) — the contract is frozen, mobile conforms. No backend or web work is expected; if mobile audit surfaces a gap on either side, it is escalated as a new brief, not folded into the mobile session.

### QA Preparation

- **Spec:** [features/qa-preparation.md](features/qa-preparation.md)
- **Status:** `in-progress-web`
- **Branch:** `feature/qa-preparation`
- **Why active:** schema, renderer, and 12 portal page topics are done and approved by Mastermind. Remaining work: 10 owner-dashboard page topics, then portal feature/flow topics. Continuing in a fresh Mastermind chat after handoff.

**Tasks remaining:** 10 owner-dashboard page topics (authored at the simplified voice from the start); 1 deferred feature/flow topic (`report-flow`).

### User Deletion

- **Spec:** [features/user-deletion.md](features/user-deletion.md)
- **Manual test cases:** [features/user-deletion-test-cases.md](features/user-deletion-test-cases.md)
- **Status:** `shipped` (code) / `verifying` (manual smoke pending on full surface)
- **Branch:** `feature/user-deletion`
- **Why active:** code-complete across backend (13 sessions, 487 tests passing) and web (14 sessions, 154 tests passing). Self-service deletion with 7-day grace period, scheduled hard-delete cron, Firebase orphan reconciliation, ban-with-reason flow, 12-month re-registration prevention, banned-user content hiding, backend → web cache revalidation, and admin actions audit logging all live. Folded in the prior "Account-disabling & token-revocation enforcement" backlog item. Backlog-triage closed (21 of 21 items addressed; one item — B19 mobile parity — deferred to its own Mastermind chat post-merge).

**Tasks remaining:** manual smoke pending on full surface (blocked on messaging fix — see [issues.md](issues.md) 2026-05-19 entry). Lawyer review of legal drafts pending. Native-translator review of placeholder RS/CNR/RU translations pending. Mobile: queues into the Expo backlog as a per-feature `oglasino-expo-user-deletion-<n>` chat post-merge.

---

## Backlog

Each becomes its own `features/<slug>.md` when it goes active.

| Feature                                          | Status    | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Image pipeline (general)                         | `planned` | Backend + web mostly done. Expo has to adopt. Some overlap with Model C work above.                                                                                                                                                                                                                                                                                                                                                                                                     |
| Backend calls reduction                          | `planned` | Done on web. Expo has to adopt. Likely a small adapter task.                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Firestore rules tightening                       | `planned` | Security review and rule edits. Requires explicit Mastermind sign-off.                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Chat & messaging cleanup                         | `planned` | Needs audit before scoping.                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Privacy Policy + Terms (drafts)                  | `drafted` | Pre-lawyer drafts in `legal/privacy-policy-draft.md` and `legal/terms-of-use-draft.md`. Handoff package in `legal/lawyer-handoff.md`. Pending lawyer review and finalization before launch.                                                                                                                                                                                                                                                                                             |
| Connection-pool & cache hardening                | `planned` | Structural follow-ups from the connection-pool incident: per-key single-flight dedup on concurrent cache misses, removing `@Transactional` from `getAllBaseSites`, the `redisBaseSites` 24h-TTL cold-cache trigger (warmup-gating closed the post-deploy window only), `RateLimitFilter` categorisation of public reference-data endpoints, and explicit `@Order` on the four `@Component` filters. See [features/connection-pool-hardening.md](features/connection-pool-hardening.md). |

---

## Expo backlog

Tracks every feature that is `web-stable` or `shipped` on backend/web but not yet adopted on mobile. Docs/QA maintains it per the 2026-05-17 decision: append a row when a feature reaches `web-stable` or `shipped`; update or remove the row when mobile adopts the feature and its slug reaches `mobile-stable` on the feature spec.

Mobile adoption sessions are slugged per feature, not per time window. A session adopting `<feature-slug>` is `oglasino-expo-<feature-slug>-1`, `-2`, etc.

| Feature                                                                  | Web/Backend status | Mobile status | Adopted in session | Notes                                                                                                                                      |
| ------------------------------------------------------------------------ | ------------------ | ------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| [Product validation](features/product-validation.md)                     | `web-stable`       | `not-started` |                    | Active feature; mobile adoption is the only remaining work. Contract is frozen in the spec's `## Platform adoption` section.               |
| [Product filtering and search](features/product-filtering-and-search.md) | `shipped`          | `not-started` |                    | Reference-implementation spec capturing the audited web reality; spec lacks a `## Platform adoption` section (flagged to Mastermind).      |
| [User Deletion](features/user-deletion.md)                               | `shipped`          | `not-started` |                    | Code-complete on backend + web (2026-05-19). Mobile adoption opens its own `oglasino-expo-user-deletion-<n>` chat post-merge per the 2026-05-17 slug decision. |
| Image pipeline (general)                                                 | `web-stable`       | `not-started` |                    | No feature spec yet; backlog row notes "Backend + web mostly done. Expo has to adopt." Some overlap with the Model C create-flow decision. |
| Backend calls reduction                                                  | `web-stable`       | `not-started` |                    | No feature spec yet; backlog row notes "Done on web. Expo has to adopt. Likely a small adapter task."                                      |

Mobile status values: `not-started`, `in-progress`, `adopted`.

---

## Decisions log

Decisions live in [decisions.md](decisions.md). Reference only.

---

## Session log

Most recent engineer sessions, newest first. Full archives in `sessions/`.

- **2026-05-18** — `oglasino-docs`, qa-preparation session 19: final docs cleanup brief — content fixes (home-page top-level filter clarity, free-zone two image descriptions, no stale-defect cuts found), `issues.md` line-number backfill on two 2026-05-14 catalog-slug entries plus dependency-note cross-references, Known-issue cross-reference promotions across 6 pitfalls in 5 topics (`catalog-page` slug, `product-page` Serbian tooltips, `category-navigation` Kategorije label, `follow-flow` isFollowingCurrent, `messages-page` silent-send + 15-cap), flow-topic `route` field standardization (3 topics, primary-surface-only), image-comment retrofit final audit (verified all 58 entries across 17 topics, 0 additions). Closing `decisions.md` entry drafted and applied (10 sections summarising sessions 8–19). Conventions Part 5 amended: archive-pointer stub rule (7A) + Config-file impact section formalised in the engineer template (7B). `npx tsc --noEmit` from `oglasino-web` clean.
- **2026-05-17** — Mastermind, configuration-audit close-out: cross-file audit of every CLAUDE.md, bootstrap, and the four config files surfaced 17 findings. Resolved structurally rather than as a typo sweep: Docs/QA becomes sole writer of `conventions.md`, `decisions.md`, `state.md`, `issues.md`, with session-closure gate preventing pending-draft closure (Entry 1 in `decisions.md`). Router agent acknowledged as fifth engineer agent across conventions Parts 2/3/4/8/9 and the five CLAUDE.md files (Entry 2). Mobile catch-up session slug discipline pinned to per-feature, not per time window; Expo backlog table added to `state.md` (Entry 3). Conventions Part 11 heading fixed, Part 10 reformatted to ATX headings, Part 5 `<n>` indexing disambiguated, "Config-file impact" section added to engineer session template. Four bootstraps rewritten to route config writes through Docs/QA and add closure gates. Five CLAUDE.md files updated: engineer count flipped to five, "no writes to the four config files" hard rule added, session-summary references updated, Docs/QA CLAUDE.md rewritten with new write authority. Three issues.md entries from the audit logged: `master-plan.md` content outdated (this session), plus the configuration-audit findings absorbed into the conventions rewrite itself.
- **2026-05-17** — `oglasino-backend`, bug chat closeout: six sessions on the connection-pool incident's deferred TTL question and three adjacent bugs surfaced during investigation. Two read-only investigations (`redisBaseSites`, then the other six caches plus translations) established that five reference caches have no runtime write path, and four code sessions implemented the verdicts: (1) TTL removed from `redisBaseSites`, `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`; TTL kept on `redisUserInfo` / `redisUserAuth` with documenting comments; helper signature reformed `long ttl` → `Duration ttl` (`null` = no expiry). (2) `DefaultTranslationService.updateTranslation` now refreshes the in-memory `backendTranslations` map on `BACKEND_TRANSLATIONS` edits — closes the admin-edit-not-visible-until-restart gap on push notifications and email content. (3) `DefaultUserFacade.updateCurrentUserData` evicts `redisUserInfo` after `generateUserTranslations` — removes the ordering-dependent `shortBio` freshness coupling the deferred-follow-up Javadoc warned about; Javadoc deleted. (4) `redisBaseCurrency` cache value swapped from `Currency` entity to `CurrencyDTO` — closes the latent lazy-init footgun if `Currency` ever grows a `@ManyToOne`. 360 tests passing at the end of the fourth code session (baseline 355 → +2 + 1 + 2). Closes the daily-trigger window from the 2026-05-16 production incident; structural concurrent-cache-miss fix remains in [features/connection-pool-hardening.md](features/connection-pool-hardening.md).
- **2026-05-17** — `oglasino-docs`, qa-preparation session 11: `category-navigation` topic appended (`type: 'flow'`, entry 15) — first flow topic in the feature. Covers header catalog menu + breadcrumb chain + slug routing as three surfaces, with four entry points (header menu, breadcrumbs, subcategory selector dialog, direct URL). All five content arrays used; structural-choice rationale for flow topics captured in the session "For Mastermind." Five pitfalls (one "Known issue" pitfall cross-referencing the two 2026-05-14 catalog-slug `issues.md` entries), 14 checklist items, three images. `relatedTopics`: `catalog-page`, `product-page`, `home-page`, `global-header-search`. Typecheck clean. One new `issues.md` entry — hardcoded Serbian "Kategorije iz \<X\>" breadcrumb button label (`OglasinoBreadcrumbs.tsx:81`, low). Trust-boundary check on slug routing: clean (display scope only, server-derived category IDs, identity-derived authorization).
- **2026-05-15 / 2026-05-16** — Bug-fixer chat. Ten entries closed (`parseFiltersFromQueryParams` dead `| undefined`; dead `regionAndCity` field on `createProductSchema`; favorites brittle positional narrowing on `initialProductsData`; `EMPTY_OVERVIEWS` constant duplication across 8 sites; `NumberOfViews` redundant duplicate fetch; `field-price` scroll target no-op on update page; both 2026-05-14 hydration-flash entries on `ProductFunctions` and `UserDetails`; `/admin/products/[userId]` filter chips and empty-fragment entries closed by the same admin-filter-pipeline fix; Messages "Report" dropdown inert). One entry surfaced and closed in the same chat (`/admin/products` chip clicks broken after in-app nav from disallowed admin route — same `FilterManager` deps-array root cause as the `/admin/products/[userId]` entry). One entry investigated and deferred (Terms vs Privacy JSON-LD asymmetry — body amended to reference the project-wide SEO delivery bug surfaced during the investigation). Five new entries logged for future work: project-wide SEO JSON-LD delivery (medium), report-submit endpoint trust-boundary unknown (medium-pending-audit), `useAuthResolved` adoption pending (low), `isAllowedPath()` allowlist anti-pattern (low), sitemap product-count over-fetch (low), `oglasino-web` tsconfig `strict: false` (low). Net `issues.md` change: roughly −5 entries open. Chat closes 2026-05-16.
- **2026-05-16** — Bug chat close-out: dependency audit + upgrade pass for `oglasino-backend` and `oglasino-web`. Four engineer sessions (two read-only audits, two upgrades). Web: 17 `package.json` edits (2 pins relaxed, 8 range-tightens, 7 in-major upgrades, 1 `overrides` block) + cleared 3 moderate `npm audit` findings. Backend: 4 `pom.xml` edits (`postgresql` patch, `firebase-admin` minor, AWS SDK s3 + url-connection-client paired minor). Both repos green on test/lint/format/audit post-upgrade. Five adjacent findings routed to `issues.md` (2 web, 2 backend, 1 Spring-Boot-GA tracking note). No work was promoted to feature; Spring Boot 4.1.0 GA upgrade flagged as future Mastermind feature when GA lands.
- **2026-05-16** — `oglasino-backend`, dependency-upgrade session 1: four `pom.xml` version edits (`postgresql` 42.7.10 → 42.7.11, `firebase-admin` 9.8.0 → 9.9.0, AWS SDK `s3` + `url-connection-client` 2.42.28 → 2.44.7). `spotless:check` green, compile green (3 pre-existing `-Xlint` notices), 355 tests passing. Semconv pin verdict: still load-bearing (post-mediation version unchanged at 1.41.1). Two adjacent findings routed to `issues.md`.
- **2026-05-16** — `oglasino-web`, dependency-upgrade session 1: 17 `package.json` edits applied (2 pins relaxed, 8 range-tightens, 7 in-major upgrades, 1 `overrides` block for postcss security). All 12 verification steps green: install clean, tsc clean, lint exit 0 (211 pre-existing warnings), format clean (no prettier-plugin-tailwindcss diff), 154 tests passed, 0 vulnerabilities. Two adjacent findings routed to `issues.md`.
- **2026-05-16** — `oglasino-backend`, dependency-audit session 1: read-only audit of 34 direct dep rows. Bucketed: 13 `safe-patch` (12 no-upgrade-available + 1 patch target), 18 `review-minor` (15 blocked on Spring Boot 4.1.0 / Security 7.1.0 / Servlet 6.2.0 GA; 3 actionable today), 2 `major-skipped` (Flyway 11→12 pair), 1 `pinned` (opentelemetry-semconv). Plugin-updates goal hung in Claude Code session; Igor ran it manually in terminal (~20s) and pasted output for incorporation — all plugins at latest.
- **2026-05-16** — `oglasino-web`, dependency-audit session 1: read-only audit of 62 direct dep rows. Bucketed: 50 `up-to-date` (informational bucket added by engineer), 4 `safe-patch`, 1 `safe-minor`, 3 `review-minor`, 2 `major-skipped` (eslint 9→10, typescript 5→6), 2 `pinned` (react/react-dom exact-19.2.0, no documented reason). 3 moderate-severity `npm audit` findings, all chained off a transitive postcss <8.5.10 bundled by `next`.
- **2026-05-15** — DevOps closeout: Cloudflare router worker split-flag maintenance gate propagated across backend + web deploy workflows; droplet `admin-bypass-allow.sh` script created; three-step restore runbook documented. New infra doc, secret inventory corrected, two new `issues.md` entries.
- **2026-05-15** — `oglasino-docs`, legal-drafts session 1: pre-lawyer drafts of Privacy Policy, Terms of Use, and lawyer handoff package added to `legal/`. `state.md` and `decisions.md` updated. Cross-references in existing docs synced. Five pre-launch action items + six adjacent observations routed.
- **2026-05-15** — `oglasino-docs`, qa-preparation session 7: `notifications-page` and `favorites-page` topics appended to `topics.ts` (notifications keeps all three pitfalls; favorites keeps the swallow-errors-as-empty pitfall; bugs surfaced routed to `issues.md`). Typecheck clean.
- **2026-05-14** — `oglasino-backend`, connection-pool incident: CurrentLanguageFilter Phase 2 — control-flow fix (`doFilter` moved outside `try`; downstream exceptions propagate on all routes), X-Lang 400 rule (`LANG_MISSING_OR_INVALID`) on non-allowlisted public routes with a 7-entry allowlist, plus userauth-ttl patch (`redisUserAuth` TTL 1min→30min) and stale-comment cleanup. 355 tests passing.
- **2026-05-14** — `oglasino-backend`, connection-pool incident: CurrentLanguageFilter Phase 1 (read-only investigation — route inventory, allowlist shape), redis-cleanup batch (`DefaultBaseCurrencyService` `@Lazy` self-injection, `updateTranslation` `@Transactional`, `application-dev.yaml` dead TTL line removed, `CurrentLanguageFilter` swallow now logs WARN), userauth-ttl investigation (verdict: TTL safe to raise). 346 tests passing.
- **2026-05-14** — `oglasino-backend`, connection-pool incident: connection-pool-3 docker-compose healthcheck flipped liveness→readiness + `start_period` 90s→120s; connection-pool-2 `CacheWarmupService` gates readiness via `AvailabilityChangeEvent`. 342 tests passing.
- **2026-05-14** — `oglasino-backend`, connection-pool incident: investigations — `investigation-connection-pool` (root cause: thundering-herd cold-cache miss exhausts HikariCP pool), `investigation-translations-ttl` (verdict: no TTL exists, no fix needed), plus `investigation-redis-caching` Step 2 completion (caller traces — BaseSite/Language caches confirmed not bypassed). Read-only.
- **2026-05-14** — `oglasino-docs`, qa-preparation session 6: `user-page` topic appended (`type: 'page'`), cross-references to `product-page` and `messages-page`; `UserDetails` hydration-flash defect routed to `issues.md`.
- **2026-05-14** — `oglasino-docs`, qa-preparation session 5: `product-page` topic appended — 7 pitfalls, 20 checklist items, 7 images, related topic `catalog-page`; `ProductFunctions` hydration-flash routed to `issues.md`. Typecheck clean.
- **2026-05-14** — `oglasino-docs`, qa-preparation session 4: `messages-page` topic appended (pitfalls C + E kept; A/B/D routed to `issues.md` as bugs).
- **2026-05-14** — `oglasino-docs`, qa-preparation session 3: five page topics appended in one batch — `pricing-page`, `about-page`, `free-zone-page`, `privacy-page`, `terms-page` (privacy + terms thin by design); multiple adjacent bugs surfaced and logged to `issues.md`.
- **2026-05-14** — `oglasino-docs`, qa-preparation session 2: `catalog-page` topic appended; two example topics removed; `home-page` + `catalog-page` become the authoring reference. Typecheck clean.
- **2026-05-14** — `oglasino-docs`, qa-preparation session 1: pilot — `home-page` topic appended to `topics.ts` (7 controls, 6 how-to, 7 expect, 1 pitfall, 12 checklist, 4 images). Typecheck clean.
- **2026-05-14** — `oglasino-web`, qa-preparation session 3: `QaImage.url` → `imageKey` renamed across `topics.ts`, `page.tsx`, and the feature spec; no content values changed.
- **2026-05-14** — `oglasino-web`, qa-preparation session 2: `QaTopic` schema rebuilt (type discriminator, required `overview`, five optional content arrays, conditional rendering); `/design` page rebuilt; search corpus broadened; `t.txt` deleted; two example topics included.
- **2026-05-14** — `oglasino-web`, qa-preparation session 1: read-only route inventory + `/design` structure audit. No code changes.
- **2026-05-14** — `oglasino-backend`, product-validation session 4: code-review patch (F4 fallback drift, F5 unused autowire, F6 SLF4J token, F7 keyword-stuffing duplication), `product.image.duplicate` + `product.update.fail.message` seeded, legacy `product.update.fail.{title,description}` rows deleted. 333 tests passing.
- **2026-05-14** — `oglasino-web`, product-validation session 4: F2.3/F2.4/F1.4/F2.1 fixes, price required for non-free-zone categories (manual-test bug), `ensureSystemErrorKey` removed, `free` field dropped from `NewProductRequestDTO`. 153 tests passing.
- **2026-05-13** — `oglasino-web`, product-validation session 3: pre-validate wiring, step 4 failure UX, snapshot refresh on save, image MIME allowlist, `productStepMapping.ts` helpers. 136 tests passing.
- **2026-05-13** — `oglasino-backend`, product-validation session 3: configurable thresholds in `ConfigurationService`, gibberish length-awareness, `GibberishAnalyzer` wired into description chain, product `ERRORS` translations moved from `0002` to `0001` files, RU + CNR translations added, split repeating-chars threshold (description gets its own key). 333 tests passing.
- **2026-05-13** — `oglasino-web`, product-validation session 2: adopted unified wire shape, introduced `ProductEditState` view-model, removed dead code (`codeToTranslationKey`, `oldName`/`oldDescription` from wire DTO), namespace discriminator landed (later removed in session 4). 90 tests passing.
- **2026-05-13** — `oglasino-backend`, product-validation session 2: trust-boundary fixes (`oldName`/`oldDescription` removed from `UpdateProductRequestDTO`, `MASSIVE_CHANGE` moved server-side, `MassiveNameChangeValidator` deleted), no-op short-circuit, `FILTER_OPTION_NOT_IN_FILTER` enforced, severity-precedence status routing. 305 tests passing.
- **2026-05-13** — `oglasino-web`, product-validation session 1: read-only audit. Surfaced `oldName`/`oldDescription` trust-boundary violation and namespace mismatch on update page.
- **2026-05-13** — `oglasino-backend`, product-validation session 1: `ProductErrorCode` rewritten with `translationKey` + `httpStatus` per constant, unified wire shape across all status codes, `RateLimitFilter` adopts the unified body, EN + SR translations seeded for all 41 codes (RU + CNR followed in session 3). 292 tests passing.

---

## Risk watch

Tracked across sessions. Updated when status changes.

- **Mobile is multiple features behind.** Once backend and web are stable on validation, mobile starts accumulating adapter work. Plan: batch mobile catch-up into a focused session after web hits `web-stable` on validation. Product-validation mobile chat opens next; other features (image pipeline, backend-calls reduction) queue behind it. Tracked in detail in the new Expo backlog section above.
- **Wizard abandonment rate unknown.** Model C is correct without this data, but if production shows < 10% abandonment, Model A would have been cheaper. Worth measuring once launched.
- **Formal QA regression pass not yet run.** Igor has completed a manual end-to-end pass against a live local stack for product-validation. The formal QA regression milestone (scheduled in 1–2 weeks) is the remaining gate before launch.
- **`master-plan.md` content is outdated.** Referenced from `meta/conventions.md` Part 1 ("Status indicators") and `meta/conventions.md` Part 1 ("Long-term goal: one source of truth"). The file exists but its content has not been kept current; some sections reference a project structure that has moved on. Surfaced during the 2026-05-17 configuration audit. Logged separately in `issues.md`. Fix scope: a focused Docs/QA session to audit the file's content against the current state of the project and either update it section-by-section or rewrite the parts that are dead.
- **`backendTranslations` field concurrency (accepted-and-known).** `DefaultTranslationService.backendTranslations` is a non-volatile reference, rebuilt by full reassignment. As of 2026-05-17 it is reassigned more frequently than before (on every `BACKEND_TRANSLATIONS` admin edit, not just at boot). The full-reassignment style means readers see either the old map or the new — no torn map — but there is no formal memory-model guarantee without `volatile` or `AtomicReference`. Pre-existing, low severity, accepted as-is. Fix when next touching the file: one keyword (`volatile`) or wrap in `AtomicReference<Map<String, Map<String, String>>>`. File path: `src/main/java/com/memento/tech/oglasino/service/impl/DefaultTranslationService.java:42` (in `oglasino-backend`).
- **Dead `baseCurrency` variable in `PriceQueryGenerator` (accepted-and-known).** `PriceQueryGenerator.getPriceRangeQuery2` has a dead `var baseCurrency = baseCurrencyService.getBaseCurrency()` (assigned, never read). One wasted Redis cache hit per filter query; no correctness impact. Pre-existing, very low severity. Fix in passing the next time anyone touches `PriceQueryGenerator`. File path: `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/PriceQueryGenerator.java:68` (in `oglasino-backend`).
- **`CatalogDTO.categories` and `CatalogDTO.orderTypes` perpetually null on BaseSite cache path (accepted-and-known).** `CatalogDTO` declares `categories: List<CategoryDTO>` and `orderTypes: List<OrderTypeDTO>`, but the `Catalog` entity has no matching fields, so `modelMapper.map(cat, CatalogDTO.class)` at `DefaultBaseSiteService:106` leaves them null on every `BaseSiteDTO.catalog`. Either populated elsewhere by a separate hydration path (fine) or stale leftover fields. Low severity — today's clients do not trust these fields off this payload. Worth a five-minute check during the connection-pool-hardening feature work. Files: `src/main/java/com/memento/tech/oglasino/dto/CatalogDTO.java` and `src/main/java/com/memento/tech/oglasino/entity/Catalog.java` (in `oglasino-backend`).

---

## Future work (post-launch)

Things deliberately not done now, to revisit later.

- **Consolidate all developer docs into `oglasino-docs/`.** Currently `oglasino-backend/docs/` and `oglasino-web/docs/` retain repo-internal "how to work in this repo" docs. Long-term goal is single source of truth. See `conventions.md` Part 1 for the rule that prevents new docs from growing there.
- **Migrate `oglasino-docs/` to Jira or Confluence.** All docs use kebab-case filenames and relative links specifically to make this migration trivial.
- **Content moderation v2.** Policy-scoring engine. See `future/content-moderation-v2.md`.
