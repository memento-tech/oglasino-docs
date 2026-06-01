# Expo Release Readiness

**Status:** `in-progress`
**Chat:** Mastermind — Expo Release Readiness (2026-05-23)
**Owner:** Igor
**Scope:** mobile (`oglasino-expo`) only, with cross-references to existing backend and web work.

This document is the canonical output of a Mastermind chat that audited the entire mobile codebase to determine what stands between today and a regression-testable release. It is a **planning artifact**, not a feature spec in the usual sense — no single engineer implements it. Instead, it carries a triaged inventory of every gap and a queue of future Mastermind chats that will resolve them one at a time.

The chat that produced this document closes when this file is committed. Each item in the "Future chats" section opens its own Mastermind chat per `meta/conventions.md` Part 10.

---

## Prerequisite: structural foundation must ship first

A structural audit of `oglasino-expo` on 2026-05-24 (rating: 5.5/10) found 26 cross-cutting structural issues across seven categories — lifecycle gaps, performance shape, RN convention violations, state-shape problems, web-seam mismatches, and platform-API gaps. These issues do not map to any single feature chat in this document's A–I queue; they cross-cut all of them.

The audit produced a new spec at [`features/expo-structural-foundation.md`](expo-structural-foundation.md) that defines four prerequisite foundation chats (Φ1 auth lifecycle, Φ2 navigation, Φ3 performance, Φ4 service layer + error contract). **All four Φ chats must ship before any chat in this document's queue can open.** The revised full queue is: Φ1 → Φ2 → Φ3 → Φ4 → A → B → C → D → E → H → G → F → I → Ω → Ψ.

See the [`decisions.md`](../decisions.md) entry dated 2026-05-24 ("Expo structural audit closed") for the full reasoning, alternatives considered, and cross-cutting decisions.

---

## 1. Why this document exists

The mobile app has not been touched under the current conventions system. The `sessions/` archive contains 266 entries across `oglasino-backend`, `oglasino-web`, `oglasino-docs`, `oglasino-router`, and `oglasino-firestore-rules` — and zero entries for `oglasino-expo`. Every feature the web and backend have shipped under conventions is unadopted on mobile.

Igor wanted to know, before starting regression testing in 1–2 weeks: what's the gap, what breaks if mobile ships today, and what's the order to close the gap in. This chat answered those questions through ten parallel read-only audits.

This document records the answers.

---

## 2. How the chat ran

Phase 1 (Intake) confirmed:

- Scope is mobile features, not devops or store submission. Local-installable dev build is the runtime target.
- Look back across all docs we have.
- Admin removal is in scope as pre-release cleanup.
- Four locales on web; mobile should match.
- All work in this chat is audits only. Implementation happens in separate chats.

Phase 2 (Audit) ran in two waves:

**Wave 1 — Documentation-side mapping (Docs/QA).** Before any mobile-side audit, Docs/QA produced a feature-shipping map that listed every feature this project has touched, where each one shipped, and what mobile's state appeared to be from the documentation trail alone. This map drove the batching for wave 2.

**Wave 2 — Mobile-side audits (Mobile Engineer, parallel).** Ten audit briefs ran in parallel against `oglasino-expo`:

1. General Expo health
2. Product validation
3. Product filtering and search
4. Image pipeline
5. User deletion + account-disabling enforcement
6. Messaging
7. Consent Mode v2
8. Google Analytics v1
9. Backend calls reduction
10. Admin removal scope

Phase 3 (Seam analysis) consolidated findings into the four buckets in section 4 below.

Phase 4 (this document) is the canonical spec.

Phase 5 (Engineering briefs) is intentionally deferred — each future chat in section 5 will write its own.

---

## 3. Audit artifacts produced

All audit files live in `.agent/` of the respective repo and were archived to `sessions/` by Docs/QA after this chat closes. Future chats reading this spec should pull the relevant audit file as input rather than re-discovering its contents.

