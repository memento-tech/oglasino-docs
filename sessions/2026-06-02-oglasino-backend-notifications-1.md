# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-02
**Task:** Read-only audit. No code changes. Write your findings to `.agent/audit-notifications.md`. Inventory the CURRENT state of notification infrastructure for the in-app + push notifications feature (events: new message, product favorited, new follower, admin bans product).

## Implemented

- Read-only audit only — no production code touched. Full findings in
  `.agent/audit-notifications.md`.
- Inventoried the existing `notifications` package: a live pipe that writes an in-app
  notification to Firestore (`notifications/{firebaseUid}/userNotifications`) *and* fans
  out push to two platforms — Firebase **Web Push** (`DefaultFirebasePushService`) and Expo
  (`DefaultExpoPushService`) — driven by `@Async DefaultNotificationsService.sendAsyncNotification`.
- Mapped the four target events: #2 (favorite) is wired and live; #3 (follow) has a clean
  hook point (`DefaultFollowService.toggleFollow`) but fires nothing; #1 (message) and #4
  (admin ban product) have **no server-side hook** — messaging is Firestore-only server-side,
  and there is **no product-ban write-path at all** (`ModerationState.BANNED` is never
  assigned; admin product controller is search-only).
- Surfaced one Part 11 trust-boundary violation (`PushTokenDTO.userId` client-supplied, not
  derived from auth) and two real defects in the Firebase Web Push path (immutable-`Map.of()`
  `.put` throw on null `data`; null `categoryId` NPE on review-disapprove).
- Confirmed the suspected `fcmToken`-in-Firestore-users-doc duplication does **not** exist on
  the backend (zero `fcmToken` references repo-wide).

## Files touched

- `.agent/audit-notifications.md` (new, audit output — not production code)

## Tests

- None run — read-only audit, no code changed. (`./mvnw test` not applicable.)

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (this is a Phase-2 audit; Mastermind/Docs-QA own any pipeline status
  entry after seam analysis)
- issues.md: no change authored by me. Three findings are drafted below in "For Mastermind"
  for Mastermind to triage into `issues.md` (the push-token Part 11 violation, the two
  Firebase Web Push defects, the hard-coded `"rs"` language). I did not write to `issues.md`
  (engineer agents don't).

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; no debug logging/commented code added.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged — see "For Mastermind" (push-token trust boundary,
  two Firebase Web Push defects, hard-coded `"rs"` language, `navigate`/`navigation` key
  inconsistency).
- Part 6 (translations): N/A this session — no seeds added. Noted existing `notif.*`
  `BACKEND_TRANSLATIONS` keys (10 per locale, all four locales present) for the wired pipe.
- Other parts touched: Part 7 (error contract) — N/A (no endpoints changed); Part 11 (trust
  boundaries) — central to the audit, one live violation reported.

## Known gaps / TODOs

- I did not read web/expo/firestore-rules code (cross-repo, forbidden). The seam questions in
  audit §5 (which Firestore path clients read; whether a client writes `fcmToken`; platform
  token shapes) need the counterpart audits to close.
- I did not exhaustively confirm the exact `POST /api/secure/favorite` route mapping (read the
  facade hook, not the favorite controller annotation) — immaterial to the audit conclusions.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **No "Brief vs reality" challenge needed, but two brief assumptions were off and are worth
  noting:**
  1. The brief anticipated `NotificationCategoryId` as a data model with values to enumerate
     by caller — it is an enum (6 values: 4 are zero-caller stubs `MESSAGE`,
     `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`, `INFO`; live: `SAVED_PRODUCT`, `NAVIGATION`).
     The pipe is more built-out than "scoping a feature" implies — favorite + admin
     review/report already ship notifications.
  2. Brief event #4 ("admin bans a product → notify owner") assumes a ban operation exists. It
     does not — `ModerationState.BANNED` is never written anywhere; the admin product endpoint
     is search-only. The ban capability must be **built** before a notification can hang off
     it. This is a design decision, not a wiring task.

- **Findings to triage into `issues.md` (drafted; engineer did not write the file):**
  - **(medium, Part 11)** `POST /api/secure/push/token` trusts client-supplied
    `PushTokenDTO.userId` as the FK + notification routing key — no check against
    `SecurityContextHolder`. A caller can register their device token under a victim's id and
    receive the victim's push stream. Fix: derive `userId` from
    `currentUserService.getCurrentUserIdStrict()`, drop it from the DTO (or reject mismatch).
    Same class on `/push/token/detach` (raw-token delete, no ownership check — lower impact).
  - **(medium)** `DefaultFirebasePushService` (Web Push) throws on any notification with
    `data == null`: `Map.of()` (immutable) + `.put` → `UnsupportedOperationException`,
    swallowed → silent no-push. Latent for every future null-`data` notification.
  - **(medium)** `DefaultAdminReviewService.sendFailNotification` leaves `categoryId` null →
    `DefaultFirebasePushService:58` NPEs (swallowed). Combined with the above, web recipients
    get no push on review-disapproval (Firestore doc still written).
  - **(low)** Hard-coded recipient language `"rs"` in `DefaultAdminReviewService`
    target-user/fail notifications and reviewer button labels — non-RS users get Serbian text.
  - **(low)** `data`-key inconsistency: report uses `"navigate"`, reviews use `"navigation"`.

- **Suggested next step:** Phase-3 seam analysis should pair with web/expo/firestore-rules
  audits on the §5 seams (Firestore read path + rules, push-token platform shapes, `fcmToken`
  client duplication), and treat #1 (message) and #4 (ban) as build-not-wire — each needs a
  new server-side capability before a notification exists to send.
