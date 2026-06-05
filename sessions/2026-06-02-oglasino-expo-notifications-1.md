# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-02
**Task:** Read-only audit. No code changes. Write findings to `.agent/audit-notifications.md`. Scope the notifications feature (in-app + mobile push): the notifications screen, push config inventory, notification data shape, deep-link handling, the four events, and seams.

## Implemented

- Nothing implemented — this is a Phase-2 read-only audit. Sole deliverable is `.agent/audit-notifications.md`.
- Audited the in-app notifications surface (screen → store → Firestore client) and the push surface (`PushNotificationsInit` → `pushNotificationRegister` → `pushTokenService` → backend routes) end-to-end against the actual code.
- Diagnosed the "unstable / flickers in and out" symptom as listener re-subscribe driven by `user` object identity in two effects' deps, compounded by `initialLoad` replacing the notifications array wholesale and a transient `set({user:null})` calling `reset()`. Explicitly ruled out whole-store subscription (selectors are scoped) and reduced hydration-race to a minor contributor.
- Inventoried push as substantially configured in code/app.config (client path is real, not stubs); the genuine gap is out-of-repo EAS/Firebase credentials + backend send-side. Flagged this as a Brief-vs-reality item (brief assumed push unconfigured).

## Files touched

- `.agent/audit-notifications.md` (new, audit deliverable) — no source code touched.

## Tests

- Ran: none. Read-only audit; no code changed, so lint/tsc/test gates do not apply.
- Result: N/A
- New tests added: none. (Noted in the audit: the repo has zero notification tests.)

## Cleanup performed

- none needed (no source code touched).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change required by this session. (The notifications feature has no Expo-backlog row to retire here — this is the Phase-2 audit that precedes spec/adoption. If/when notifications adoption ships, a backlog-row edit would be drafted then. No unstated config-file dependency.)
- issues.md: no change. Existing relevant entries already present: 2026-06-02 "Notifications are unstable" (open, item logged for the notifications feature) and the 2026-06-01 cold-start/LHNR fix. This audit characterizes the root cause of the former; no new entry authored (out-of-scope findings are surfaced in the audit + "For Mastermind" for triage rather than self-filed).

## Obsoleted by this session

- nothing (read-only audit).

## Conventions check

- Part 4 (cleanliness): N/A — no source code changed.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): several flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no translation keys added/changed; the screen uses existing COMMON/BUTTONS/DIALOG keys).
- Other parts touched: Part 11 (trust boundary) — flagged the client-supplied `userId` on `/secure/push/token` for backend reconciliation. Part 8 (routes reusable across web/mobile) — noted web is the reference for the notification shape/enums.

## Known gaps / TODOs

- Out-of-repo items the audit could not verify: EAS push credentials (APNs/FCM v1), backend `/secure/push/token[/detach]` route existence + send-via-Expo-Push-API, Firestore security rules for `notifications/{uid}/userNotifications`. These belong to backend / firestore-rules / EAS audits in the seam-analysis phase.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code added.
  - Considered and rejected: nothing — audit only.
  - Simplified or removed: nothing — audit only.

- **Flicker root cause (the brief's headline question).** Listener re-subscribe on `user` object identity (`NotificationsInit` dep `[hasHydrated, user]`; the screen dep `[user]`), where `authStore` replaces `user` with a fresh object on token refresh / foreground re-sync / login. Each swap tears down the Firestore listener and re-runs `initialLoad`, which **replaces** the array with ≤20 docs — so anything `loadMore`'d past 20 disappears. A transient `set({user:null})` triggers `NotificationsInit`'s `reset()`, blanking the list. Whole-store subscription is NOT the cause; hydration race is minor. Full detail with file:line in the audit §1.

- **Part 4b adjacent observations (for triage, not self-filed):**
  - **(medium)** Category id mismatch — iOS category registered as `NAVIGATE` (`PushNotificationsInit.tsx:30`) but handled as `NAVIGATION` (`:89`); `NAVIGATE` is not in the `NotificationCategoryId` enum. A `NAVIGATE` push falls to the no-op `default`. Out of scope for an audit; logged.
  - **(medium)** Double subscription — `NotificationsInit` subscribes app-wide and the screen subscribes again on mount (`notifications.tsx:32-38`). Two live listeners on the same query; the screen's is redundant (it never `initialLoad`s).
  - **(low/medium)** Realtime `subscribe` only *adds* docs — never reflects `seen` flips or deletes from other devices (`useNotificationStore.ts:90-107`); and it overwrites the `limit(20)` pagination cursor with its own `limit(10)` doc when new items arrive (`:103`), which can skip/duplicate rows on scroll.
  - **(low)** `initialLoad` passes `null` to a `startAfter`-based query (`useNotificationStore.ts:43` → `firebaseNotifications.ts:78`); not the intended first-page form. Verify on-device it doesn't throw/misbehave.
  - **(low)** Dead/vestigial cross-platform scaffolding: `markNotificationAsShown` has no consumer; the `nonShown`/`shown` filter is never exercised; `FirebaseNotification.link` is declared but never rendered. Reconcile against web (honor or drop).
  - **(low)** Two different product-URL helpers for the same SAVED_PRODUCT category: push-tap `getNormalizedProductUrl` vs in-app `getDashboardNormalizedProductUrl`.
  - **(low)** `loadMore` never toggles `loading`, so the `!loading` guard in `onEndReached` doesn't throttle it and the footer spinner only shows on initial load.

- **Trust boundary (Part 11):** `/secure/push/token` receives a client-supplied `userId` (`pushTokenService.ts:3-9`). Backend should derive the owning user from the authenticated identity, not trust the body `userId`. For the backend seam audit.

- **Brief vs reality:** push is substantially configured (client path real; `app.config.ts` declares plugin, background mode, RNFirebase, per-tier google-services, EAS projectId). The real unknowns are out-of-repo (EAS/Firebase credentials, backend send-side). Suggest the canonical spec frame the mobile push work as "verify credentials + backend + fix the small in-code gaps (NAVIGATE/NAVIGATION, missing categories, dead shown/link)", not a from-scratch build.

- **Suggested seam-analysis verification set:** (1) backend notification-write field/enum parity (esp. NAVIGATION); (2) `/secure/push/token[/detach]` DTO field names — note attach sends `token`, detach sends `pushToken`; (3) Firestore rules for read + `seen` update on `notifications/{uid}/userNotifications`; (4) push send path uses Expo Push API with matching `categoryIdentifier`/`data`; (5) EAS push credentials.
