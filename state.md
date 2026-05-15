# State

The single source of truth for "where are we." Mastermind reads this at the start of every planning session. Engineers read the relevant slice. Docs/QA keeps it current.

**Last updated:** 2026-05-15
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

**Tasks remaining:** 10 owner-dashboard page topics; feature/flow topic triage and authoring.

---

## Bug queue

Tracked separately from feature work. Bugs are discovered during engineer sessions, code review, or QA, and logged here. A separate Mastermind chat handles bug planning and brief authoring.

**Source of truth for bug details:** [`issues.md`](issues.md). This section is the planning view — what's queued, what's being worked on, what's parked. Full descriptions, evidence, and fix history live in `issues.md`.

### Bug statuses

- `queued` — logged, not yet scoped
- `scoped` — brief drafted, ready to run when an engineer slot opens
- `in-progress` — engineer session active
- `fixed` — code shipped, verified
- `parked` — too big or wrong moment; revisit later (note why)
- `wontfix` — decided not to address (note why)

### Active bug queue

| Bug                                                    | Status | Severity | Repo | Notes |
| ------------------------------------------------------ | ------ | -------- | ---- | ----- |
| _(empty — populate as bugs are logged in `issues.md`)_ |        |          |      |       |

### How bug work runs

Bug sessions run in parallel with feature work, in free time between feature briefs. Each bug session:

1. Mastermind (bug chat) drafts a brief from the `issues.md` entry.
2. Engineer runs the session, produces `.agent/last-session.md`.
3. Igor reviews, commits, updates the `issues.md` entry status to `fixed` (or amends with what was learned).
4. Mastermind (bug chat) updates the row in this table.

If a bug turns out to be larger than a single focused session (architectural change, cross-repo coordination, security audit), it is moved out of the bug queue and into the feature pipeline above as its own `features/<slug>.md`.

---

## Backlog

Each becomes its own `features/<slug>.md` when it goes active.

| Feature                         | Status    | Notes                                                                               |
| ------------------------------- | --------- | ----------------------------------------------------------------------------------- |
| Image pipeline (general)        | `planned` | Backend + web mostly done. Expo has to adopt. Some overlap with Model C work above. |
| Backend calls reduction         | `planned` | Done on web. Expo has to adopt. Likely a small adapter task.                        |
| Firestore rules tightening      | `planned` | Security review and rule edits. Requires explicit Mastermind sign-off.              |
| Chat & messaging cleanup        | `planned` | Needs audit before scoping.                                                         |
| User deletion                   | `planned` | Igor has extensive documentation. GDPR considerations for Croatia (EU). When implemented, the deletion path MUST evict `redisUserAuth` for the deleted user's `firebaseUid` (and the matching `redisUserInfo` entries) — confirmed requirement from the connection-pool chat's auth-TTL investigation; with the TTL now at 30min, a missed eviction means 30min of stale auth on a deleted user. |
| Privacy Policy + Terms (drafts) | `planned` | Docs/QA writes pre-lawyer drafts based on Igor's intake.                            |
| Connection-pool & cache hardening | `planned` | Structural follow-ups from the connection-pool incident: per-key single-flight dedup on concurrent cache misses, removing `@Transactional` from `getAllBaseSites`, the `redisBaseSites` 24h-TTL cold-cache trigger (warmup-gating closed the post-deploy window only), `RateLimitFilter` categorisation of public reference-data endpoints, and explicit `@Order` on the four `@Component` filters. See [features/connection-pool-hardening.md](features/connection-pool-hardening.md). |
| Account-disabling & token-revocation enforcement | `planned` | Backend `FirebaseAuthFilter` should read `authData.disabled()` (currently fetched into the cache but never consulted) and the `verifyIdToken(checkRevoked=true)` trade-off — plus the cross-repo frontend rejection-handling the backend change forces. Routed to feature because the one-line backend change has a real web footprint. See [features/account-disabling-enforcement.md](features/account-disabling-enforcement.md). |

---

## Decisions log

Decisions live in [decisions.md](decisions.md). Reference only.

---

## Session log

Most recent engineer sessions, newest first. Full archives in `sessions/`.

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

- **Mobile is multiple features behind.** Once backend and web are stable on validation, mobile starts accumulating adapter work. Plan: batch mobile catch-up into a focused session after web hits `web-stable` on validation. Product-validation mobile chat opens next; other features (image pipeline, backend-calls reduction) queue behind it.
- **Wizard abandonment rate unknown.** Model C is correct without this data, but if production shows < 10% abandonment, Model A would have been cheaper. Worth measuring once launched.
- **Formal QA regression pass not yet run.** Igor has completed a manual end-to-end pass against a live local stack for product-validation. The formal QA regression milestone (scheduled in 1–2 weeks) is the remaining gate before launch.

---

## Future work (post-launch)

Things deliberately not done now, to revisit later.

- **Consolidate all developer docs into `oglasino-docs/`.** Currently `oglasino-backend/docs/` and `oglasino-web/docs/` retain repo-internal "how to work in this repo" docs. Long-term goal is single source of truth. See `conventions.md` Part 1 for the rule that prevents new docs from growing there.
- **Migrate `oglasino-docs/` to Jira or Confluence.** All docs use kebab-case filenames and relative links specifically to make this migration trivial.
- **Content moderation v2.** Policy-scoring engine. See `future/content-moderation-v2.md`.
