# Audit — Expo release readiness, docs-side mapping

**Repo:** oglasino-docs
**Date:** 2026-05-23
**Task:** Produce the feature-shipping map. For every feature this project has worked on, state what shipped where, what's documented, and the docs-side read on mobile's current state.
**Mode:** read-only. No edits, no commits.

---

## Section 1 — Inventory of features

| Feature (slug) | Spec exists? | Backend state | Web state | Mobile state (documented) | Mobile state (your read) | Confidence |
|---|---|---|---|---|---|---|
| product-validation | yes | `shipped` | `web-stable` | "mobile adoption is the only remaining work" (`state.md`, spec `## Platform adoption`) | not started | high |
| product-filtering-and-search | yes | `shipped` | `shipped` | `not-started` (Expo backlog table, `state.md`) | not started | high |
| user-deletion | yes | `shipped` | `shipped` | §19.9 "Mobile adoption" says separate chat post-merge; Expo backlog says `not-started` | not started | high |
| messaging | yes | `shipped` | `shipped` | §11.6 says "Mobile catch-up adopts this spec's contracts"; Expo backlog says `not-started` | not started | high |
| consent-mode-v2 | yes | `shipped` | `shipped` | `## Platform adoption` table says `not-started`; "Separate Mastermind chat" | not started | high |
| google-analytics-v1 | yes | `shipped` | `shipped` | `## Platform adoption` table says `not-started`; "Separate Mastermind chat" for mobile SDK story + ATT | not started | high |
| cookies-closing | yes | `shipped` (Items 1–3 + Item 4 language; theme deferred) | `shipped` (same) | `## Platform adoption`: "This feature is web-only. No mobile work queues." | n/a-mobile | high |
| routing-and-language | yes | n/a (web-only reference doc) | `shipped` (reference doc, not feature spec) | not documented | n/a-mobile | high |
| qa-preparation | yes | n/a (web-only) | `in-progress-web` (`state.md`) | not documented | n/a-mobile | high |
| seo-foundation | yes | `planned` | `planned` | §13 says "Mobile is not in scope. Mobile is not a search-indexable surface." | n/a-mobile | high |
| connection-pool-hardening | yes (stub) | `planned` | n/a (backend-only structural work) | not documented | n/a-mobile | high |
| account-disabling-enforcement | yes (stub) | `planned` (but see Notes — folded into user-deletion) | `planned` | spec §cross-repo says "both web and mobile clients need to handle…" | not started | medium |
| image-pipeline-general | no | `planned` (`state.md` backlog: "Backend + web mostly done") | `planned` (backlog: "Backend + web mostly done") | Expo backlog says `not-started` | not started | medium |
| backend-calls-reduction | no | `planned` (`state.md` backlog: "Done on web") | `planned` (backlog: "Done on web") | Expo backlog says `not-started` | not started | medium |
| firestore-rules-tightening | no | `planned` (`state.md` backlog) | n/a | not documented | n/a-mobile | high |
| chat-messaging-cleanup | no | `planned` (`state.md` backlog) | n/a | not documented | unclear | low |
| privacy-policy-terms-drafts | no (in `legal/`) | `drafted` (`state.md` backlog) | n/a (legal docs, not code) | not documented | n/a-mobile | high |
| portal-cold-load-cascade | no | n/a | `shipped` (`decisions.md` 2026-05-21) | not documented | n/a-mobile | high |
| brevo-smtp | no | `shipped` (`decisions.md` 2026-05-22) | n/a | not documented | n/a-mobile | high |
| user-deletion-auth-contract | yes (companion) | `shipped` (companion to user-deletion) | `shipped` | see user-deletion row | see user-deletion row | high |

---

## Section 2 — Notes per feature

### product-validation

State info from `state.md` active feature section ("Status: `web-stable`"), `features/product-validation.md` `## Platform adoption` section, and Expo backlog table row 1. No contradictions between sources. The spec's Platform adoption section is the most detailed mobile intake I've found across all features — it covers frozen contracts, trust-boundary equivalence, wizard vs single-form open question, and explicitly says mobile conforms without renegotiation. Confidence high.

### product-filtering-and-search

State from `state.md` Expo backlog table row 2 and `features/product-filtering-and-search.md` (status: `shipped`). The spec is a reference implementation capturing the audited web reality from 2026-05-14. Expo backlog notes the spec "lacks a `## Platform adoption` section (flagged to Mastermind)." No session archive or decision references mobile work on this feature. Confidence high that mobile is not started.

### user-deletion

