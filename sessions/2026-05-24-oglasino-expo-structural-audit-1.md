# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-24
**Task:** Read-only structural audit of `oglasino-expo` across seven categories (web-pattern-leak, RN-convention, state-shape, web-seam, lifecycle, performance, platform-API). No code changes.

## Implemented

- Produced `.agent/audit-expo-structural.md` — a 26-finding structural audit covering all seven categories from the brief.
- Cross-referenced all Bucket-1 bugs (B1–B18) from `expo-release-readiness.md` against current codebase state; confirmed 7 still present, noted in the audit's "Bucket-1 bugs noticed in passing" section.
- Identified 3 cross-repo questions (Q1–Q3), one new (Q2 — ban response shape) and two re-confirming readiness audit questions (X1, X3).
- Mapped each finding to the existing future-chat queue (A–I) or explicitly marked as not mapping.

## Files touched

- `.agent/audit-expo-structural.md` (+new, ~600 lines)

## Tests

- N/A — read-only audit, no code changes.

## Cleanup performed

- None needed — read-only session.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing. This session produces a new audit artifact; it does not obsolete any prior work.

## Conventions check

- Part 4 (cleanliness): N/A — read-only, no code changes
- Part 4a (simplicity): N/A — read-only, no abstractions introduced
- Part 4b (adjacent observations): confirmed — 18 out-of-scope observations logged in the audit's "Out-of-scope observations" section
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- Runtime verification was not possible (brief prohibits running the app). Performance findings (F7, F8, F25) need profiler traces to quantify actual impact.
- Image upload pipeline (`uploadImages.ts`) was not audited deeply.
- i18n system internals (translation caching, 21-calls-per-init) were not re-investigated.
- Deep-link behavior was not verified at runtime.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Audit headline:** The mobile app is structurally wrong for the platform in three dominant areas: (1) **lifecycle gaps** — no auth re-validation on cold start or foreground resume, `logoutFirebase` doesn't call `auth.signOut()`, `initAuthListener` never called, no 401/banned-user interceptor, no network state detection; (2) **performance shape** — zero `React.memo`, `expo-image` installed but unused, whole-store Zustand subscriptions, AppContext not memoized; (3) **RN convention violations** — all layouts use bare `<Slot />` with no native navigators, message list not inverted, `<div>` elements in RN code.

- **Top 5 most consequential findings:**
  1. **F1 — `initAuthListener` never called.** The auth store defines a Firebase `onAuthStateChanged` listener that syncs users to the backend, but it is never invoked. The app relies entirely on stale AsyncStorage data for auth state.
  2. **F2 — `logoutFirebase` doesn't call `auth.signOut()`.** Firebase session persists after logout; axios interceptor continues attaching the old user's Bearer token.
  3. **F5 — No 401/403 response interceptor.** Banned/deleted users are never force-logged-out.
  4. **F12 — All layouts use `<Slot />`; no native navigators.** No native transitions, no swipe-back, no tab state preservation. The app feels like a web SPA.
  5. **F8 + F9 — Zero React.memo and expo-image unused.** The product feed — the app's primary surface — re-renders every card on every state change and has no image disk caching.

- **Sequencing recommendation for Mastermind:** The audit surfaced cross-cutting performance and navigation issues (F7, F8, F9, F10, F12, F25) that don't map to any single chat in the A–I queue. Recommend inserting a **"mobile structural foundation"** chat before chat A, scoped to: native navigators (F12), auth guards (F13), expo-image adoption (F9), React.memo on list items (F8), Zustand selector refactoring (F7). This establishes the structural floor that all subsequent feature chats build on. Without it, feature chats will build on the wrong navigation and performance foundations, requiring rework.

- **Process note:** The audit was conducted via six parallel read-only investigations covering auth/lifecycle, stores/state, lists/performance, services/web-seams, navigation/UI, and messaging/chat. This parallelization was effective — the six areas are largely independent, and the synthesis step (cross-referencing findings across areas) identified several compound issues (e.g., Finding 3 combines logout gaps from the auth investigation with store-cleanup gaps from the stores investigation).