| Audit | Repo | File |
|---|---|---|
| Docs-side feature-shipping map | oglasino-docs | `.agent/audit-expo-release-readiness-docs.md` |
| General Expo health | oglasino-expo | `.agent/audit-expo-readiness-general.md` |
| Product validation | oglasino-expo | `.agent/audit-expo-readiness-product-validation.md` |
| Product filtering and search | oglasino-expo | `.agent/audit-expo-readiness-product-filtering.md` |
| Image pipeline | oglasino-expo | `.agent/audit-expo-readiness-image-pipeline.md` |
| User deletion + account disabling | oglasino-expo | `.agent/audit-expo-readiness-user-deletion.md` |
| Messaging | oglasino-expo | `.agent/audit-expo-readiness-messaging.md` |
| Consent Mode v2 | oglasino-expo | `.agent/audit-expo-readiness-consent-mode.md` |
| Google Analytics v1 | oglasino-expo | `.agent/audit-expo-readiness-google-analytics.md` |
| Backend calls reduction | oglasino-expo | `.agent/audit-expo-readiness-backend-calls-reduction.md` |
| Admin removal scope | oglasino-expo | `.agent/audit-expo-readiness-admin-removal.md` |

---

## 4. The findings — triaged into four buckets

Every concrete finding from the eleven audits sorts into one of four buckets. The bucket determines what happens to the finding.

### Bucket 1 — Existing mobile bugs

Things mobile already has wrong, independent of any feature adoption. These do not require a feature chat — they're standalone bugs that ride into whichever feature chat opens the relevant surface. The "Carrier chat" column names which future chat naturally picks each one up.

| # | Finding | Source | Severity | Carrier chat |
|---|---|---|---|---|
| B1 | `logoutFirebase` doesn't call `auth.signOut()`. Firebase Auth session persists after user "logs out". `auth.currentUser` remains set, axios interceptor continues to attach `Authorization: Bearer <token>`, authenticated API calls continue to work. | User deletion audit | High | E |
| B2 | `allowPromoEmails` initialization bug at `user.tsx:70`. Line 69 sets it correctly, line 70 overwrites with `allowPhoneCalling`'s value. Promo emails toggle shows phone-calling preference. | Consent audit | Medium | C |
| B3 | `allowNotifications` toggle is fully dead. State declared, toggle rendered, never initialized from server, never sent on save. | Consent audit | Medium | C |
| B4 | Five COOKIES-namespace translation keys (`required.label`, `required.description`, `required.sub.description`, `config.label`, `config.description`) were deleted from backend SQL. Mobile still references them. Settings screen shows raw key strings or empty labels today. | Consent audit | Medium | C |
| B5 | `<div>` element in RN code at `FiltersDialog.tsx:148`. Defensive crash on unknown filter type — RN has no `<div>`. | Filtering audit | Low | D |
| B6 | `clearAllFilters()` doesn't reset `selectedProductStates` / `selectedModerationStates`. Orphaned filter state after "clear all". | Filtering audit | Low | D |
| B7 | Search submit drops category context. Web preserves the `/catalog/...` path on submit; mobile navigates to root. | Filtering audit | Low | D |
| B8 | `UserProductsList` reads from `usePortalFilterStore` and applies all portal filters to a user-profile product listing. If user has filters set on home, they silently apply on user profile pages. | Filtering audit | Medium | D |
| B9 | Product state and moderation state filter labels render raw enum values (`"ACTIVE"`, `"INACTIVE"`) instead of translation keys, on dashboard and admin surfaces. | Filtering audit | Low | D |
| B10 | `NotificationType.NORMAL = 'NOTMAL'` typo in `firebaseNotifications.ts`. No current impact (messaging push notifications unwired); becomes a bug the moment MESSAGE notifications wire if backend uses correct spelling. | Messaging + General audits | Low | B |
| B11 | `getActiveChat` fallback reads `withUserFirebaseUid` / `withUser` from chat root doc — fields that don't exist there (they're on the sidecar). Breaks for any chat outside the 15-cache. Web fixed this; mobile didn't adopt. | Messaging audit | Medium | B |
| B12 | Five hardcoded Serbian strings in messaging UI (search placeholder, select-chat message, load older, two blocking notices, "Nove poruke" preview). Part 6 violation. | Messaging audit | Low | B |
| B13 | Description link/contact errors written to `nameErrorKey` instead of `descriptionErrorKey` in `productValidator.ts:189,191`. Description link errors display on the name field. | Validation audit | Low | A (will disappear — entire validator deleted by A) |
| B14 | 70-word hardcoded `BANNED_WORDS` array in `productValidator.ts`. Duplicates server-side moderation with stale, hardcoded data. Spec says client checks are structural only. | Validation audit | Low | A (will disappear — deleted by A) |
| B15 | Two hardcoded Serbian strings in image-picker UI (`ImageSourceSheet.tsx`, `ProductReviewImageImport.tsx`). | Image pipeline audit | Low | H |
| B16 | Several ad-hoc `console.error` and `console.warn` calls violating Part 4. Spread across `authService`, `authStore`, `productService`, `userService`, `productList`, `uploadedProductDialog`. | Multiple audits | Low | Distributed (each carrier chat removes the ones in its surface) |
| B17 | Hardcoded `"DODAJ ZA BASE SITE"` development placeholder at `user.tsx:230`. Part 6 violation visible to users. | Consent audit | Low | C |
| B18 | `var` keyword used instead of `const`/`let` in several places (`user.tsx:178`, `useChatStore.ts:454`). Cosmetic. | Multiple audits | Cosmetic | Distributed |