State from `state.md` active feature section ("Status: `shipped`"), `features/user-deletion.md` §19.9 "Mobile adoption", and Expo backlog table row 3. The spec says mobile adoption opens its own `oglasino-expo-user-deletion-<n>` chat post-merge. The B19 backlog-triage item (mobile parity) was explicitly deferred to its own Mastermind chat. No session archive references mobile work. Confidence high.

### messaging

State from `state.md` active feature section ("Status: `shipped`"), `features/messaging.md` §11.6, and Expo backlog table row 6. The spec notes: "Mobile catch-up adopts this spec's contracts. Chat ID scheme, field names, rule expectations are all platform-agnostic." A relevant concern: the `users/{userId}` Firestore rules were tightened to owner-only (Brief 1), and the spec explicitly notes "the mobile app may or may not [read cross-user] — out of scope; if mobile reads cross-user, mobile breaks and is fixed in the mobile catch-up chat." This is a potential breaking change for mobile. Confidence high.

### consent-mode-v2

State from `state.md` active feature section ("Status: `shipped`"), `features/consent-mode-v2.md` `## Platform adoption` table, and `decisions.md` 2026-05-21. Platform adoption table explicitly says mobile is `not-started` with note about iOS ATT and different SDK story. The backend mirror (`allowPreferenceCookies`) was removed end-to-end, which may affect mobile if it was consuming that field. Confidence high.

### google-analytics-v1

State from `state.md` active feature section ("Status: `shipped`"), `decisions.md` 2026-05-23, and Expo backlog table row 7. The spec's `## Platform adoption` table says mobile `not-started`. However, the spec's `**Status:**` field still says `in-progress-web` — this is stale and contradicts `state.md` and `decisions.md`, which both confirm shipped. No session archive references mobile analytics work. Expo backlog notes the distinct mobile SDK story (`@react-native-firebase/analytics` or `expo-firebase-analytics`) plus ATT on iOS. Confidence high.

### cookies-closing

State from `state.md` active feature section and `features/cookies-closing.md` `## Platform adoption`. The spec explicitly states: "This feature is web-only. Mobile (`oglasino-expo`) has no equivalent surface today — the Expo app does not have a card-size selector or a multi-scope (portal/owner/admin) split. No mobile work queues from this feature." The `isDashboard → portalScope` migration and `useLanguageStore` collapse are web-internal and don't produce a mobile contract. Theme is deferred. Confidence high that this is n/a for mobile.

### routing-and-language

State from `features/routing-and-language.md` (reference doc, no status field). This is a permanent reference for the web routing infrastructure — not a feature spec in the lifecycle sense. No mobile relevance; routing is entirely web middleware. Confidence high.

### qa-preparation

State from `state.md` active feature section ("Status: `in-progress-web`"). Web-only feature (in-app QA reference page at `/design`). No mobile relevance. Confidence high.

### seo-foundation

State from `features/seo-foundation.md` — recently moved from `future/` to `features/` per git status. The spec is a full canonical spec (not a stub) covering eight cross-repo seam resolutions plus 14 work items across `oglasino-web`, `oglasino-backend`, and `oglasino-router`. §13 explicitly says: "Mobile is not in scope for this feature. Mobile is not a search-indexable surface." However, the spec has no `**Status:**` line, and `state.md` does not list it in either the active features or the backlog section. This is a documentation gap. Confidence high that mobile is n/a.

### connection-pool-hardening

State from `features/connection-pool-hardening.md` (stub, `planned`) and `state.md` backlog table. Backend-only structural work (HikariCP pool, cache patterns). No mobile relevance. Confidence high.

### account-disabling-enforcement

State from `features/account-disabling-enforcement.md` (stub, `planned`). The spec names web and mobile as needing rejection-handling work. However, `decisions.md` 2026-05-18 (User Deletion shipped) explicitly states: "This decision folds in the prior backlog item 'Account-disabling & token-revocation enforcement'" — and `state.md` no longer lists it in the backlog. The stub spec is stale. The core enforcement work (backend `FirebaseAuthFilter` reading `authData.disabled()`, `revokeRefreshTokens` on ban/deletion) shipped as part of user-deletion. What remains unverified from the docs trail is whether mobile's rejection-handling has been addressed. My read: `not started` for mobile, because user-deletion's mobile adoption (which inherits this concern) is itself not started. Confidence medium because the fold is indirect.

### image-pipeline-general

No spec in `features/`. State from `state.md` backlog ("Backend + web mostly done. Expo has to adopt. Some overlap with Model C work above.") and Expo backlog table row 4 ("No feature spec yet"). The `decisions.md` 2026-05-13 entry on "Image pipeline create flow: Model C" covers the decision but not a full spec. Confidence medium — "mostly done" is ambiguous about whether backend/web are truly shipped.

### backend-calls-reduction

