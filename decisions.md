# Decisions

Append-only log of decisions made across planning sessions. Igor commits entries. Mastermind drafts them. Engineers read but don't write.

Format: each decision has a date, a one-line summary, the reasoning, and the alternatives we considered. Newest at the top.

---

## 2026-05-19 — Backend → web cache revalidation: best-effort fire-and-forget pattern

Backend POSTs `{"tags": ["user:<id>"]}` to web `/api/revalidate` after a user state transition commits, so other viewers of the affected profile see fresh `UserInfoDTO` inside the 5-minute SSR `revalidate: 300` window. Implemented via `UserStateChangedEvent` + `@TransactionalEventListener(AFTER_COMMIT, fallbackExecution = true)` + `DefaultWebRevalidationService`. Failure is logged and swallowed. Shared-secret header authentication (Option A) chosen over IP allowlist (Option B) or no-auth (Option C) because shared-secret is the simplest defensible model for server-to-server inside a hosted environment.

**Reasoning.** Frontend-driven revalidation (the existing self-deletion / self-restoration flow) covers the actor-side cache; the missing piece was viewer-side caches when *another* user is the actor (admin bans, hard deletes, scheduled deletions). Best-effort is correct because the 5-minute TTL is the upper bound on staleness — a failed revalidate is recoverable without operator action.

**Alternatives considered.** Explicit post-commit method calls (Option B per the brief) — rejected because every new state-transition path would need to remember to call the client; event publication composes with future paths without code edits at the consumer.

---

## 2026-05-19 — Banned-state UI visibility design

When a user is banned (`disabled = true`), the system mirrors the existing pending-deletion content-hiding posture: products flip to `INACTIVE` with `deactivated_by_system = true`, public profile returns 404, messaging is gated on the counterparty side, and the `UserInfoDTO.state` field returns `BANNED`. Banned wins over pending-deletion in the composition rule (`UserState.resolve(disabled, deletionStatus)`); admin unban during pending-deletion does NOT restore products (only when `deletionStatus = ACTIVE`).

**Reasoning.** The pre-B2 design banned a user without hiding their content — meaning banned-users' listings stayed reachable via direct URL, and counterparty messaging stayed enabled. Symmetric content-hiding closes the privacy gap while preserving the pending-deletion grace period semantics (unban doesn't auto-restore deletion intent).

---

## 2026-05-19 — Backlog-triage closed for User Deletion feature

Round-3 handoff identified 21 backlog items (B1–B21). This pass closed: B1 (push-token detach), B2 (banned-state UI), B3 (auth-listener consolidation, B3-a path), B6 (filename typo), B8 (cookie-write conditional), B10 (cache invalidation), B11 (InfoDialog dismissal), B12 (cross-browser ban catch), B14 (audit log for unban), B15 (dead code in UsersFilterRequestDTO), B16 (Javadoc/impl mismatch), B17 (per-row lock query N+1), B18 (projection cardinality smell), B20 (admin Users detail page state indicators). One item out of scope: B19 (mobile parity — separate Mastermind chat post-merge). Closed via 9 backend sessions + 6 web sessions on `dev` branch.

---

## 2026-05-19 — Cache-invalidation profile fix (Next.js 16)

The `/api/revalidate` route handler used `revalidateTag(tag, 'default')`, which under Next.js 16 is the long-cache profile (renews tag lifetime under the default profile), not immediate invalidation. Backend revalidation calls succeeded at HTTP level but did not invalidate the SSR cache; users saw stale `UserInfoDTO` for up to 5 minutes after state transitions. Fixed by switching to `revalidateTag(tag, { expire: 0 })` — `CacheLifeConfig` with zero expiry forces immediate invalidation.

**Reasoning.** The round-3 handoff named this as a project-wide bug. This pass fixed the `/api/revalidate` site. Other call sites (`oglasino-web/app/actions/cacheActions.ts` and possibly others) still need the same fix — tracked as separate audit follow-up.

---

## 2026-05-19 — Pre-production schema fold for V1 migrations

All schema additions and renames during the pre-launch period edit `V1__init_schema.sql` in place, rather than creating new `V2__*.sql` migration files. This applies to all tables, columns, indexes, and constraints touched by the User Deletion feature. The convention will end when production deploys; from that point onward, new migrations are append-only.

**Reasoning.** The repo has no production deploy yet; rebuilding the schema from a single V1 file on every dev/stage reset is simpler than managing a growing migration chain. Documented in conventions.md.

---

## 2026-05-18 — Firestore Rules registered as the sixth engineer repo

A new repo `oglasino-firestore-rules` holds the Firestore Security Rules separately from `oglasino-backend`. The Firestore Rules engineer agent is now the sixth engineer agent (seventh agent total counting Mastermind).

Stack: Firestore Security Rules language for rules; TypeScript + Vitest + `@firebase/rules-unit-testing` v4 for tests; Firebase Tools 13 for the local emulator; Node 20+. Deployment via `firebase deploy --only firestore` is forbidden for the agent (same deploy ban as every other engineer agent).

The existing conventions Part 3 rule "No Firestore security rule edits without an explicit instruction in the brief" stays load-bearing; it now disambiguates by naming `oglasino-firestore-rules` as the rules' home. The Backend agent does not edit Firestore rules — that's the Firestore Rules agent's exclusive lane.

Updates to meta-documents: Part 2 (repo layout), Part 3 (agent list, deploy ban list, hard-rules note), Part 4 (cleanliness commands), Part 9 (stack table), Mastermind bootstrap Phase 5 brief-order, DevOps bootstrap Phase 2 check surfaces.

**Reasoning.** Firestore rules deserve their own deploy lifecycle separate from the Spring backend's; the previous arrangement of rules-in-backend forced the rules to ride the backend's deploy cadence. Splitting also enables independent rule-tests via the emulator without spinning up a Spring context.

**Alternatives considered.** Keep rules in `oglasino-backend` and have Backend agent edit them when authorized (rejected — Backend agent is Spring/Java only, the JS/TS test fixtures and Firestore Rules DSL fall outside that stack discipline). Have Backend edit `oglasino-firestore-rules` under a narrow cross-repo exception (rejected — same reason; the JS test code makes this not a true narrow exception).

---

## 2026-05-18 — Part 4a (Simplicity) enforcement tightened

Part 4a has been in conventions since 2026-05-13 but the recurring failure mode is engineers ticking "confirmed" on the Conventions check without engaging with the rule. The fix tightens enforcement, not the rule itself.

**What changed.** Part 4a gains an "Enforcement" section requiring structured evidence in every session summary, in three categories: complexity added (with justification), complexity considered and rejected, complexity simplified or removed. Each category accepts "nothing" but must be explicitly written. The Conventions check entry for Part 4a is replaced with a pointer to the evidence, removing the one-word "confirmed" option. The session summary template carries a new required block in "For Mastermind" capturing the three categories.

Mastermind verdicts a session as REVISE if the structured evidence is absent or if the evidence reveals undeclared complexity additions.

**Reasoning.** Engineers self-attesting to "confirmed" on a complex rule is a known weak enforcement pattern. The Conventions check has succeeded at catching brief-vs-conventions drift on rules that have a specific factual answer (Part 6 namespace, Part 7 wire shape). It has been weakest on Part 4a, which requires judgment. Asking engineers to *name* their decisions — what they added, what they didn't, what they removed — raises the cost of skipping the rule and gives Mastermind something concrete to verdict on.

**Alternatives considered.**