### Bucket 2 — Feature adoptions

Features that shipped on web or backend but mobile hasn't adopted. Each becomes its own future Mastermind chat. Detailed in section 5.

| ID | Future chat | Scope estimate |
|---|---|---|
| A | Mobile validation rebuild | Large |
| B | Mobile messaging adoption | Medium |
| C | Mobile user-settings cleanup + consent-field removal — **merged into G** (2026-05-30); no separate C chat opens | Small-to-medium |
| D | Mobile filtering & search polish | Small |
| E | Mobile user-deletion adoption | Medium-large |
| F | Mobile analytics (GA v1) | Medium |
| G | Mobile consent UI v1 — **absorbs C** (consent-field cleanup); spec authored 2026-05-30 at [`consent-mode-mobile.md`](consent-mode-mobile.md) | Medium (new feature) |
| H | Mobile image-pipeline polish | Small |
| I | Mobile backend-calls optimization | Shipped 2026-05-31 — small after predecessors. Spec at `backend-calls-reduction-mobile.md`. |

### Bucket 3 — Pre-release cleanup

Pre-release housekeeping that isn't feature work and isn't an existing bug. Most fold into other chats; one item — admin removal — runs as its own focused chat.

| # | Item | Source | Disposition |
|---|---|---|---|
| P1 | Remove admin surface from mobile (48 files + 13 coupling edits). | Admin removal audit | Future chat **α** (its own dedicated chat, see section 5) |
| P2 | Delete stale `oglasino-dev-firebase-adminsdk-fbsvc-002e6b2f58.json` at repo root. Key already invalidated per Igor; just file deletion. | General audit | Folds into P1 chat (Igor confirms; engineer does `git rm` as part of admin cleanup, since it's mechanical and in the same neighborhood) |
| P3 | Decision needed: delete `expo-image` (unused) or adopt it for caching. | General + Image pipeline audits | Decision deferred to H. Chat H decides. |
| P4 | Delete dead code identified across audits: `firebaseAnalytics.ts`, `ConsentData.ts`, `GlobalCookie.ts`, `authStorage.ts`, `testConnection()`, `updateUserAuth`, `productUpdateNameValidator.ts`, `AdminSessionGuard.tsx` (unused), `checkIsBlocked`, `loginWithFacebookFirebase`, `setCatalog` / `catalogStorage`. | Distributed | Each future chat deletes the dead code in its own surface per Part 4 |
| P5 | Two specless backlog items (`image-pipeline-general`, `backend-calls-reduction`) need Phase 4 specs before mobile adoption per Part 10. | Docs-side audit | Chats H and I each open with a Phase 4 spec-authoring step before drafting briefs. The spec gets authored from the audit findings + minimal additional intake. Phase 4 inside the same chat — no separate spec-authoring chat needed. |
| P6 | Docs hygiene: `google-analytics-v1.md` status says `in-progress-web` (shipped); `account-disabling-enforcement.md` stub stale (folded into user-deletion); `seo-foundation.md` not in `state.md`; `future/ga4-analytics-v1.md` remnant. | Docs-side audit | Separate Docs/QA chore session (logged in section 6) |

### Bucket 4 — Cross-repo trust-boundary questions

Things mobile can't verify alone. They need a backend audit to answer. Logged here for whoever opens the next backend chat.

| # | Question | Source |
|---|---|---|
| X1 | Does `POST /secure/push/token` validate the `userId` request-body parameter against the authenticated principal, or trust it from the body? | User deletion audit |
| X2 | Does `POST /secure/user/update` read `id` and `firebaseUid` from the request body, or derive the user from `SecurityContextHolder`? | User deletion audit |
| X3 | Does the old `POST /secure/product/addUpdate` endpoint still exist on the backend? If yes, does it still read `oldName` / `oldDescription` / `productState` / `moderationState` from the request body? | Validation audit |
| X4 | Do the image upload-token and view-token endpoints verify chat participation server-side when `scope: 'chat'` is sent with a `chatId`? | Image pipeline + Messaging audits |
| X5 | Does the `/auth/firebase-sync` backend endpoint read `idToken` from the request body, or only from the Authorization header? If the body field is unused, mobile can stop sending it. Answered 2026-05-31 (chat I backend audit): `LoginRequest` has no `idToken` field; token comes only from the `Authorization` header. Mobile dropped the body field (C1). | Backend-calls audit |

These don't block this chat. They go on the agenda for whoever opens the next backend audit, and they may also surface during the implementation of E (user deletion) or A (validation rebuild), at which point the engineer flags and Mastermind routes.

**2026-05-24 structural audit update:** the structural audit re-confirmed X1 (as Q3) and X3 (as Q1) and added a new question Q2 (USER_BANNED 403 response shape — does the backend include a machine-readable code in the error body?). All three answers feed Φ1 — see [`features/expo-structural-foundation.md`](expo-structural-foundation.md) for the parallel backend audit chat that opens alongside Φ1 to answer Q1, Q2, Q3.

---

## 5. Future chats — the queue

Each row opens its own Mastermind chat per `meta/conventions.md` Part 10. The order below is the recommended default; Igor can re-order based on priorities at the time. Chats labeled "Spec exists" use the named spec as their Phase 4 input. Chats labeled "No spec — author in chat" do Phase 4 inside the chat by writing a spec from the audit findings before drafting briefs.

### α — Admin removal from mobile

**Status:** shipped 2026-05-24 — see [`decisions.md`](../decisions.md) entry 2026-05-24.

**Scope:** delete 48 files, edit 13 coupling points, remove admin button from `UserMenu`, narrow `PortalScope` union, drop `useAdminFilterStore` export, drop `getAdmin*` service functions, drop admin dialog IDs, move `filter.all.label` translation key from `ADMIN_PAGES` to a non-admin namespace, remove `ADMIN_PAGES` namespace entry. Backend SQL: one translation key relocation, plus eventual `ADMIN_PAGES` namespace deletion.

**Also includes P2** (delete stale Firebase Admin SDK file from repo root — key already invalidated).

**Spec source:** `oglasino-expo/.agent/audit-expo-readiness-admin-removal.md`. The audit is detailed enough to brief from directly — no further Phase 4 work needed. The chat opens by reading the audit, briefing the Mobile Engineer (one session), reviewing, then handling the one cross-repo translation seed via a Backend Engineer brief.

**Inputs the next chat needs:** the audit file above; conventions; `state.md`; `decisions.md`.

**Expected outcome:** admin surface fully removed, app builds clean, `tsc --noEmit` and lint pass, no behavioral change for non-admin users. Expo backlog gets no row (this isn't an adoption — it's deletion).

**Estimated effort:** one session.

---

### A — Mobile validation rebuild

**Scope (from audit):** mobile currently calls the old combined `POST /secure/product/addUpdate` endpoint for both create and update. The spec defines three separate endpoints (`create`, `update`, `pre-validate`). The mobile DTO still carries removed fields (`oldName`, `oldDescription`, `productState`, `moderationState`). The frontend uses translation keys in the `product.internal.*` pattern under the frozen `VALIDATION` namespace; the spec uses `product.<field>.<code_lowercase>` under `ERRORS`. Zero of 10 sampled error codes have matching keys. No pre-validate call. No status-code distinction. No per-field error parsing. The whole service layer rebuilds.

**Spec source:** `oglasino-docs/features/product-validation.md` already exists and is detailed.

**Inputs:** the spec; `oglasino-expo/.agent/audit-expo-readiness-product-validation.md`; image pipeline audit (for the integration touchpoint).

**Bucket-1 bugs that ride along:** B13 (wrong field on description errors), B14 (hardcoded banned-words list). Both disappear automatically when the duplicated client-side moderation gets deleted per the spec.

**Cross-repo dependency:** X3 must be answered. If the old endpoint was removed, mobile create/edit is currently broken against the validation-refactor backend, and the rebuild is more urgent. If it still exists as an alias, mobile works today against old behavior; the rebuild is still required.

**Estimated effort:** several sessions. This is the largest single feature chat in the queue.

---

### B — Mobile messaging adoption

**Scope:** the audit found **no release-blocking cross-user Firestore reads** — the highest-confidence risk going in turned out to be cleanly handled (mobile routes user-data enrichment through `GET /auth/firebase/{uid}`, not direct cross-user Firestore reads). The remaining work is spec adoption: atomic batch for existing-chat sends, send-failure toast, deleted-user fallback rendering, `tempProductReason` → `tempProductContext` rename, mark-seen `seenLocal` mutation fix, auto-linkify, block badge on chat list, "Load more" button, `deleteChat` reorder, forward-block field name alignment.

**Spec source:** `oglasino-docs/features/messaging.md`.

**Inputs:** the spec; `oglasino-expo/.agent/audit-expo-readiness-messaging.md`.

**Bucket-1 bugs that ride along:** B10 (NORMAL/NOTMAL typo — one-character fix while the engineer is in that file), B11 (`getActiveChat` fallback bug — same fix web applied), B12 (five hardcoded Serbian strings).

**Estimated effort:** medium — multiple sessions across read, write, listener, and UI layers.

---

### C — Mobile user-settings cleanup + consent-field removal

> **Merged into chat G (2026-05-30).** C and G both touch `app/owner/dashboard/user.tsx`, so the 2026-05-30 Mastermind chat merged them into one: C runs first (cleans the screen) and G builds the consent UI on it. No separate C chat opens. The merged scope — including the one scope change Igor made, **`allowNotifications` is now deliberately left alone** (not removed/wired; a future notifications feature owns it) rather than the "remove or wire" open question below — is canonical in [`consent-mode-mobile.md`](consent-mode-mobile.md) § Part C. The text below is the pre-merge planning sketch, retained for history.

**Scope:** remove `allowPreferenceCookies` end-to-end from mobile (6 occurrences across 3 files). Delete the "Strictly necessary" and "Preference cookies" toggle blocks (they reference deleted backend translation keys). Fix `allowPromoEmails` initialization bug (B2). Remove the dead `allowNotifications` toggle (B3) or wire it up if Igor wants it functional. Delete dead `ConsentData.ts` and `GlobalCookie.ts` types. Remove hardcoded `"DODAJ ZA BASE SITE"` placeholder (B17). Clean up Part 4 console-log violations on this surface.

This chat **does not design a consent UI**. That's chat G.

**Spec source:** `oglasino-docs/features/consent-mode-v2.md` — the field-removal section gives the contract. The cleanup-side scope is small enough that the audit + spec together are sufficient input.

**Inputs:** the spec; `oglasino-expo/.agent/audit-expo-readiness-consent-mode.md`; `.agent/audit-expo-readiness-user-deletion.md` (for the `AuthUserDTO` field cleanup overlap).

**Bucket-1 bugs that ride along:** B2, B3, B4, B17, B16 (the console-error subset on `user.tsx`).

**Open product question for the chat:** does Igor want `allowNotifications` removed or wired correctly? Cheap to decide in the chat's Phase 1.

**Estimated effort:** one to two sessions.

---

### D — Mobile filtering & search polish

**Scope:** fix `clearAllFilters()` to reset product/moderation state (B6). Fix search submit to preserve category context (B7). Fix `UserProductsList` filter bleed-through (B8). Replace the `<div>` runtime crash with `<View>` / `<Text>` (B5). Translate product/moderation state enum labels (B9). Optionally adopt web's `applyRandom` suppression when search or order is set. Decide on `excludeIds` for user-profile pages. Decide on wiring the deep-link filter parser (currently dead code with a `TODO`).

**Spec source:** `oglasino-docs/features/product-filtering-and-search.md`.

**Inputs:** the spec; `oglasino-expo/.agent/audit-expo-readiness-product-filtering.md`.

**Bucket-1 bugs that ride along:** B5, B6, B7, B8, B9.

**Estimated effort:** one to two sessions.

---

### E — Mobile user-deletion adoption

**Scope:** add `disabled`, `state`, `deletionStatus` to `AuthUserDTO`; add `state` to `UserInfoDTO`. Build the 401/403 interceptor that signs the user out on `USER_BANNED`. Build the ban-notice dialog. Build the deletion UI surface (settings screen → danger zone). Add `deletionInFlight` flag to `useAuthStore`. Add single-flight guard to `initAuthListener` (mobile equivalent of web's `UseTokenRefresh` fix). Fix `logoutFirebase` to call `auth.signOut()` (B1). Clean up the AsyncStorage cleanup gaps (`useViewTokenStore` on involuntary sign-out, `useFilterStore` admin-state leak). Decide on `ensureUserInFirestore` skip-for-returning-users optimization.

**Spec source:** `oglasino-docs/features/user-deletion.md` and `oglasino-docs/features/user-deletion-auth-contract.md`.

**Inputs:** both specs; `oglasino-expo/.agent/audit-expo-readiness-user-deletion.md`.

**Bucket-1 bugs that ride along:** B1.

**Cross-repo dependencies:** X1 and X2 should be answered before or during this chat. The push-token and user-update endpoints' trust-boundary behavior shapes what mobile sends and what the audit needs to verify.

**Estimated effort:** medium-large — this is auth-critical and touches many surfaces.

---

### F — Mobile analytics (GA v1)

**Scope:** install `@react-native-firebase/analytics` + `@react-native-firebase/app` + `expo-tracking-transparency`. Wire ATT prompt at app launch (iOS App Store prerequisite). Initialize the analytics SDK gated on ATT result (and on EU consent, see G). Instrument all 12 events from the spec at the mobile surfaces named in the audit. Add `wasRegister` to `AuthUserDTO` so mobile can distinguish `sign_up` from `login` (or adopt a different discriminator). Add `productId` prop to `CallUserButton` (mirror web's GA v1 brief 6 change). Delete dead `firebaseAnalytics.ts` (web SDK vestige).

**Spec source:** `oglasino-docs/features/google-analytics-v1.md`. Update its `**Status:**` line first (it currently says `in-progress-web` but is shipped — docs hygiene item ζ in section 6 handles this).

**Inputs:** the spec; `oglasino-expo/.agent/audit-expo-readiness-google-analytics.md`; the consent audit for context.

**Hard dependency:** **chat G or an ATT-only gating decision must precede this.** Apple will reject builds with analytics SDKs that don't show the ATT prompt. EU users need GDPR consent. Decision in Phase 1 of this chat: ship with ATT-only (legally thin for EU) or wait for G.

**Estimated effort:** medium.

---

### G — Mobile consent UI v1 (absorbs C)

> **Spec authored 2026-05-30** at [`consent-mode-mobile.md`](consent-mode-mobile.md). Option **(b)** was chosen — a single-axis, two-state device-local "Allow analytics" decision in AsyncStorage plus the `isAnalyticsConsentGranted()` gate — rejecting (a) ATT-only (leaves EU Android ungated) and (c) the four-signal web model (no cookies/SSR/Google Consent Mode on mobile). G also **absorbs chat C** (consent-field cleanup), which now runs as Part C of the merged spec. Phase 4 is complete; Phase 5 engineering briefs come next. The text below is the pre-spec planning sketch.

**Scope:** new feature. Design and build a mobile consent surface. Options considered in the chat's Phase 1: (a) ATT-only on iOS + no-op on Android (legally minimal); (b) Simple two-state opt-in/opt-out toggle stored in AsyncStorage + the relevant gating logic; (c) Full four-signal Consent Mode v2 model adapted to mobile primitives (no cookies — AsyncStorage equivalent). Whatever's chosen, the chat authors a spec at `oglasino-docs/features/consent-mode-mobile.md` in Phase 4, then briefs from there.

**Spec source:** [`consent-mode-mobile.md`](consent-mode-mobile.md) (authored 2026-05-30). The web reference is [`consent-mode-v2.md`](consent-mode-v2.md).

**Inputs:** the spec; the three 2026-05-30 read-only audits (`audit-consent-mode-mobile-expo.md`, `-web.md`, `-backend.md`, archived to `sessions/`); the prior `audit-expo-readiness-consent-mode.md` and `audit-expo-readiness-google-analytics.md` (for the ATT prerequisite framing).

**Estimated effort:** medium — Part C cleanup (one session) + Part G build (one to two sessions) + backend `mobile.consent.*` seed.

---

### H — Mobile image-pipeline polish

**Scope:** decide and execute on `expo-image` adoption (P3 — adopt for caching, or delete the unused package). Surface progressive status text in `UploadedProductDialog` (Model C decision mentioned it; mobile shows only a spinner). Remove stale `oldName`/`oldDescription`/`productState`/`moderationState` from `UpdateProductRequestDTO` (folds into A if A runs first; otherwise this chat does it). Delete dead `productUpdateNameValidator.ts`. Fix hardcoded Serbian strings in `ImageSourceSheet.tsx` and `ProductReviewImageImport.tsx` (B15). Decide on adding `AppState` cleanup for in-flight uploads, or leave as-is and rely on backend sweeper.

**Spec source:** none yet. **Phase 4 in this chat authors a Phase 4 spec** at `oglasino-docs/features/image-pipeline-general.md` from the existing decisions + the audit. The spec stays brief — it's a "documents the current contract" spec, not a re-design.

**Inputs:** `decisions.md` 2026-05-13 Model C entry; `oglasino-expo/.agent/audit-expo-readiness-image-pipeline.md`; `.agent/audit-expo-readiness-product-validation.md` (for the DTO overlap).

**Bucket-1 bugs that ride along:** B15.

**Order note:** runs after A if possible — A deletes the DTO fields that H would otherwise have to clean up.

**Estimated effort:** small to medium.

---

### I — Mobile backend-calls optimization

> **Shipped 2026-05-31.** Fresh Phase-2 audit confirmed the predecessors (Φ1, boot-redesign Gate 4, version-checksums, expo-maintenance-split) absorbed most of §I. Residual work shipped as [`backend-calls-reduction-mobile.md`](backend-calls-reduction-mobile.md): C1 (drop dead `idToken`), C2 (cache `getUserForId`), C4 (skip per-login Firestore read) on mobile; C5 (cache active-base-site codes, all clients) on backend. Per-pathname version check confirmed already gone (boot-redesign). See [`decisions.md`](../decisions.md) 2026-05-31.

**Scope:** translation caching — wire up the commented-out AsyncStorage cache, implement (or coordinate with backend on) the version-check endpoint. ~~Reduce maintenance polling from 5s to a saner interval.~~ (**Absorbed by `expo-maintenance-split`** — the dedicated 5s backend maintenance poll is removed entirely, replaced by the worker `X-Oglasino-Maintenance` signal in the response interceptor plus the re-pointed exit poller.) Remove version check from every pathname change (keep on cold start and foreground resume only). Add single-flight guard to `syncUserToBackend` (mobile equivalent of web's `UseTokenRefresh` fix — overlaps with E). Remove redundant `idToken` from `/auth/firebase-sync` body and the double `getIdToken()` call (after X5 is answered). Cache `getUserForId` results. Decide on `ensureUserInFirestore` skip-for-returning-users (overlaps with E). Decide on `expo-image` (overlaps with H).

**Spec source:** none yet. **Phase 4 in this chat authors a brief Phase 4 spec** at `oglasino-docs/features/backend-calls-reduction-mobile.md` from the audit findings.

**Inputs:** `oglasino-expo/.agent/audit-expo-readiness-backend-calls-reduction.md`; relevant decision entries.

**Order note:** the translation-caching item requires a backend version endpoint that doesn't exist. That endpoint is its own small backend feature — either Mastermind opens a separate backend chat to spec and ship it first, or chat I scopes around mobile-only optimizations and queues the version-endpoint work as a follow-up.

**Estimated effort:** medium.

---

## 6. Docs-hygiene follow-ups (Docs/QA chore)

These are documentation contradictions surfaced by the docs-side audit. They're not feature work and shouldn't block any chat — but they should run as a single Docs/QA chore session before too long, since stale docs will confuse future chats.

| # | Item | File |
|---|---|---|
| ζ1 | Update status from `in-progress-web` to `shipped` | `oglasino-docs/features/google-analytics-v1.md` |
| ζ2 | Archive or redirect the stale `account-disabling-enforcement` stub (folded into user-deletion per `decisions.md` 2026-05-18) | `oglasino-docs/features/account-disabling-enforcement.md` |
| ζ3 | Add `seo-foundation` to `state.md` (currently in `features/` but listed nowhere) | `oglasino-docs/state.md` |
| ζ4 | Delete stale `future/ga4-analytics-v1.md` remnant | `oglasino-docs/future/ga4-analytics-v1.md` |

Not in this chat. Igor opens a small Docs/QA chore brief when convenient.

---

## 7. Sequencing recommendation

**Updated 2026-05-24** after the structural audit inserted four foundation chats (Φ1–Φ4) and two closing chats (Ω, Ψ). See [`features/expo-structural-foundation.md`](expo-structural-foundation.md) for the foundation program spec.

The revised full queue:

1. **α** (admin removal) — **shipped 2026-05-24.**
2. **Φ1** (auth lifecycle foundation) — opens next. Fixes auth listener, logout, 401/403 interceptor, auth guards, hydration race. Every subsequent chat depends on auth being correct.
3. **Φ2** (navigation foundation) — replaces bare `<Slot />` with `Stack` and `Tabs`. Feature chats need the navigation shape in place.
4. **Φ3** (performance foundation) — React.memo, Zustand selectors, expo-image adoption, AppContext memoization. Needs navigation in place first (Tabs persistence changes re-render math).
5. **Φ4** (service layer + error contract) — Part 7 error contract across all 17 services. Makes chat A possible.
6. **A** (validation rebuild) — slightly smaller; Φ4 delivers the service layer A depends on.
7. **B** (messaging adoption) — unchanged scope.
8. **C** (user-settings cleanup) — **merged into G** (2026-05-30); runs as Part C of the merged chat, not as a separate chat.
9. **D** (filtering & search polish) — unchanged scope.
10. **E** (user-deletion adoption) — **massively smaller** after Φ1; most of E was auth plumbing (listener, logout fix, 401/403 interceptor, auth guards). E retains only the deletion UI surface.
11. **H** (image pipeline polish) — **somewhat smaller** after Φ3; expo-image adoption (P3) moved to Φ3. H retains image upload UX, stale DTO cleanup, hardcoded strings.
12. **G** (consent UI v1) — **absorbs C** (consent-field cleanup, runs first as Part C). Spec authored 2026-05-30. Must precede F.
13. **F** (analytics v1) — unchanged scope. Depends on G or ATT-only decision.
14. **I** (backend-calls optimization) — **shipped 2026-05-31.** Smaller than scoped; predecessors absorbed most of §I.
15. **Ω** (final structural sweep + dead-code removal) — new. Covers the audit's 18 out-of-scope observations.
16. **Ψ** (runtime verification pass) — new. QA-style chat to verify structural fixes on real devices. Last in queue.

Bug-fix-only chats are unnecessary — every Bucket 1 bug rides into the carrier chat as noted. The exception is if a bug needs to ship faster than its carrier chat opens, in which case Igor opens a one-off bug-fix session against that file.

---

## 8. Risk watch additions for `state.md`

After this chat closes, the following entries should appear in `state.md`'s Risk Watch section:

- **Mobile is multiple features behind, with a clear gap inventory and queue.** Replaces the older generic version. Reference this spec.
- **Existing mobile bugs deferred to feature chats.** B1 (session leak on logout) and B4 (broken settings UI) are the highest-severity. Both ship in their carrier chats (E, C). If either chat slips, Igor opens a one-off bugfix.
- **Cross-repo trust-boundary questions queued.** X1–X5 await the next backend audit. None are confirmed exploits; all are "verify before assuming clean."
- **Translation caching gap on mobile (21 calls/init).** Confirmed waste, deferred to chat I. Affects cold-start time on every app open.
- **5-second maintenance poll on mobile.** Confirmed 720 calls/hour. ~~Deferred to chat I.~~ Absorbed/superseded by `expo-maintenance-split` — the dedicated poll is removed (worker `X-Oglasino-Maintenance` signal + re-pointed exit poller).

---

## 9. What this chat does NOT decide

To stay in scope (per `meta/mastermind-bootstrap.md`):

- **No implementation.** Audits only.
- **No store/devops work.** Out of scope per Igor.
- **No backend or web changes.** Cross-repo questions logged in Bucket 4; the backend chat handles them.
- **No design for the future consent UI** (chat G).
- **No SDK choice locked in for analytics** (chat F's Phase 1 confirms `@react-native-firebase/analytics`, given `expo-firebase-analytics` is deprecated).
- **No commitment on which Bucket 1 bugs are release-blocking vs nice-to-have.** That decision belongs to Igor as carrier chats open and as regression testing surfaces real impact.

---

## 10. Definition of done for this chat

- This spec committed to `oglasino-docs/features/expo-release-readiness.md`.
- All eleven audit files archived to `oglasino-docs/sessions/`.
- `decisions.md` updated with the chat-closure entry.
- `state.md` updated: this feature in active features, Risk Watch refreshed, future-chat queue reflected in the backlog as appropriate.
- The chat closes.

Future chats reopen the work, one at a time, in the order Igor chooses.