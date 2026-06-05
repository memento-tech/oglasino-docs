# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-02
**Task:** Read-only audit. No code changes. Write notifications-feature findings (page/center, data shape, web push, the four events, seams) to `.agent/audit-notifications.md`.

## Implemented

- Read-only audit only — no code changed. Deliverable is `.agent/audit-notifications.md`.
- Mapped the full web notification surface: `src/notifications/` module + three initializers (`FirebaseWorkerInit`, `ForegroundPushInit`/`PushInitializer`, `UseTokenRefresh`) + the service worker.
- Established that the notifications page reads a **persistent Firestore subcollection** `notifications/{uid}/userNotifications` via a live `onSnapshot` (not a backend poll), with cursor paging and mark-all-seen on open.
- Documented the `AppNotification` shape (`type`/`categoryId` enums, untyped `data` bag) and the three click-nav handlers, including the SW-vs-app locale-strip inconsistency.
- Found the headline seam: web registers the FCM token to **two** sinks — backend `POST /secure/push/token` and Firestore `users/{uid}.fcmToken` — via two near-duplicate token-fetch impls; which one the backend uses is unverified.
- Confirmed the four events (favorite/follow/message/admin-ban) do nothing notification-related web-side; all are plain backend calls.

## Files touched

- `.agent/audit-notifications.md` (new, audit deliverable)
- `.agent/2026-06-02-oglasino-web-notifications-1.md` (this summary)
- `.agent/last-session.md` (exact copy of this summary)

No `src/` or `app/` files modified.

## Tests

- Not run — read-only audit, no code change touches any test path.

## Cleanup performed

- none needed (no code written).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (Audit deliverable produced; Mastermind/Docs-QA decide whether to log a notifications-feature scoping entry.)
- issues.md: no change authored by me. Surfaced candidates for triage are listed in "For Mastermind" below (dead test page; double token sink; SW locale-strip inconsistency) — drafted there, not written to issues.md.

## Obsoleted by this session

- Nothing obsoleted by this session (read-only). Separately, the audit *identifies* likely-dead code (`app/[locale]/test/notifications/page.tsx` calling a backend endpoint state.md records as deleted; duplicate token-fetch impls) but I did not delete anything — out of an audit's scope and pending cross-repo verification.

## Conventions check

- Part 4 (cleanliness): N/A — no code written. Existing dead-code candidates flagged, not touched.
- Part 4a (simplicity): N/A — no code written.
- Part 4b (adjacent observations): recorded in the audit's "Adjacent observations" section (dead test page, duplicate token fetch, unseen-count page cap, double onSnapshot, loadMore normalization gap).
- Part 6 (translations): N/A this session.
- Part 7 (error contract): N/A this session.
- Hard rule "verify file existence before relying on Read": every claim in the audit is anchored to a file:line I opened directly; cross-repo claims are marked [needs cross-repo verification] rather than asserted.

## Known gaps / TODOs

- All cross-repo questions (backend Firestore write shape, Firestore Rules `seen`-update permission, which push-token sink the backend reads, `data.navigate` prefix convention, prod SW config regeneration) are enumerated in the audit §5 and flagged [needs cross-repo verification]. I cannot answer them from this repo.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing (read-only audit).

- **Headline seam to resolve before scoping the feature:** web writes the FCM token to two independent places — backend `POST /secure/push/token` (`pushTokenService.ts`, driven by `UseTokenRefresh`) **and** Firestore `users/{uid}.fcmToken` (`setUserFcmToken`, driven by `ensureUserInFirestore` and the owner settings toggle). Decide the single source of truth; one path is dead weight. Needs backend confirmation of which sink the web-push sender reads.

- **Locale-prefix seam (the known issue):** in-app + foreground handlers strip the routing locale via `stripRoutingLocale(data.navigate)`; the service worker uses `data.navigate` raw (cannot import the app router). Standardize what the backend puts in `data.navigate` and make the SW consistent.

- **Candidate issues.md entries (drafted, not written — Docs/QA owns the file):**
  1. *(medium)* `app/[locale]/test/notifications/page.tsx` POSTs to `/public/notification/test`, which state.md (2026-05-20, messaging Brief 4) records as deleted backend-side. Dead/broken page; also has `console.error` and hardcoded copy. Candidate for deletion — pending confirmation the endpoint is truly gone.
  2. *(low)* Duplicate FCM token-fetch implementations (`devicePush.getFirebaseToken` vs `fcmClient.getFcmToken`) — consolidate when the token sink is unified.
  3. *(low)* `unseenCount` is computed from the first page of 10 only (`useNotifications.ts:68`), so the bell badge undercounts beyond 10 unseen; the `>100 → '...'` branch is unreachable.
  4. *(low)* `/notifications` page + `AuthNotificationButton` each open an independent `onSnapshot` on the same subcollection (the hook is per-call, not a shared store despite the `…Store` name).

- **Config-file impact / closure gate:** no config-file edit is required by this session. The above are drafts for Mastermind/Docs-QA to triage, not changes I made or am requesting be made as a precondition.