- Add SOLID, DRY, KISS, Separation of Concerns, Modularity, Encapsulation as named principles in conventions (rejected — adds vocabulary, not enforcement; failure mode is engineers ticking boxes, not engineers lacking vocabulary; six new principles to tick instead of one).
- Make Part 4a absolute (ban new abstractions without two existing callers; ban configuration for single-value cases) (rejected — already considered and rejected in the original Part 4a decision; absolute rules kill good engineering instincts along with bad).
- Add Boy Scout Rule and YAGNI to conventions (rejected — Boy Scout directly contradicts Part 4b which is already in conventions; YAGNI duplicates Part 4a's configuration guideline).
- Add the principles list as a vocabulary section alongside enforcement (rejected — Option B from the chat discussion; deferred until enforcement-only proves insufficient. Lower-cost path tried first).

---

## 2026-05-18 — QA Preparation: content phase rules and patterns established

Sessions 8–19 authored every in-scope portal page topic plus the six in-scope feature/flow topics (17 in total). The simplification sweep across batches 1–5 and the final cleanup brief (this session) settled a stable set of rules and patterns for QA-topic content. Captured here so future authoring — owner-dashboard topics and the deferred `report-flow` — inherits them.

**1. Trust-boundary findings.** Two flow-topic audits produced full chain-of-evidence trust-boundary content too dense for the topic body and routed instead as verification asks for separate features:

- `start-message-flow` payload (session 12). Firestore rules are the sole boundary for `chats/<id>` writes. The rules must enforce: (a) the authenticated `request.auth.uid` is one of the two participants on the chat document being created or updated; (b) the `withUser.id` field on the request matches the other participant — not a forged third user; (c) the `productId` field, when present, references a product whose owner is the other participant — preventing an attacker from forging an "interested in <other-user's-product>" context; (d) the message body fields cannot be written by the recipient on behalf of the sender. Routed to the Firestore-rules-tightening backlog feature.

- `follow-flow` payload (session 13). The Spring controller `POST /secure/follow/<userId>` is the boundary. The controller must enforce: (a) the path-variable `userId` resolves to an existing, non-deleted user — preventing follows against ghost ids; (b) the `userId` is not the authenticated caller's own id — preventing self-follow rows; (c) repeat follows from the same caller-target pair are either idempotent or rejected — preventing duplicate-row growth on the join table. Routed as a one-shot read-only audit in `oglasino-backend`.

**2. Trust-boundary check model.** The depth standard set in sessions 10, 11, 13: trust-boundary checks must trace the chain — client supply → server derivation → identity layer → authorization. Each verification ends with a single labelled conclusion: "clean," "clean conditional on X," or "concern/violation," with named (a)-(c) checks for the verification surface. Engineers writing trust-boundary checks at this depth is the standard for Part 11 going forward.

**3. Trust-boundary content belongs in audits and decisions, not topic body.** Precedent set in session 17; applied at scale in session 18. The topic body is for testers; the chain-of-evidence — request shape, claim flow, server checks — lives in session summaries and `decisions.md`. Topics carry the *consequence* of trust-boundary findings (a Known-issue pitfall, a routed verification task) but not the chain itself.

**4. Flow-topic structural rule.** Established session 11: `optionsControls` is used when a flow has concrete surfaces to enumerate; skipped when purely behavioural. All four authored flow topics (`category-navigation`, `start-message-flow`, `follow-flow`, plus the future `report-flow`) used `optionsControls`. The schema's other four optional arrays (`howToUse`, `whatToExpect`, `pitfalls`, `qaChecklist`) carry the behavioural content.

**5. Flow-topic `route` field rule.** Established this brief: primary-surface-only. The `route` string carries the dominant entry surface; additional surfaces are enumerated in `optionsControls`. No schema change — the field stays `string`. Applied to `category-navigation`, `start-message-flow`, `follow-flow` in session 19.

**6. Stale-defect-bullet rule.** Established sessions 14–15: if the underlying fix has landed, the topic bullet is cut entirely — no "this used to behave differently" footnote. Applied across batches 1–5 and confirmed no-op in session 19's final stale-defect verification pass.

**7. "Known issue" pitfall pattern legitimized.** Bugs that materially affect a QA cycle before they are fixed may appear in the topic as a "Known issue" pitfall cross-referenced to the matching `issues.md` entry by date and title. Bugs unrelated to the topic's surface stay in `issues.md` only and are not surfaced in the topic. Precedent: `privacy-page`, `terms-page`, `global-header-search`, `category-navigation`. Promoted to standing pattern in this brief: `catalog-page` (slug failure modes), `product-page` (Serbian tooltips), `category-navigation` (Kategorije Serbian label), `follow-flow` (isFollowingCurrent seed), `messages-page` (silent text-send failure, chat-list cap). All five share the same prose shape: "Known issue: <symptom>. Tracked in issues.md (<date> entry — <title>); until fixed, <tester guidance>."

**8. Triage outcomes.** The five-topic triage in chat 2 decided six feature/flow topics in scope: `global-header-search`, `portal-config-dialog`, `category-navigation`, `start-message-flow`, `follow-flow`, plus the deferred `report-flow`. Dropped during triage: `filters`, `product-list-and-pagination`, `favorite-a-product-flow`, `cross-baseSite-redirect-flow`. The dropped list is recorded here so the rationale survives — each was judged either covered adequately inside an existing page topic (`filters` and `product-list-and-pagination` are inside `home-page` and `catalog-page` already; `favorite-a-product-flow` is inside `product-page` and `favorites-page`) or not material as a QA topic (`cross-baseSite-redirect-flow` is a single-redirect mechanic, not a flow).

**9. Brief-authoring lessons.**

- Entry-point candidate lists are unverified hints, not facts. Briefs that name candidate entry points must phrase them as "confirm in code which actually exist" rather than as assertions. Surfaced session 12 (the brief named a messages-page kebab entry that did not exist) and session 13 (only 1 of 3 named follow-handoff candidates was real).

- Stop pre-stating `<n>` in briefs whenever prior archived sessions exist. Engineers determine `<n>` from their own `.agent/` folder. Conditional on the conventions Part 5 amendment in this same session (block 7A below) — once Docs/QA leaves an archive-pointer stub in the source repo's `.agent/` folder, the engineer's own count remains correct after Docs/QA has archived-and-deleted prior session files.

**10. Session-summary "Config-file impact" pattern.** Sessions 13–18 used this section consistently — a short index, listed for each of the four config files as either "no change" or "draft below for Docs/QA." Formalised in conventions Part 5 in this session (block 7B below).

**Reasoning.** Rules established during the content phase needed a durable home outside of session-summary archives. Future authoring (owner-dashboard topics; the deferred `report-flow`) inherits this rule set without re-litigating per topic.

**Alternatives considered.** Recording each rule as a separate `decisions.md` entry at the time it was established (rejected — would have produced ten thin entries; the consolidated form is cheaper to read and more durable). Folding all ten into `meta/conventions.md` directly (rejected — these are feature-specific patterns, not project-wide conventions; the decisions log is the right home, conventions Part 5 captures only the engineer-template structural rule).

---

## 2026-05-18 — User Deletion feature shipped (backend complete; frontend pending separate session)

Backend implementation of self-service account deletion with 7-day grace period, scheduled hard-delete cron, Firebase orphan reconciliation, ban-with-reason flow, and 12-month re-registration prevention is complete and patched per PR review. Frontend work proceeds in a separate Mastermind chat using the canonical spec at [`features/user-deletion.md`](features/user-deletion.md) as authoritative.

**Key engineering decisions captured during the feature:**

- **Pre-production schema fold.** `V1__init_schema.sql` was edited in place rather than adding a new Flyway migration. The decision is documented at the top of V1 and was scoped tightly: no preserved environment had V1 applied, so the fold was safe. Post-launch, all schema changes go in new Flyway migrations as normal.
- **Cron expressions live in `application*.yaml`, not the `Configuration` table.** `@Scheduled(cron = ...)` resolves at bean-construction time from the Spring Environment and cannot read from the runtime `Configuration` cache. The three cron knobs (`user.deletion.hard.delete.cron`, `user.deletion.audit.purge.cron`, `user.deletion.firebase.reconciliation.cron`) sit in YAML; the other twelve user-deletion knobs (batch sizes, retentions, the reconciliation enable flag, the cascade threshold) live in `Configuration` and remain DB-tunable. See corrected spec §3.9 and §7.
- **`user_deletion_requests` rows are intentionally ephemeral.** `ON DELETE CASCADE` on the FK to `users(id)` deletes the request row when the user row is deleted (Mastermind Option B). Spec §8.1 step 15 is removed (no row to update); the audit log carries the durable record. The dead `deleteCompletedOlderThan` repository method was removed in Phase 7. See corrected spec §8.1 and §9.3.
- **Translation seeds for large features may use dedicated SQL files.** The user-deletion feature seeded 33 keys per language across 7 namespaces; doing this inline would have meant ~14 multi-line edits across 8 dense (1000+ line) existing seed files. Engineers shipped four dedicated `0004-data-user-deletion-translations-{EN,RS,RU,CNR}.sql` files instead. Codified as an exception to Part 6 Rule 3 (10+ keys across multiple namespaces threshold).
- **`TransactionTemplate.executeWithoutResult` is blessed alongside `@Lazy self` for cache-aware self-calls.** The user-deletion feature used `executeWithoutResult` for the explicit transactional-boundary split in `DefaultUserDeletionService.runHardDelete`; the existing `@Lazy self` precedent from `DefaultBaseCurrencyService` (decisions.md 2026-05-14) handles caching-proxy-aware self-calls. Both patterns are blessed; mixing them in one method is a smell. Codified as new Part 13.
- **Partial-index `WHERE` predicates must be IMMUTABLE.** Postgres rejects partial indexes whose predicates call STABLE or VOLATILE functions (error 42P17). The `user_deletion_locks` partial-index predicate had to be dropped during V1 work; the active-vs-expired check now runs at query time. Codified as new Part 12.
- **`getBooleanConfig` (silent-false on missing) is the right pattern for destructive batch toggles.** The Firebase reconciliation job uses `getBooleanConfig`, not `getRequiredBooleanConfig` — silent-false is the safer fallback for a destructive cron job than throwing at tick time. The Configuration seed sets the key explicitly so the default never applies in normal operation. See corrected spec §9.2.
- **`disapproved` reviews authored by the deleting user are deleted at hard delete.** The original spec covered approved (anonymize), pending (delete), and reviews-about (delete) but was silent on the disapproved-authored case; PR review caught that `userRepository.delete(user)` would FK-fail without it. Patch session 6 added `ReviewRepository.deleteByReviewerIdAndDisapproved`. See corrected spec §3.3 and §5.

**This decision folds in the prior backlog item "Account-disabling & token-revocation enforcement"** from `state.md`. The veto plumbing landed inside user-deletion: backend `FirebaseAuthFilter` reads `authData.disabled()` (Phase 6), `setDisabled(true)` is called on ban (Phase 5), `revokeRefreshTokens` is called on ban AND on Day-0 deletion request (Phase 5 / patch session 6). The backlog entry is removed by this decision.

**PR review** surfaced 4 critical+high backend bugs (cron-restore race, disapproved-authored review FK violation, audit-log trigger-overwrite, audit-log purge before close); all patched in backend session 6 with focused tests. Backend test suite ends at **444 passing, 0 failing**. Frontend Mastermind chat opens with the corrected spec as the authoritative input.

**Reasoning.** Backend is the platform-level deliverable that frontend and mobile build against; flipping the feature to a shipped state on the backend half unblocks the frontend chat and the future mobile adoption. The corrections above were uncovered by engineers during implementation and by the PR review; capturing them in the canonical spec (rather than only in session summaries) is what keeps future readers from re-litigating decisions that are already settled.

**Alternatives considered.** Hold the shipped flag until frontend lands (rejected — the spec is authoritative for frontend's chat input regardless of state.md status; flipping when backend is done is the established pattern, with frontend tracked via the platform-adoption section on the feature spec). Park the spec corrections as `issues.md` entries (rejected — the corrections are spec-text changes, not bugs; the spec is the canonical surface and stale spec misleads the next reader). Codify the schema/Spring rules inside the user-deletion spec only (rejected — both rules apply repo-wide; conventions Parts 12 and 13 is where they're discoverable to future engineers without context on this feature).

---

## 2026-05-17 — Docs/QA is the sole writer of the four config files; session-closure gate added

Three structural changes propagated across `conventions.md`, the four chat bootstraps, and the five CLAUDE.md files. The configuration audit close-out surfaced enough drift (auth-description issue marked `fixed` while conventions still carried the wrong text; the new Docs/QA `.agent/` archival permission layered without removing the older blanket "never edits other repos" rule; the conventions/CLAUDE.md "four agents" count contradicting the Router agent's own self-identification as a fifth) that the fix is structural, not a typo sweep.

**Docs/QA writes; everyone else drafts.** `conventions.md`, `decisions.md`, `state.md`, and `issues.md` are now written by Docs/QA only. Mastermind, bug chat, DevOps chat, and legal drafts chat draft text in their session output and hand the draft to Igor. Igor briefs Docs/QA in a separate session. Docs/QA applies. Igor commits. Engineer agents have read access via the sibling docs repo; they never write.

Docs/QA may make small independent fixes — typos, broken links, stale dates, formatting consistency — without an upstream draft. Anything substantive (a new rule, a contradiction resolution, a status flip, a new entry) requires an upstream drafter. When in doubt, Docs/QA treats the change as substantive and asks Igor.

This **flips** the 2026-05-13 Docs/QA hard rule "Never edits `decisions.md` or `meta/conventions.md`." That rule was load-bearing for the period when Igor was the sole writer of config files. In practice it produced a recurring drift pattern: the canonical example is the 2026-05-14 `issues.md` entry on the Firebase-auth Parts 9 / 11 wording — drafted by Docs/QA in a session summary, marked `fixed` in `issues.md`, but the conventions edit itself never landed because no agent had write authority and the manual apply step got skipped. The new model removes that gap: the draft and the apply are now two explicit steps with a named agent responsible for each.

**Session-closure gate.** No chat closes with pending config-file drafts. If a Mastermind / bug / DevOps / engineer / legal-drafts session produces a draft for any of the four config files and that draft has not been applied by Docs/QA, the originating chat does not mark itself complete. "Drafted but pending" is not a valid closure state. The Mastermind feature spec at `features/<slug>.md` is held to the same gate — Phase 4 does not end until the spec file exists on disk.

**Conventions Part 3 was rewritten** to capture both rules in one place: the agent list (six agents now, with Router added — see the next entry), the cross-repo edit rules including the narrow Docs/QA `.agent/` exception preserved from the 2026-05-15 decision, the new "Config-file writes" section, the new "Session-closure gate" section, and the updated Mastermind and Docs/QA hard rules.

**Conventions Part 5 was rewritten** to add the "Config-file impact" section to the engineer session-summary template — a short index at the end of every summary that lists, for each of the four config files, either "no change" or "draft below for Docs/QA." Makes pending edits scannable without scrolling through "For Mastermind." Part 5 also disambiguated the `<n>` indexing prose ("first session for a slug starts at `<n>=1`, producing a filename ending in `-<slug>-1.md`"), resolving the earlier contradiction between "starts at `1`" and "starts at `-1`."

**Reasoning.** The drift pattern was real and recurring. Three layers of "never edits" / "may now edit `.agent/`" / "drafts but doesn't write" had accumulated in conventions without the old layer being narrowed. The cost of leaving it was a future agent reading the wrong layer first. The cost of fixing it was one structural pass plus the closure gate, which costs nothing per session and prevents a class of bug worth more than the rule's overhead.

**Alternatives considered.** Keep the existing model and just enforce "Igor applies" more carefully (rejected — that's been the model and it's the model that produced the drift). Give Mastermind direct write access to `decisions.md` and `state.md` (rejected — Mastermind doesn't run in the docs repo and would need a separate session anyway; routing through Docs/QA is cheaper and keeps one agent accountable for what hits disk). Make every drafting chat call its own Docs/QA session inline (rejected — would force a context switch mid-chat and bloat the chat that produced the draft; the draft-then-Docs/QA-session-later flow is what the closure gate is designed for).

---

## 2026-05-17 — Router acknowledged as the fifth engineer agent

The `oglasino-router` repo has a real CLAUDE.md and runs as a Claude Code engineer agent, but every other meta-document treated the system as having four engineer agents (Backend, Web, Mobile, Docs/QA). Conventions Part 2's repo layout omitted `oglasino-router/`. Conventions Part 3 listed five agents but only the four-engineer-agent shape. Four of the five CLAUDE.md files said "you are one of four engineer agents" while `oglasino-router/CLAUDE.md` correctly said five. DevOps Phase 2's propagation map was the only meta-document that named the router repo explicitly.

**The Router agent is now first-class** in every meta-document. Conventions Part 2 lists `oglasino-router/` in the repo layout. Conventions Part 3 names six agents (Mastermind + five engineer agents) with the Router agent's responsibility, branch, and hard rules captured alongside the other engineer agents. The five engineer-agent CLAUDE.md files all say "five engineer agents (Backend, Web, Mobile, Router, Docs/QA)." Mastermind bootstrap Phase 5 mentions router as a possible brief slot in the typical brief order (backend → web → router if affected → mobile → docs cleanup).

**Conventions Part 4 gained the Router lint/test commands** (`npm run lint`, `npm test`). Part 9's stack table gained an "Edge" row covering the Cloudflare Worker / TypeScript / Wrangler 4 / Vitest stack. Part 8 gained a bullet calling out the worker as the edge boundary for the platform.

**The Router agent's hard rules** in conventions Part 3 capture the deploy bans (`wrangler deploy`, `wrangler dev` against production resources, live KV writes outside explicit brief authorization) and the four critical-care areas the worker depends on (maintenance matrix, fail-open KV reads, admin-request regex, `redirect: "manual"` forwarding, stage `noindex` header). The latter four are documented in detail in `oglasino-router/CLAUDE.md` and not duplicated in conventions.

**DevOps bootstrap Phase 2** now references conventions Part 2 for the repo inventory rather than re-enumerating repos inline. The 2026-05-15 maintenance-gate split decision retroactively validates this — that work touched four repos (router, backend, web, droplet) and the DevOps chat managed the propagation correctly because Phase 2's map shape worked. The change is to stop duplicating the repo inventory in the bootstrap.

**Reasoning.** The Router agent's absence from conventions was a coverage gap, not a policy decision. The 2026-05-15 maintenance-gate work showed the router repo is fully integrated into DevOps and Mastermind chats; only the conventions file (and the four downstream CLAUDE.md files) lagged. Same drift pattern as the Firebase-auth wording or the Gradle/Maven mismatch — a rule that contradicts the operational reality. Closed at the conventions level so future agents inherit the correct count.

**Alternatives considered.** Document the Router agent in DevOps bootstrap only and leave conventions agnostic (rejected — engineer-agent rules apply to the Router agent the same way they apply to Backend/Web/Mobile/Docs/QA; the bootstrap can't carry the canonical rules). Treat the Router as infra rather than as an engineer agent (rejected — it has a CLAUDE.md, gets briefs, runs Claude Code sessions, produces session summaries; it's an engineer agent in every operational sense).

---

## 2026-05-17 — Mobile catch-up session slug discipline; Expo backlog table added to state.md

Mobile is multiple features behind backend and web (product-validation, image pipeline, backend-calls reduction). The catch-up sessions need a slugging rule and a tracker.

**Slug discipline.** Mobile adoption sessions are slugged **per feature**, not per time window. A session adopting `product-validation` on mobile is `oglasino-expo-product-validation-1`, the next is `-2`, etc. Each adopted feature carries the same slug it has in `features/<slug>.md`. Time-window slugs like `mobile-catchup-2026-05` are not used.

This matches the existing `(repo, slug)` numbering rule in conventions Part 5 — that rule is designed around features, not time windows. A "catch-up" slug would have been a time-window artifact that doesn't compose with the rest of the system. Per-feature slugs keep mobile consistent with how backend and web are numbered, and let each adopted feature's `## Platform adoption` section in `features/<slug>.md` carry its own session history.

If mobile turns out to need a focused multi-feature session that doesn't map cleanly to one feature (a shared utility, a cross-cutting refactor), the Mastermind chat that opens it decides the slug at that point. The default is per-feature. Switching to a different shape requires a Mastermind decision.

**Expo backlog table.** `state.md` gains a new section, **Expo backlog**, between "Backlog" and "Decisions log." Tracks every feature that is `web-stable` or `shipped` on backend/web but not yet adopted on mobile. Docs/QA maintains it: every time a feature reaches `web-stable` or `shipped`, append a row; every time mobile adopts a feature (the slug reaches `mobile-stable` on the feature spec), remove the row or flip its status. Per the new Docs/QA write authority — small mechanical maintenance like this doesn't need an upstream draft.

Table columns: Feature, Web/Backend status, Mobile status, Adopted in session, Notes. "Mobile status" is one of `not-started`, `in-progress`, `adopted`. The table is seeded retroactively with three rows for product-validation, image pipeline, and backend-calls reduction.

**Reasoning.** Two problems were solved together. The first was the slug question — when the mobile catch-up Mastermind chat opens, the engineer agent needs to know how to name the session, and the answer is "the feature's existing slug, with `<n>` starting at 1 in the Expo repo." The second was visibility — the risk watch entry "Mobile is multiple features behind" was the only place that captured the backlog, and risk-watch prose isn't structured enough to track adoption status per feature. A table gives Igor an at-a-glance view of mobile's debt without reading the full session log.

**Alternatives considered.** One slug per catch-up window — `mobile-catchup-2026-05` (rejected — composes badly with `(repo, slug)` numbering and obscures which feature each session adopted). Punt the slug question to the Mastermind chat that opens the catch-up (rejected — predictable enough to pre-decide; the cost of not deciding is a one-question pause at the start of every mobile chat). Keep the backlog as risk-watch prose only (rejected — gets buried, and adoption status doesn't render cleanly in prose; the table is the natural shape for "feature × adoption status").

---

## 2026-05-15 — Docs/QA may write to engineer repos' `.agent/` folders for session archival

The conventions Part 3 "No cross-repo edits" rule is amended to permit Docs/QA to write within other repos' `.agent/` folders for the specific purpose of session archival. Docs/QA may copy named session files into `oglasino-docs/sessions/` and delete the source files from engineer repos after verifying the archive copy is identical.

**Reasoning:** the archival flow needs source cleanup. Without it, named session files accumulate in both locations and there's no clear "the archive is the canonical copy" boundary. Forcing Igor to delete sources manually defeats the point of having Docs/QA archive at all. The risk of granting write access is narrow because the permission is scoped to one folder name (`.agent/`) and one operation type (archival cleanup).

**Alternatives considered:** keep the rule as-is and let Igor delete sources manually (rejected — defeats the archival workflow). Grant Docs/QA broader write access to engineer repos (rejected — opens the floodgates to scope creep, no concrete need). Move archival to each engineer's responsibility (rejected — engineers can't see other repos' archive, and the centralized archive is the point of having Docs/QA at all).

---

## 2026-05-17 — Redis cache TTLs: five removed, two kept, helper signature reformed; in-memory translations layer brought under the precedent

Two read-only investigations (`sessions/2026-05-17-oglasino-backend-base-site-ttl-investigation-1.md` and `sessions/2026-05-17-oglasino-backend-cache-ttl-investigation-1.md`) inventoried every cache configured in `RedisConfig.java` against every runtime mutation path to the underlying entities. Four code sessions implemented the verdicts.

**TTL removed (five caches):** `redisBaseSites`, `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. None has a runtime write path to its backing entity. `BaseSite` writes are boot-only via `CatalogManager.initBaseSiteCatalog @Order(2)`, before warmup. `Language` and `Currency` are SQL-seeded with no `@Modifying` or `save`/`delete` callers anywhere in `src/main`. `CacheWarmupService` populates all five at boot (`@Order(10)`), before the `AvailabilityChangeEvent` flips readiness to `ACCEPTING_TRAFFIC` per the 2026-05-14 connection-pool decision. Admin-driven manual mutations route through `CacheAdminController`'s evict/refresh endpoints. The TTL served no correctness role; it served only as the trigger for the daily thundering-herd that hit production on 2026-05-16 (and the 24-minute variant for `redisBaseSiteOverviews` whose value was a unit-mix typo).

**TTL kept (two caches, 30 min each):** `redisUserAuth` and `redisUserInfo`. Real runtime writes exist — `DefaultUserService.saveUser`, called from `createUserSynchronized`, `updateCurrentUserData`, `assignUserRegionAndCity`, `disableUser`, and `enableUser` — explicitly paired with `@CacheEvict` for both cache regions. Eviction is the correctness mechanism. The 30-minute TTL is the documented backstop against a future code path bypassing `saveUser`; the user-deletion feature in the backlog is the named risk and will need its own eviction per the 2026-05-14 constraint. `redisUserAuth` already carried a documenting comment at `RedisConfig.java:41-46`; `redisUserInfo` got an equivalent four-line comment in this work pointing at `DefaultUserService.saveUser` / `evictUserInfoCache` as the freshness mechanisms.

**Precedent for future caches.** A cache without runtime writes should not carry a TTL; reliance on warmup-gated readiness plus admin-driven eviction via `CacheAdminController` is the correct shape. A cache with runtime writes must pair every write with `@CacheEvict` (or equivalent re-indexing, in the translations pattern) and may carry a documented TTL backstop. When adding a new cache, the documenting comment at the cache-config site states which freshness mechanism applies — either "no TTL: no runtime write path; populated by CacheWarmupService at boot, operator-driven mutations go through CacheAdminController" or "TTL is a backstop only; freshness owned by `@CacheEvict` on `<method>`."

**Helper signature.** `RedisConfig.java`'s `addTypeConfig` and `addListTypeConfig` now take `Duration ttl` where `null` means "no expiry." Replaces the prior `long ttl` (implicit minutes) form that produced the `redisBaseSiteOverviews` 24-vs-1440 transcription error. Local change; the helpers have no callers outside `RedisConfig.java`.

**Translations precedent extended to the in-memory layer.** The 2026-05-14 "Redis caching & filter control-flow fixes" decision (point 4) framed translations as no-TTL Redis with explicit re-indexing on every write. That framing was correct but incomplete: `DefaultTranslationService` also maintains an in-memory `backendTranslations` map for the `BACKEND_TRANSLATIONS` namespace, populated at boot via `@PostConstruct` and consumed by `getBackendTranslation` for backend-emitted text (push notification bodies, email content). `updateTranslation` did not refresh this map, so admin edits to `BACKEND_TRANSLATIONS` keys landed in DB and Redis but stayed stale in memory until restart. `updateTranslation` now conditionally calls `indexBackendTranslations()` when the mutated namespace is `BACKEND_TRANSLATIONS`. The translations-precedent contract now covers both layers: every translation mutation invalidates every cached representation in the same transaction boundary.

**Three orthogonal bugs fixed in the same chat.** Each surfaced as an adjacent observation during the TTL investigation and was fixed as a discrete bug:

- `DefaultUserFacade.updateCurrentUserData` evicts `redisUserInfo` after `generateUserTranslations` — closes the ordering-dependent shortBio freshness coupling the deferred-follow-up Javadoc had warned about; Javadoc deleted in the same session.
- `redisBaseCurrency` cache value is now `CurrencyDTO` instead of the `Currency` entity, mirroring every other configured cache. Closes the latent lazy-init footgun if `Currency` ever grows a `@ManyToOne`. The canonical pattern for record-DTO conversion in this codebase is direct constructor (`new CurrencyDTO(...)`), not ModelMapper — two existing call sites use this pattern; this work adds a third.

**What this does not fix.** The structural concurrent-cache-miss mechanism — multiple requests racing on a cold cache, each opening a JPA transaction via `@Transactional` on `getAllBaseSites`, exhausting the HikariCP pool — is unchanged. Removing the TTL closes the daily-expiry trigger (and the 24-min `redisBaseSiteOverviews` variant) but does not make the cold-cache path itself safe under concurrent misses on the next deploy or restart. Per-key single-flight deduplication and removing `@Transactional` from `getAllBaseSites` remain in `features/connection-pool-hardening.md`.

**Reasoning.** Inventoried mutation paths matched the verdicts cleanly: five caches with zero runtime writes had nothing for eviction-on-write to do, so TTL was the only freshness mechanism and was purely backstop. With warmup-gated readiness already covering post-deploy cold caches and admin-evict covering operator mutations, TTL had no role left except triggering pool exhaustion at expiry. The two user caches with real writes and paired eviction matched the translations precedent in spirit (write-then-invalidate); their TTL backstops a known future risk (user-deletion, account-disabling enforcement) and is documented at the code site.

**Alternatives considered.**

- `Optional<Duration>` for the helper signature instead of nullable `Duration` — rejected because `Optional` as a method parameter is the documented Java anti-pattern, the helper is `private`, and Spring's own `RedisCacheConfiguration.entryTtl` API treats absence as "don't call."
- `long` with a `0`-or-`-1` sentinel for "no expiry" — rejected because it preserves the implicit-unit footgun that produced the original transcription error.
- Per-cache one-line rationale comment versus a single block comment above the five TTL-less call sites — engineer's choice landed on a single block comment for the shared rationale; per-cache splitting available if grep-ability becomes useful later.
- Unconditional `indexBackendTranslations()` on every `updateTranslation` — rejected for `COMMON`, `BUTTONS`, `INPUT`, etc. paying a `BACKEND_TRANSLATIONS` rebuild they never benefit from. Conditional gate is one enum compare.
- ModelMapper for the `Currency → CurrencyDTO` swap — rejected because two existing call sites use direct constructor and adding a third pattern would violate Part 4a (match the surrounding code's style). ModelMapper's default record matching in 3.2.6 works more by accident than by explicit bean configuration; direct construction is type-safe and one line shorter.

**Accepted-and-known items** logged in `state.md` Risk Watch rather than `issues.md` per Igor's disposition on this chat:

- `DefaultTranslationService.backendTranslations` field is non-`volatile`; concurrency window now exercised more frequently than before. Pre-existing, low severity, one-keyword fix.
- `PriceQueryGenerator.getPriceRangeQuery2` has a dead `var baseCurrency` assignment. Pre-existing, very low severity, fix in passing.
- `CatalogDTO.categories` and `CatalogDTO.orderTypes` are perpetually null on the BaseSite cache path. Investigate during the `connection-pool-hardening` feature work.

---

## 2026-05-15 — Maintenance gate split into two KV flags; deploy flows full-lockdown with manual three-step restore

The Cloudflare router worker's single-flag maintenance gate is now a two-flag gate. The propagation has landed across all four affected repos (worker, backend, web, droplet) and is documented here for the record.

**Worker level.** `oglasino-router` (prod and stage) reads two string-typed KV keys from the `CONFIG` namespace: `maintenance.active` and `admin.bypass.disabled`. Only the literal `"true"` is truthy; absent / non-`"true"` is treated as `false`, so the default with no KV entries is open. The pre-existing third input `use.backend.check` is unchanged. The full behavior matrix lives in [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md).

**Deploy flow (both backend and web).** Backend (`oglasino-backend`) and web (`oglasino-web`) deploy workflows flip **both** `maintenance.active = "true"` and `admin.bypass.disabled = "true"` via `curl` against the Cloudflare REST API before the deploy artifact lands, then wait 60 seconds before deploying. The 60-second wait is 2× the worker's `cacheTtl = 30` so every edge sees both flags `"true"` before the new artifact is reachable. There is **no auto-off** and **no backend liveness probe** in the deploy flow — both flags remain ON after the deploy completes and the workflow's final message points the operator at the restore runbook.

**Restore runbook — three manual steps, in order.** (1) `/opt/oglasino/scripts/admin-bypass-allow.sh` on the droplet flips `admin.bypass.disabled` to `"false"` so admin traffic can reach the panel. (2) Refresh caches in the admin panel — warmed under admin-only load. (3) `/opt/oglasino/scripts/maintenance-off.sh` on the droplet flips `maintenance.active` to `"false"` and resumes end-user traffic against a warmed cache. Ordering is load-bearing: step 1 before step 2 because the operator cannot otherwise reach the panel; step 2 before step 3 because flipping `maintenance.active` off cold dumps end-user traffic at an empty cache.

**Deliberately deferred.** (a) The backend `MaintenancePageController` trust-boundary defect — the maintenance state is anonymous-toggleable from the backend's own admin surface, which violates the server-as-trust-boundary rule (Part 11). Routed to Mastermind as feature work, not addressed in this propagation. (b) Three worker-internal findings: `use.backend.check` sits outside the fail-open `try`/`catch`; the worker's matrix-comment block doesn't mention `use.backend.check`; the admin locale-prefix regex hardcodes `xx-xx` and excludes the three `rsmoto-*` web locales. The first two are logged in [`issues.md`](issues.md) as small fixes; the third is out of scope and belongs with the next locale-routing feature.

**Reasoning.** A single-flag gate cannot distinguish "deploy in progress — keep operators in, keep users out" from "all traffic blocked." Splitting the bypass off into its own flag composes a master/bypass model that delivers both modes with one switch each. The manual three-step restore is the operational standard Igor wants: an explicit human pause between "deploy completed" and "users routed at it" gives the operator a chance to verify caches and surface health before public traffic arrives.

**Alternatives considered.**

- **Per-surface flags (`portal.maintenance.active` + `admin.maintenance.active`).** Rejected. Two independent surface gates require two independent decisions on every deploy; the master+bypass model collapses that to a single primary toggle with an opt-out, which is cheaper to reason about and was already the worker-level design that landed.
- **Auto-off on successful deploy.** Rejected. Igor wants the three-step restore as the operational standard, with the operator's cache-refresh as a deliberate checkpoint between admin-only and public traffic. Auto-off would skip that checkpoint.
- **Replacing the broken backend liveness probe with a Cloudflare-fronted one.** Rejected. The original probe wasn't load-bearing for the deploy decision — its absence does not change correctness. Delete-and-don't-replace was chosen over rewrite-and-keep.

---

## 2026-05-15 — Legal drafts: Privacy Policy and Terms of Use authored

Pre-lawyer drafts of the Privacy Policy and Terms of Use authored for Oglasino, plus a lawyer handoff package. Files: `legal/privacy-policy-draft.md`, `legal/terms-of-use-draft.md`, `legal/lawyer-handoff.md`.

**Key choices captured in the drafts:**

- **Controller identity.** Igor Stojanović, individual sole proprietor, Niš, Serbia. Planned incorporation post-launch handled via Terms Assignment clause + future Terms update.
- **Contact addresses.** privacy@oglasino.com for privacy matters; support@oglasino.com for general support and appeals. Neither mailbox yet operational; both are critical pre-launch action items.
- **Lawful bases.** Contract for most processing (account, profile, listings, messages, reviews, auto-translation). Legitimate interest for moderation, IP rate-limit logging, audit retention. Consent for preference cookies and the five communication-preference toggles.
- **Retention.** Active accounts: while account is active. IPs: 30 minutes. Application logs: 90 days. Deletion audit (hashed): 30 days. Banned-user audit (hashed email): 12 months.
- **No data sale, no AI training on user content.** Mirrored in both documents as explicit user-facing commitments.
- **International transfers.** OpenAI and reCAPTCHA (both US) covered under DPF + SCCs. All other processors EU-hosted (Firebase EU region, R2 to be set EU pre-launch, DigitalOcean fra1, Vercel fra1).
- **Children.** 18+ self-declared at registration.
- **Governing law.** Serbian. Jurisdiction: exclusive Niš with EU consumer carve-out. Controlling language: Serbian.
- **Account deletion.** 7-day soft delete with reporting window — profile remains publicly visible with "Scheduled for deletion" badge, phone hidden, listings hidden, messaging blocked both ways, reports remain open. Hard delete after 7 days. Admin postponement supported (transparent model) for legal investigations and internal abuse investigations.
- **Content rules.** Permitted with conditions: hunting weapons, tobacco/alcohol, home-bred and agricultural animals. Prohibited list covers illegal items, prescription medication, counterfeit, stolen goods, adult content, restricted animals, IP-infringing items, hate content, misleading listings, spam, personal contact info in listings, and external links.
- **Termination.** Discretionary, proportionate. Notice for minor violations, immediate for serious. Email-based appeals. Admin-initiated termination has no 7-day grace period. 12-month re-registration prohibition for banned accounts.

**Pre-launch action items the operator has committed to (logged separately):**

1. Set up and monitor privacy@oglasino.com.
2. Set up and monitor support@oglasino.com.
3. Configure Cloudflare R2 bucket jurisdiction to EU.
4. Accept OpenAI's data-processing addendum.
5. Add "Manage cookie preferences" footer link.

**Adjacent observations surfaced during intake (logged for routing to feature work):**

- `Report.reporter` and `Review.reviewer` are non-nullable `@ManyToOne` FKs that will block user hard-deletion per the spec. Routing needed.
- Review image cleanup may not be covered by the current user-deletion spec.
- User-deletion spec data-inventory table specifies profile is hidden at Day 0, but the operator's actual intent (clarified during intake) is profile-stays-visible-with-status-badge so reporting can continue. Spec must be corrected before implementation.
- Reviews left by deleted users are kept (anonymized); the review-creation UI should disclose this to users at posting time.
- Existing `oglasino-platform/privacy.md` and `oglasino-platform/terms.md` are factually inaccurate against the platform (Facebook login, SMS, Supabase, active Google Analytics — none of which exist). New drafts supersede them entirely; old files to be archived or deleted.
- User-deletion spec config values (`retention-normal-deletion-days`, `retention-banned-months`) were updated by operator to 30/12 during intake; verify the spec doc reflects these so implementation matches the Privacy Policy.

**Reasoning.** The drafts were authored against the platform's audited reality (entities, deletion spec, feature behavior) rather than the prior `oglasino-platform/` drafts, which were placeholder material not aligned with the platform as built. Plain-English style was chosen on operator preference; translation into Serbian, English (final), and Russian is a separate workstream.

**Alternatives considered:** Drafting against the existing `oglasino-platform/privacy.md` and `terms.md` as a base (rejected — they were factually inaccurate, structurally insufficient for GDPR, and editing them would have produced compromised documents rather than clean ones). Publishing without a lawyer review (rejected — operator considered this for cost reasons, agent declined per `meta/legal-drafts-bootstrap.md` Section 4, operator accepted and committed to lawyer engagement).

---

## 2026-05-14 — Connection-pool exhaustion incident: warmup-gated readiness + auth-cache TTL

**Context.** Production-adjacent (stage) logs showed HikariCP connection-pool exhaustion under concurrent load — ~20 simultaneous requests draining an 8-connection pool, cascading into server-wide 500s. Investigation traced the mechanism: `BaseSiteFilter` runs on every request and calls a `@Cacheable + @Transactional` method (`DefaultBaseSiteService.getAllBaseSites`); on a cold-cache window, concurrent misses each open a JPA transaction and grab a connection, and Spring's `CacheInterceptor` has no per-key locking to deduplicate them. `/api/public/translations` appeared in the failure logs as a victim of upstream filter-chain contention, not a culprit. The incident surfaced on stage (pool size 8); the same mechanism is latent on prod (pool size 18).

**Decisions.**

1. **Cache warmup now gates readiness.** `CacheWarmupService` publishes `AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC)` only after the warmup pass completes. The app starts in `REFUSING_TRAFFIC`. This closes the post-deploy cold-cache window — traffic is not routed until the global caches are warm. Chosen over moving warmup into an earlier lifecycle phase, because that would race ahead of `CatalogManager` seeding the `Catalog` rows that `getAllBaseSites` dereferences. The warmup pass flips readiness even if an individual cache fails to warm — the gate is "warmup has run," not "warmup is perfect" — so a transient single-cache failure cannot permanently brick readiness.

2. **The docker-compose healthcheck targets readiness, not liveness.** `infra/docker-compose.yml` and `infra/docker-compose-stage.yml` healthchecks now hit `/actuator/health/readiness` instead of `/actuator/health/liveness`. Liveness reports `CORRECT` early in boot, before warmup; without this change the warmup-gating in decision 1 would be operationally inert because nothing consulted the readiness signal. `start_period` was raised from 90s to 120s (≈2× the confirmed ~60s typical boot+warmup) so the stricter check does not cause a boot loop. If a deploy ever boot-loops on this healthcheck, the response is to lengthen `start_period` further, not to revert — the gate is the load-bearing fix.

3. **`redisUserAuth` TTL raised from 1 minute to 30 minutes.** An investigation traced every production code path that mutates an auth-relevant field (`userRole`, `subscriptionType`, `subscriptionActive`, `disabled`) and confirmed every one routes through `DefaultUserService.saveUser`, whose `@CacheEvict` is the real invalidation mechanism. The 1-minute TTL was a pure safety net with nothing relying on it. 30 minutes matches `redisUserInfo`'s TTL. The eviction `@CacheEvict` — not the TTL — remains the correctness mechanism; the TTL is a backstop.

**Constraint this establishes.** Any future code path that mutates an auth-relevant field on a `User` row must route through `saveUser` (or carry its own `@CacheEvict` for `redisUserAuth`). With a 30-minute TTL, an unevicted change is stale for up to 30 minutes instead of 1. In particular, the planned user-deletion feature must evict `redisUserAuth` for the deleted user's `firebaseUid`.

**Scoped out as feature-sized work (see `features/`):** per-key single-flight deduplication of concurrent cache misses; removing `@Transactional` from `getAllBaseSites`; the `redisBaseSites` 24h-TTL cold-cache trigger (the warmup gate closes the _post-deploy_ window, not the daily-TTL-expiry one); `RateLimitFilter` categorisation of public reference-data endpoints; and the four `@Component` filters having no explicit `@Order`. These were deliberately not fixed in the bug-fixer chat because the honest fix is structural.

---

## 2026-05-14 — Redis caching & filter control-flow fixes (bug-fixer chat)

**Context.** A bug-fixer chat working the connection-pool incident surfaced several adjacent, independently-real bugs in the Redis caching layer and the request-filter chain. Each was investigated and fixed in the same chat.

**Decisions.**

1. **`DefaultBaseCurrencyService` self-invocation bypass fixed.** `convertToBaseCurrency()` and `updateBaseCurrency()` called `this.getBaseCurrency()` directly, bypassing the Spring proxy and the `@Cacheable` annotation — so a price-range-filtered product search hit the DB twice for base-currency data that should have been cached. Fixed via `@Lazy` self-injection (`self.getBaseCurrency()`). This is the first `@Lazy` self-reference in the codebase; it sets the precedent for proxy-respecting self-calls.

2. **`DefaultTranslationService.updateTranslation` made explicitly transactional.** The method mutated a `Translation` entity but had no `@Transactional`, persisting only as a side effect of OSIV plus a later call's transaction. It now carries its own `@Transactional` — if OSIV is ever disabled, admin translation edits would otherwise have silently stopped persisting.

3. **`CurrentLanguageFilter` control flow corrected, and `X-Lang` handling made explicit.** The filter's `try` block wrapped `filterChain.doFilter(...)`, silently swallowing _all_ downstream exceptions and turning genuine 500s into empty 200s. `doFilter` was moved outside the `try` — downstream exceptions now propagate on every route. Additionally: a missing or unresolvable `X-Lang` header now returns **400 `LANG_MISSING_OR_INVALID`** on language-dependent public routes, while a confirmed allowlist of language-independent public routes (`/api/public/baseSite/*`, `/api/public/config`, `/api/public/translations`, `/api/public/health/check`, `/api/public/maintenance/*`, `/api/public/verify-recaptcha`, `/api/public/app/version/*`) continues with language unset. The 400 rule applies only to `/api/public/*` — secure and auth routes were not inventoried for language-dependence and pass through unchanged. Missing and malformed `X-Lang` are treated identically.

4. **Translations Redis cache confirmed to have — correctly — no TTL.** An investigation expecting to find and remove a harmful TTL on the translations cache found there was never a TTL to remove. Every DB-write path to `oglasino_translation` (boot SQL seed; admin `updateTranslation`) triggers a full Redis reindex, so the cache cannot go stale. No change made; recorded so the absence of a TTL is understood to be deliberate and correct, not an oversight.

**Routed out, not fixed here.** `ProductErrorCode` contains cross-cutting error codes that do not belong in a product-scoped enum (`LANG_MISSING_OR_INVALID`, added by decision 3, followed an existing bad precedent). Fixing only the new code would leave the enum inconsistently cleaned; the honest fix is a refactor moving all cross-cutting codes to a proper enum, which is beyond a bug-fixer chat's mandate. Logged to `issues.md` as a high-severity entry — the one deliberate, documented exception to this chat's "nothing parks to issues.md" rule, made because the real fix is feature-sized.

---

## 2026-05-14 — QA Preparation: imageKey rename correction (supersedes earlier entry)

The earlier entry today — "QA Preparation: QaImage field renamed url → imageKey" — was wrong on two points. It claimed the rename was "no code rework" and that `features/qa-preparation.md` had been "updated to match." Both were false. The renderer (`page.tsx`) references the field directly, so renaming the type alone fails typecheck; the Docs/QA pilot session caught this when it attempted the rename and `tsc` rejected it. And the spec was never edited — it still showed `url?: string` with the old comment.

The actual change: a thin web brief renames `QaImage.url` to `imageKey` across three files — `topics.ts` (the type), `page.tsx` (the renderer reference plus the local variable `renderableImageUrls` → `renderableImageKeys`), and `features/qa-preparation.md` (the schema definition and the `images` note). No content values change — no topic entry had set the field yet.

This entry supersedes the earlier one. The earlier entry stays in the log (append-only), but its premise and its "spec updated" claim are void — this entry is the accurate record.

**Reasoning:** the field holds an R2 object key, not a URL — `ProductImageCarousel` consumes keys. A field named `url` that wants a key misleads every content author who fills it. The original error was Mastermind's: the decision was drafted assuming the field was unreferenced, without that being verified. Logged plainly so the correction is on the record and the "verify before claiming no-rework" lesson is visible.

**Alternatives considered:** deleting the earlier entry (rejected — decisions.md is append-only; a wrong entry gets superseded, not erased). Leaving the rename for a future regular web brief (rejected — no regular web brief was queued; the field would stay misnamed indefinitely and every image entry would inherit it).

---

## 2026-05-14 — QA Preparation: QaImage field renamed url → imageKey

The `QaImage` type in the QA Preparation schema had a field `url`, documented as "Cloudflare URL — filled in after the image is uploaded." The web rebuild session surfaced that `ProductImageCarousel` — the component the QA page reuses to render images — does not consume URLs. It consumes R2 object keys, which it runs through `publicImageUrl(key, 'hero')` to build the final CDN URL. A full URL placed in that field would produce a broken nested URL at render time.

The field is renamed `url` → `imageKey`. The doc comment is corrected to "R2 object key, not a URL." `features/qa-preparation.md` is updated to match: the `QaImage` type definition and the `images` note in the Schema section.

No code rework — the rebuild session shipped before any topic had an image entry, and the two example topics have an `images` entry with `name`+`description` and no key. The example in `topics.ts` needs the field name corrected; that rides along with the next web touch or the first docs brief that adds an image.

**Reasoning:** a field named `url` that requires a key is a trap for content authors. Caught before the Docs/QA phase, so the cost is one rename plus a spec edit. Caught after, it would be a find-and-replace across every authored topic with an image.

**Alternatives considered:** make `ProductImageCarousel` URL-aware (rejected — changes a shared component the product surface depends on, out of this feature's scope, and only needed because the field was misnamed). Keep `url` and instruct authors to paste a key anyway (rejected — the field name should tell the truth about what goes in it).

---

## 2026-05-14 — QA Preparation: schema and structure decisions

The QA Preparation feature builds one in-app QA reference page at `/[locale]/design` in `oglasino-web`, content-driven, covering every in-scope page, feature, and flow. The existing `/design` page is the structural template — its layout and data model are kept, its example content is discarded and rebuilt. Five decisions settled after the route inventory + structure audit (`oglasino-web` audit, 2026-05-14):

**Content stays TypeScript, not JSON.** The existing `app/[locale]/design/topics.ts` exports a `QaTopic` type and a `qaTopics: QaTopic[]` const, statically imported and bundled. This is kept. The `QaTopic` type serves as the content schema — Docs/QA authors entries against it with typecheck enforcement. Rejected: JSON file + runtime load — more moving parts, loses type enforcement, and the hot-swap benefit is near-zero since Docs/QA runs with a TS toolchain.

**No prod-exclusion mechanism.** The page can be visible in production. Its content only describes behavior already reachable through the portal — there is nothing sensitive to gate. The audit's "no guard exists" finding is acknowledged and accepted, not treated as a gap.

**`type` field added to the schema.** Every topic is one of `'page' | 'feature' | 'flow' | 'other'`. The current schema is a single flat record with no type discriminator; the feature explicitly deals in four kinds of topic. The field makes each entry self-describing and lets the header group or filter by type later.

**Model A — pages and features are both top-level topics.** A page topic and a feature topic are peers in the flat `qaTopics` list, not nested. A page that has main features references them as separate topics; some pages have no features and reference nothing. Rejected: Model B (nesting `features: QaFeature[]` inside a page topic) — requires new schema, new rendering, and new header logic; the audited page was built flat and adopting nesting is real web work for a gain that Model A plus the `type` field already delivers.

**Cross-references between topics.** A page topic links to its feature/flow topics so a QA tester clicks and scrolls to the sibling topic on the same page. The spec defines the mechanism — current direction is a dedicated `relatedTopics` field holding topic `id`s, separate from `relatedLinks` (external URLs) — to be finalized in `features/qa-preparation.md`.

**Branch:** `feature/qa-preparation`, branched off `dev`.

**Reasoning:** the feature is mostly docs work with a thin web slice. Keeping the existing structure and making the smallest schema additions that serve the four-topic-type model keeps the web work minimal and front-loads the effort onto content authoring, which is where it belongs. All five decisions bias toward the audited reality over a rebuild.

**Alternatives considered:** captured per-decision above.

---

## 2026-05-14 — Conventions Part 4 corrected: backend is Maven, not Gradle

`meta/conventions.md` Part 4 (Cleanliness) listed the backend formatter and test commands as `./gradlew spotlessCheck` and `./gradlew test`. The backend repo is Maven. Engineers have been running `mvn spotless:check` and `mvn test` all along — the convention was wrong and the engineers' actual practice was right. Part 4's backend line is corrected to the Maven commands.

This surfaced during the product-filtering bug sweep: two backend sessions reported running `mvn` commands, and the engineer flagged the convention mismatch explicitly in its session summary.

**Consequence for queued work:** two backend briefs drafted but not yet run — the `oldName` trust-boundary fix and the `translationKey`-on-the-wire change — reference `./gradlew` in their hard rules and definition-of-done sections. Both briefs must have their commands corrected to Maven before they are handed to the backend engineer. The briefs are drafts in chat history, not committed artifacts, so this is a correction-at-hand-off, not a committed-file change.

**Open item:** the corrected line uses unscoped `mvn spotless:check` and `mvn test`. The original Gradle line implied "for touched modules" scoping. If the backend is multi-module and scoped runs are wanted, the exact `mvn -pl <module>` invocation needs Igor's confirmation — this entry does not guess at Maven module-scoping syntax. Unscoped commands are correct (the filtering-sweep backend sessions ran them and passed), just potentially broader than necessary.

**Reasoning:** a convention that contradicts actual practice is worse than no convention — it either gets ignored (eroding the conventions file's authority) or followed by a new agent and breaks. Same category as prior "convention didn't match reality" corrections. Cheap to fix, and it unblocks the two queued briefs from inheriting a wrong command.

**Alternatives considered:** guessing the Maven module-scoping syntax to preserve the "touched modules" intent (rejected — wrong syntax in a conventions file is the exact problem being fixed; better to ship the correct unscoped command and flag scoping as a follow-up).

---

## 2026-05-14 — Image convention and session-file naming added to conventions

Two additions to `meta/conventions.md`.

Part 1 gains an `### Images` subsection. Docs and any agent referencing images write the image reference as if the file exists, name it kebab-case and descriptive (`product-creation-step-2.png`, not `screenshot1.png`), and add an HTML comment above the reference describing what the image should show — scaled to image complexity. Images live in an `assets/` folder next to the referencing doc. Igor supplies the actual files after the doc is written.

Part 5's session-file rule changes. Previously every engineer wrote `.agent/last-session.md`, overwritten each session — so all session history except the latest was lost. Now the engineer writes the summary to `.agent/yyyy-mm-dd-<repo>-<slug>-<n>.md` (a permanent, uniquely-named record) and _also_ to `.agent/last-session.md` (an exact copy, kept so Igor and existing briefs have a predictable path to the latest). `<n>` is sequential per `(repo, slug)` pair, not per day; the engineer counts existing `*-<slug>-*.md` files in its own `.agent/` folder and adds one. Per-repo counting is necessary because an engineer cannot see another repo's `.agent/` folder. The rule applies going forward; pre-rule `last-session.md` files are not backfilled because overwritten content cannot be recovered.

**Reasoning:** the image convention lets docs be written before screenshots exist, without placeholder cruft, and keeps the guidance for each missing file attached to the spot it's needed. The session-naming change fixes real data loss — every engineer session before this rule destroyed the previous session's summary. Numbered permanent files give the feature a full session history; the `last-session.md` copy preserves backward compatibility with every brief and habit that already points at that path.

**Alternatives considered:** for images — visible italic captions instead of HTML comments (rejected — the guidance is a note to the file-supplier, not reader-facing content; alt text already serves the reader). For session naming — per-day numbering (rejected — a slug's sessions don't map cleanly to days; per-slug sequential is the meaningful order). Dropping `last-session.md` entirely once named files exist (rejected — every existing brief and Igor's own workflow point at that path; keeping it as a duplicate costs nothing). Backfilling old session files (rejected — impossible; overwritten content is gone).

---

## 2026-05-13 — Simplicity guideline and adjacent-observations rule added to conventions

Two related additions to `meta/conventions.md`:

**Part 4a — Simplicity.** Guidelines (not bans) on when complexity earns its place. Engineers exercise judgment but explain non-obvious choices in the session summary's "For Mastermind" section. Configuration, abstractions, defensive code, and stylistic deviations are all available when they serve a concrete purpose — and visible when they do.

**Part 4b — Adjacent observations.** Engineers flag bugs and issues they notice outside their brief's scope. Flags include file path, severity guess, and an explicit "out of scope" acknowledgement. Mastermind triages flags into `issues.md`, follow-up briefs, or wontfix.

Session summary template gains one row in "Conventions check" covering both parts.

**Reasoning:** prior session output suggested engineers default to over-engineering (extra abstractions, configuration for non-varying values, defensive checks where the contract is tight). Parallel concern: engineers don't always flag adjacent issues, so bugs that should have been visible get rediscovered later. The two rules pull in opposite directions on attention — simplify what you write, broaden what you notice — which is the right balance for code quality.

**Alternatives considered:** a strict ban list ("no new abstractions without two existing call sites," etc.) — rejected because it kills good engineering instincts along with the bad. A "be simple" exhortation without checkable substance — rejected because vague rules don't change behaviour.

---

## 2026-05-13 — Translation namespaces reconciled with code; VALIDATION frozen

Conventions Part 6 Rule 1 listed 5 namespaces. The `TranslationNamespace` enum in `oglasino-backend` has 22. Conventions were stale. Reality is correct.

Conventions Part 6 Rule 1 is rewritten to list all 22 namespaces in the same grouping the enum uses (CORE, UI, LAYOUT, GLOBAL FEATURES, PAGES, META, BACKEND), with a one-line purpose for each. `INPUTS` (singular `INPUT` in code) is dropped from the convention — the namespace does not exist on disk.

**`VALIDATION` namespace is frozen.** No new keys are added to `VALIDATION`. Anything that would have gone there goes to `ERRORS` instead. Existing `VALIDATION` keys remain in place until a post-launch migration moves them all to `ERRORS`. Rationale: `ERRORS` and `VALIDATION` overlap in practice — both hold user-facing "this is wrong, fix it" strings. One namespace for everything red-on-screen is simpler than two.

Conventions Part 6 Rule 4 (the `product.<field>.<code_lowercase>` pattern) places keys in `ERRORS`. The queued `translationKey` backend brief produces keys for `ERRORS`.

**Reasoning:** the split between `VALIDATION` and `ERRORS` is conceptually defensible but breaks down in the product creation flow, where the same backend code can surface in either context. One bucket simplifies both authoring and consumption. Freezing now (rather than migrating now) avoids a multi-locale data migration during a feature push.

**Alternatives considered:** migrate all `VALIDATION` keys to `ERRORS` immediately (rejected — touches every locale in seed SQL and every consumer in web; scope explosion mid-feature). Keep both namespaces active and document the split (rejected — the split doesn't hold in practice, as the product validation case demonstrates). Drop `VALIDATION` from the enum entirely (rejected — existing keys still resolve through it; removing the enum value breaks live translations).

---

## 2026-05-13 — Session summary template gains "Obsoleted" and "Conventions check" sections

Two additions to the engineer session summary template in `meta/conventions.md` Part 5, between "Cleanup performed" and "Known gaps / TODOs":

**Obsoleted by this session.** What this session made dead but did not delete (stale tests, unreferenced code, contradictory docs). "Nothing" is a valid answer but must be written. Distinct from "Cleanup performed" — cleanup is what was deleted, obsoleted is what should have been deleted but was not, with reasons.

**Conventions check.** Explicit confirmation that the engineer checked Part 4 (cleanliness) and Part 6 (translations) against the work performed, plus any other part of conventions that applied. "N/A this session" is valid where a part does not apply (e.g., Part 6 on a session that touches no translations).

Conventions Part 4 gains a line referring to the new template section, tying the dead-code rule to its diagnostic question.

**Reasoning:** Part 4's existing rule ("if a refactor obsoletes old code, the old code is deleted in the same session") relies on the engineer noticing what's obsolete. The "Obsoleted" section forces the noticing. The "Conventions check" section catches brief-vs-conventions drift — the calibrated-challenge rule covers the same ground but operates at chat time; the template covers the same ground at session-end time. Two checkpoints, one issue.

**Alternatives considered:** trust the calibrated-challenge rule alone (rejected — already failed once in observable conditions). Add a generic "I followed conventions" confirmation (rejected — too vague to fail honestly; named parts force engagement). Move the dead-code rule out of Part 4 into Part 5 entirely (rejected — Part 4 is the rules, Part 5 is the deliverable; they reinforce each other).

---

## 2026-05-13 — "Obsoleted by this session" added to engineer session template

Engineers explicitly answer "what does this session obsolete?" at the end of every session. The answer goes in a new section of the session summary template, between "Cleanup performed" and "Known gaps / TODOs."

The two sections are distinct: "Cleanup performed" is what was deleted in this session; "Obsoleted by this session" is what is now dead but was not deleted (with reasons), or "nothing." A refactor session that leaves a stale test file is the typical case — the test isn't a cleanup target of the refactor itself, but it's now dead, and the next session shouldn't have to rediscover it.

Conventions Part 4 gains one line tying the section to the existing dead-code rule. Conventions Part 5 gains the template section.

**Reasoning:** the existing Part 4 rule ("if a refactor obsoletes old code, the old code is deleted in the same session") relies on the engineer noticing what's obsolete. Adding the diagnostic question to the template forces the noticing.

**Alternatives considered:** add the question to brief-authoring habits only, with no convention change (rejected — habits drift across many briefs; the template is the durable place). Combine obsoleted-tracking into "Cleanup performed" (rejected — mixing what-was-done with what-should-be-done loses the signal).

---

## 2026-05-13 — Engineers required to challenge briefs, with calibrated threshold

All four engineer `CLAUDE.md` files rewritten to include a "Challenging the brief" section. Engineers must read code before writing code, compare brief to actual code, and surface real mismatches in a "Brief vs reality" section before implementing.

The challenge threshold is deliberately calibrated to filter noise. Engineers challenge real bugs only: trust boundary violations, missing or wrong code references, hidden dependencies, regression risks, wire-shape mismatches, off-counts. Engineers do not challenge naming, ordering, formatting, plural vs singular labels, casing, comment phrasing, or "this could be more elegant" thoughts.

Trust boundary check is now an explicit engineer responsibility. If a brief tells an engineer to violate a trust boundary, the engineer flags it CRITICAL and refuses to implement.

**Reasoning:** the `oldName` bug existed in the code, was listed in the backend engineer's audit, but was never challenged. Engineers read code and Mastermind does not — engineers are the last line of defense on brief-vs-reality discrepancies. The first pass at this rule was too broad and surfaced cosmetic issues (singular vs plural namespace labels) at the same priority as real bugs. The calibrated version explicitly excludes cosmetic categories.

**Alternatives considered:** leaving the responsibility entirely with Mastermind (rejected — Mastermind cannot read code, cannot catch this class of bug alone). Making it advisory rather than mandatory (rejected — advisory rules degrade under deadline pressure). Letting engineers challenge anything (rejected — produced noise on the validation feature when an engineer surfaced a singular/plural labeling difference as if it were a bug).

---

## 2026-05-13 — Feature lifecycle and trust boundary principle codified

Two new sections added to `meta/conventions.md`: Part 10 (Feature lifecycle) and Part 11 (Trust boundaries).

Part 10 codifies:

- One Mastermind chat per feature. When the feature ships, the chat closes.
- Five phases per chat: Intake, Audit, Seam analysis, Canonical spec, Engineering briefs.
- Engineers audit code as ground truth, not pre-existing reports or drafts.
- Pre-existing material is input only, never authoritative.

Part 11 codifies the server-as-trust-boundary principle, with the `oldName` bug as the originating example. A value used in a moderation, authorization, or state-transition decision must be derived from auth, read from the server's database, or otherwise unforgeable by the client. Client-supplied "before" values for change detection are forbidden.

**Reasoning:** Mastermind chats had grown polluted across multiple features. Reusing chats caused context drift, stale assumptions, and process erosion. The `oldName` bug exposed a trust boundary violation that no current convention prohibited. Both gaps closed at the convention level so future agents inherit the rules.

**Alternatives considered:** keeping the Feature Audit Phase as Mastermind-only behavior (rejected — engineers also need to know audits are required and what an audit brief looks like). Adding trust boundaries as a one-line note in the security section (rejected — highest-cost class of bug seen so far, deserves its own Part).

---

## 2026-05-13 — Mastermind bootstrap rewritten for per-feature chats

`meta/mastermind-bootstrap.md` fully replaced. New version defines the per-feature chat lifecycle, the five phases, the strictness rules (corrected from over-strict failure mode), the trust boundary rule, and explicit scope limits on what a chat does not do.

Removed from the old bootstrap: long-running Mastermind assumption, cross-feature institutional memory expectation.

Added: phase discipline, "current code is ground truth" rule, chat closes when feature ships, scope-creep prevention.

**Reasoning:** the previous bootstrap was built for one long Mastermind chat covering many features. Context polluted, decisions drifted, the same agent reasoned over stale assumptions. The new bootstrap forces a clean planning surface per feature.

**Alternatives considered:** patching the existing bootstrap to add the lifecycle phases (rejected — the long-running-chat assumption was load-bearing in the old structure; a patch would leave contradictions in the file). Two bootstraps, one for "complex features" and one for "simple bug fixes" (rejected — adds a classification decision before every chat; not worth the complexity).

## 2026-05-13 — Mastermind operating rules formalized

Four operating upgrades applied to Mastermind: strictness, writing style, self-contained instructions, lead-with-a-recommendation. Captured in meta/mastermind-bootstrap.md. New Mastermind chats bootstrap with this file alongside conventions.md, state.md, and decisions.md.

Alternatives considered: keeping the rules implicit in conventions.md (rejected — operating-style rules are about how an agent communicates, not about project content; mixing them clutters both files).

## 2026-05-13 — Feature spec is authoritative over sibling-repo docs

When a doc in a sibling code repo (`oglasino-backend/docs/`, `oglasino-web/docs/`, `oglasino-expo/docs/`) overlaps with a feature spec in `oglasino-docs/features/`, the feature spec wins. Engineers reading the sibling-repo doc must defer to the feature spec on any disagreement.

Added to `meta/conventions.md` Part 1 as a standing rule.

**Reasoning:** the cross-repo consolidation tracked in `future/cross-repo-docs-consolidation.md` will eventually migrate the overlapping docs into `features/`. Until then, engineers need a single, obvious tiebreaker. Without this rule, every overlap is a judgment call.

**Alternatives considered:** scope the rule only to the three currently-overlapping files (rejected — future feature specs will hit the same overlap problem). Migrate the overlapping docs now (rejected — already deferred to post-launch per `future/cross-repo-docs-consolidation.md`).

## 2026-05-13 — Agent architecture and conventions

### Stack: Claude Code per repo, Mastermind in desktop app

Igor uses Claude Code (already installed, v2.1.140) in each of `oglasino-backend`, `oglasino-web`, `oglasino-expo`, `oglasino-docs`. Mastermind runs as a long-running Claude Desktop chat. Igor is the message bus.

**Alternatives considered:** custom Node service against Anthropic SDK (rejected — duplicates Claude Code infrastructure, costs extra tokens on top of Max x20); LangGraph (overkill for 4 agents); Cursor/Windsurf (don't give clean Mastermind/Engineer separation).

### Engineers do not commit

Engineers stage changes on disk only. Igor reviews working-directory diffs in IntelliJ/VS Code (file-color differentiation in the diff view) and runs `git add` / `git commit` himself.

**Alternatives considered:** engineer commits to feature branch only (rejected — Igor prefers diff color review over commit review for catching issues).

### Mastermind reviews summaries, not raw diffs

Engineers produce structured session summaries in `.agent/last-session.md`. Mastermind reads only summaries. If something looks off, Mastermind asks for the relevant code via Igor.

**Reasoning:** prevents Mastermind's context from filling up with every line of code change. Igor explicitly wanted long-running Mastermind conversations.

### Image pipeline create flow: Model C

Hold images in memory through the wizard. Upload to R2 only at step 4. Progressive status messages during step 4 ("Uploading images..." → "Creating product..." → done).

**Alternatives considered:** Model A (upload at step 1, current behavior — rejected, accepts orphan leak on wizard abandonment); Model A+ with pending/promote pattern (rejected for now, deferred to v2 — too much new infrastructure for 1-month timeline).

### Content moderation v2 parked

The policy-scoring engine proposal (Signal with score+confidence, weighted aggregation, ALLOW/WARN/REVIEW/BLOCK tiers) is correct in theory but premature. Parked in [`future/content-moderation-v2.md`](future/content-moderation-v2.md).

**Reasoning:** no production traffic yet means no real abuse data to calibrate weights. Current binary rule system ships and works. Revisit 2–3 months post-launch.

### Translation namespaces are fixed; no parent/child key collision

Translations live in fixed namespaces (ERRORS, INPUTS, COMMON_SYSTEM, DASHBOARD_PAGES, PAGE_SPECIFIC). Keys cannot have parent-child collision (e.g. `some.key` and `some.key.suffix` cannot coexist) — use `.label` suffix on the parent.

**Reasoning:** the translation library rejects parent-child collision. This is a hard library constraint, not a stylistic choice.

### Pre-validate endpoint added at step 2 of create wizard

New `POST /api/secure/products/pre-validate` runs content moderation on `name` + `description` only. Web wizard calls it before advancing from step 2 to step 3. Returns HTTP 200 in both clean and violation cases; clients check `errors.length`.

**Reasoning:** current create UX punishes users by collecting errors only at final submit. Pre-validate at step 2 catches content issues early. Update flow doesn't need this — it's single-page.

### Repo-internal docs stay; new docs go to `oglasino-docs/`

Existing `<repo>/docs/` folders remain (getting-started, deployment, troubleshooting). No new files added there. All new documentation authored in `oglasino-docs/`.

**Reasoning:** long-term goal is single source of truth in `oglasino-docs/` for eventual Jira/Confluence migration. Moving existing docs is busywork; growing new docs in code repos perpetuates the split.

### Docs/QA's first session is a repo sweep

Before any engineering brief runs, Docs/QA inventories `oglasino-docs/` and proposes a reorganization plan (what to move, what to keep, what new folders to add). Igor approves before further work.

**Reasoning:** the docs repo has pre-existing structure I haven't fully audited. Adding `features/`, `sessions/`, `future/`, `legal/` blindly might collide with existing folders.