No spec in `features/`. State from `state.md` backlog ("Done on web. Expo has to adopt. Likely a small adapter task.") and Expo backlog table row 5 ("No feature spec yet"). No session archive I can identify as being specifically about this feature. Confidence medium — "Done on web" suggests `web-stable` but there's no formal status.

### firestore-rules-tightening

No spec in `features/`. State from `state.md` backlog ("Needs audit before scoping. Requires explicit Mastermind sign-off."). The messaging feature's Firestore rules work (Brief 1, 24 tests) tightened some rules, but this backlog item appears broader. No mobile relevance. Confidence high.

### chat-messaging-cleanup

No spec in `features/`. State from `state.md` backlog ("Needs audit before scoping."). Unclear whether this overlaps with the now-shipped messaging feature or is a separate cleanup pass. Mobile relevance unclear because scope is undefined. Confidence low.

### privacy-policy-terms-drafts

No feature spec; lives in `legal/`. State from `state.md` backlog ("Pre-lawyer drafts in `legal/`. Pending lawyer review and finalization before launch."). Legal docs, not code. No mobile relevance. Confidence high.

### portal-cold-load-cascade

No feature spec. Three fixes shipped on `oglasino-web` per `decisions.md` 2026-05-21. Web-only (FilterManager, FilteredProductList, UseTokenRefresh). No mobile relevance. Confidence high.

### brevo-smtp

No feature spec. Shipped per `decisions.md` 2026-05-22. Backend-only email infrastructure. No mobile relevance. Confidence high.

### user-deletion-auth-contract

Companion document to user-deletion. Status: `finalized`. Mobile state inherits user-deletion's row. No independent mobile consideration. Confidence high.

---

## Section 3 — Features mentioned in the docs but not in `features/`

- **image-pipeline-general** — Backend + web mostly done; Expo adoption queued. In `state.md` backlog and Expo backlog. No spec.
- **backend-calls-reduction** — Done on web; Expo adoption queued. In `state.md` backlog and Expo backlog. No spec.
- **firestore-rules-tightening** — Planned; in `state.md` backlog. No spec.
- **chat-messaging-cleanup** — Planned, needs audit; in `state.md` backlog. No spec.
- **portal-cold-load-cascade** — Three fixes shipped 2026-05-21. Captured in `decisions.md` only; no feature spec. (Arguably doesn't need one — it was a bug-fix cluster, not a feature.)
- **brevo-smtp** — Shipped 2026-05-22. Captured in `decisions.md` only; no feature spec. (Infrastructure work.)
- **content-moderation-v2** — Parked in `future/content-moderation-v2.md`. Referenced in `state.md` "Future work" section. No spec in `features/`.

---

## Section 4 — Features that look mobile-shipped without a corresponding spec section

`none`

No documentation trail — session archives, decisions, or state entries — indicates any feature has been adopted on mobile. Every Expo backlog row shows `not-started`. No `oglasino-expo-*` session archives exist in `sessions/`. The Expo backlog table has never had a row removed or status-flipped to `in-progress` or `adopted`.

---

## Section 5 — Coverage of source material

### Files in `features/`

14 files:

1. `account-disabling-enforcement.md` (stub)
2. `connection-pool-hardening.md` (stub)
3. `consent-mode-v2.md`
4. `cookies-closing.md`
5. `google-analytics-v1.md`
6. `messaging.md`
7. `product-filtering-and-search.md`
8. `product-validation.md`
9. `qa-preparation.md`
10. `routing-and-language.md` (reference doc)
11. `seo-foundation.md`
12. `user-deletion.md`
13. `user-deletion-auth-contract.md` (companion)
14. `user-deletion-test-cases.md` (companion)

### Sessions directory

`sessions/` has content: **266 archive files**. Date range: 2026-05-13 through 2026-05-23. Repos represented: `oglasino-backend`, `oglasino-web`, `oglasino-docs`, `oglasino-router`, `oglasino-firestore-rules`. No `oglasino-expo` session archives exist.

### Decisions log

`decisions.md` contains **27 decision entries** spanning 2026-05-13 through 2026-05-23. All entries read in full (927 lines).

### Issues log

`issues.md` contains entries spanning 2026-05-13 through 2026-05-23. **49 entries** (estimated by counting `##` headers). Mix of `open`, `fixed`, `wontfix`, and `parked` statuses. All entries read in full (931 lines).

### Other material read

- `state.md` — full file (230 lines), including active features, backlog, Expo backlog, session log, risk watch, future work.
- `meta/conventions.md` — full file (560 lines).
- `future/` directory — 3 files present (`content-moderation-v2.md`, `cross-repo-docs-consolidation.md`, `ga4-analytics-v1.md`).
- `.agent/` directory — 10 entries including `brief.md`, `last-session.md`, `handoffs/`, and 7 recent session files.
- Mobile-relevant sections of all 14 feature specs scanned via targeted grep + section reads.

### Access issues

None. All files readable. No broken references encountered during the read pass.

---

## Section 6 — For Mastermind

### Cross-file contradictions

1. **`google-analytics-v1.md` status field says `in-progress-web`; `state.md` and `decisions.md` 2026-05-23 both say `shipped`.** The spec's `**Status:**` line was never updated after the feature shipped. This is a stale-status gap — the spec is the canonical reference per conventions Part 1, so a reader who checks the spec first gets the wrong state.

2. **`account-disabling-enforcement.md` stub says `planned`; `decisions.md` 2026-05-18 says the feature was folded into user-deletion; `state.md` no longer lists it in the backlog.** The stub is stale. Either the stub should be archived/deleted (since the work shipped under user-deletion) or it should be updated with a "folded into user-deletion, see §X" redirect. Currently a reader who finds the stub thinks the work is unstarted.

3. **`seo-foundation.md` exists in `features/` (moved from `future/` per git status) but is not listed anywhere in `state.md` — neither as an active feature, nor in the backlog.** This is a coverage gap. The spec is a full canonical spec (not a stub) with 14 work items across three repos.

4. **`future/ga4-analytics-v1.md` still exists alongside the canonical `features/google-analytics-v1.md`.** The file in `future/` is a stale remnant. Should be deleted to avoid confusion.

### Features where documented-vs-actual gap is likely large

- **image-pipeline-general** and **backend-calls-reduction**: both say "mostly done" / "done on web" in the backlog but have no feature specs. Without specs, there's no frozen contract for mobile to adopt against. The Expo backlog rows for both say "No feature spec yet" — the Expo backlog itself flags this gap. When mobile adoption chats open for these, there's no Phase 4 canonical spec to hand them, which breaks the Phase 5 engineering-brief flow per conventions Part 10.

- **chat-messaging-cleanup**: listed as `planned` in the backlog but may partially overlap with the messaging feature that shipped (messaging Brief 2 did significant cleanup work: `tempProductReason` rename, clearing fix, atomic batches, send-failure toast, "Load more", etc.). The backlog entry's scope vs the shipped messaging feature's scope is unclear from the documentation trail alone.

### Surprises

1. **Zero `oglasino-expo` session archives.** 266 session files in `sessions/`, spanning five repos over 11 days. Not one Expo session. This is consistent with the Expo backlog showing `not-started` across all rows, but it also means the mobile repo has had zero documented agent work since the conventions system was established.

2. **The Expo backlog table is comprehensive and well-maintained.** Every feature reaching `web-stable` or `shipped` has been tracked. The table was seeded on 2026-05-17 per the slug-discipline decision and has been kept current through the GA4 shipping on 2026-05-23. This is a good docs-side signal.

3. **The `seo-foundation.md` spec appearing in `features/` without a corresponding `state.md` entry.** This is the only feature spec I found that has no presence in `state.md`. All others — even stubs — are represented either in the active features section, the backlog table, or the Expo backlog.

4. **`future/seo-foundation.md` was deleted and `features/seo-foundation.md` was created** (per git status showing `D future/seo-foundation.md` and `?? features/seo-foundation.md`), but this move hasn't been reflected in `state.md`. The file is untracked by git, suggesting the move happened in the current working session.

5. **Messaging feature's Firestore rule tightening (`users/{userId}` → owner-only) is an explicit documented potential breaking change for mobile.** The spec calls this out directly: "the mobile app may or may not [read cross-user] — out of scope; if mobile reads cross-user, mobile breaks and is fixed in the mobile catch-up chat." This is the single highest-risk documented concern for mobile adoption.

6. **Consent Mode v2 removed `allowPreferenceCookies` from backend DTOs end-to-end.** If mobile was consuming `AuthUserDTO.allowPreferenceCookies` or `UpdateUserDTO.allowPreferenceCookies`, those fields no longer exist. Not called out in the Expo backlog notes but could be a wire-shape breaking change.

---

## Cleanup performed

`none needed` — read-only audit. No files modified.

## Obsoleted by this session

`nothing` — read-only audit. No prior documents superseded.

## Conventions check

- Part 4 (cleanliness): N/A this session — no code, no files modified.
- Part 4a (simplicity): N/A this session.
- Part 4b (adjacent observations): N/A this session — observations routed into Section 6 above per the audit brief's structure.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 1 (doc style — followed for this document); Part 5 (session summary template — followed for the summary sections below); Part 10 (feature lifecycle — referenced for understanding Phase 2/4/5 status per feature).

---

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change
