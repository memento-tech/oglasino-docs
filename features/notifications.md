# Notifications

In-app and push notifications for Oglasino across web and Expo, with the backend as the
single notification authority. This spec reflects the audited current state (Phase 2,
2026-06-02) plus the seam resolutions confirmed in Phase 3.

**Status:** web-stable + mobile-stable — **COMPLETE**, verified on real devices + web by Igor 2026-06-02 (all four events — favorite, follow, ban/unban, message incl. text **and** image-only — on iOS, Android, and web). backend (`dev`), firestore-rules (`stage`), web (`dev`), mobile (`new-expo-dev`).
**Repos touched:** oglasino-backend, oglasino-firestore-rules, oglasino-web, oglasino-expo
**Out-of-repo dependency:** EAS/Firebase push-credential provisioning — development tier done (stage FCM v1 + iOS APNs key); preview/prod tiers owed when they ship (non-feature follow-up; preview reuses the stage key, prod needs the prod key — do not cross them)

---

## 1. Summary

A notification pipeline already exists and is partially live. The backend writes an in-app
notification document to Firestore and fans out push to two platforms (Firebase Web Push for
web, Expo Push for mobile). It is wired today to: product-favorited, admin review
approve/disapprove, and admin report-resolve. This feature:

- Wires the two missing target events (new follower; admin bans product).
- Adds message-send push notifications (push-only, no in-app doc).
- Closes a CRITICAL push-token trust-boundary hole (Part 11).
- Fixes two defects in the live web-push path.
- Removes a dead parallel web token-registration path.
- Fixes the Expo notification-list flicker.
- Standardizes cross-platform shape drifts.

The architecture does not change: the Spring backend owns all notification writes via the
Firebase Admin SDK; clients are consumers (read the Firestore in-app list, receive push,
mark-seen, navigate on tap) plus push-token registrants. No Cloud Functions are introduced.

---

## 2. The four target events

| #   | Event                | In-app doc? | Push? | Trigger                             | Status going in             |
| --- | -------------------- | ----------- | ----- | ----------------------------------- | --------------------------- |
| 1   | New message received | **No**      | Yes   | client→backend ping on send         | new                         |
| 2   | Product favorited    | Yes         | Yes   | `DefaultFavoriteProductFacade`      | already live                |
| 3   | New follower         | Yes         | Yes   | `DefaultFollowService.toggleFollow` | hook exists, fires nothing  |
| 4   | Admin bans product   | Yes         | Yes   | new admin ban endpoint              | ban operation must be built |

### 2.1 Message (#1) — push-only

Messages produce **no in-app notification document**. The user already has the messages page;
a bell-list entry duplicating it is redundant. Messages deliver as a **push banner only**.

- **Trigger:** client→backend ping. When the web or Expo chat store commits a message to
  Firestore, a client **emitter** fires a **post-commit, fire-and-forget** ping
  (`POST /api/secure/notifications/message-sent`, `{chatId, messageText}` only — never a
  client-supplied recipient). The backend is the trust boundary: it verifies (via Admin SDK read
  of the chat doc) that the authenticated caller is a participant of that chat and derives the
  recipient from the chat doc. It does not trust a client-supplied "I sent to X." As-built note:
  the emitter was added late (web W4, expo E3) — the receiver endpoint (backend B4) was built
  first and the on-send emitter was initially missing; caught on-device and fixed before
  `mobile-stable` (see [decisions.md](../decisions.md) 2026-06-02 process note).
- **Anti-spam (per-chat OS collapse key):** Anti-spam is delivered by a per-chat OS collapse
  key: the backend sends a push on every message-ping, and the OS replaces the prior banner for
  that chat (Android `collapseKey` / `tag`, iOS `apns-collapse-id`, Expo `collapseId`; Firebase
  Web Push sets both the RFC-8030 `Topic` header and the `Notification` tag). The collapse key
  is derived from the chatId (hashed to fit the ≤32-char Web Push `Topic` limit). The result the
  user sees — one banner per conversation that updates to the latest message — is delivered by
  OS collapse, not by a server unread-state gate. The chat doc carries no server-readable
  per-recipient unread state, so a server-authoritative first-unread gate is not possible without
  new presence infrastructure (out of scope); active-viewing suppression is a deferred
  client-side concern. Collapse is best-effort per platform; there is no in-app fallback surface
  for messages by design.
- **Push banner content:** the actual message text (e.g. "Ana: is the bike still available?").
  This is a deliberate choice — message bodies transit the Expo/FCM/APNs push infrastructure
  and show on a locked phone, the universal messenger norm, accepted here for messaging
  specifically. The in-app analytics PII rules are unaffected (no in-app doc, no analytics).
- **Image-only (no-text) messages:** both clients send `messageText: ''` (empty string, key
  present — a consistent wire shape; the client never fabricates a body) and the backend (B6)
  supplies a localized photo body `notif.message.photo.body` (EN "📷 Photo") in the recipient's
  `preferredLanguage`. The message-ping DTO accepts empty `messageText` (no longer `@NotBlank`);
  `chatId` stays required.

### 2.2 Favorite (#2) — already live, no change

`POST /api/secure/favorite` → `DefaultFavoriteProductFacade.addRemoveFromFavorites` fires a
`SAVED_PRODUCT` notification to the product owner on add. Recipient is server-derived
(`product.getOwner().getId()`). In-app + push. This feature does not change it except to
benefit from the cross-cutting defect fixes in §6.

### 2.3 Follow (#3) — wire the existing hook

`POST /api/secure/follow/{targetUserId}` → `DefaultFollowService.toggleFollow` returns `true`
exactly when a new follow row is created. Fire a `sendAsyncNotification` there. Recipient is
the followed user (`targetUserId`, server-validated as an FK). Actor is the current user. The
notification rides the `NAVIGATION` category (in-app tap → the follower's profile). New
`BACKEND_TRANSLATIONS` keys required. In-app + push.

### 2.4 Ban (#4) — build the ban operation, then notify

There is no product-ban capability today. `ModerationState.BANNED` exists in the enum but is
never assigned in `src/main`; the admin product controller is search-only. This event requires
building the operation before the notification can hang off it.

**Ban (new admin endpoint):**

- Sets `moderationState = BANNED` AND `productState = INACTIVE`.
- While `moderationState == BANNED`, the owner's activate path (`INACTIVE → ACTIVE`) is
  refused with a coded error. The owner cannot un-ban their own product. This mirrors the
  banned-user content-hiding precedent (a system-imposed state the owner cannot undo).
- Recipient of the notification is the product's owner, server-derived
  (`product.getOwner().getId()`), never client-supplied.
- In-app + push.

**Unban (mirror admin endpoint):**

- Sets `moderationState = APPROVED`. The owner regains control of `ACTIVE`/`INACTIVE`.
  (Unban does not auto-activate; it returns the product to the owner's control in its current
  `INACTIVE` state.)
- Notify the owner on unban ("your product was restored"). `[inferred — confirm unban
notification is wanted; low stakes]`

**Owner-activate guard:** the existing owner activate path must check `moderationState != BANNED`
before allowing `INACTIVE → ACTIVE` and return a coded error (Part 7) when refused. New
`ERRORS` key for the refusal.

---

## 3. Architecture (unchanged)

- **Backend is the sole notification writer.** All in-app notification documents are written by
  the Spring backend via the Firebase Admin SDK, which bypasses Firestore security rules.
- **In-app store:** Firestore subcollection `notifications/{firebaseUid}/userNotifications`,
  documents ordered by `createdAt desc`. `firebaseUid` is resolved server-side from the
  numeric `userId` via `UserService.getFirebaseUidForUserId`.
- **Clients read the Firestore subcollection directly** via realtime listeners. There is no
  backend GET endpoint for the notification list. Clients flip `seen` via `updateDoc`.
- **Push:** the backend loads the recipient's push tokens from the Postgres `push_token` table
  and fans out per platform — `FIREBASE` (web, Firebase Admin SDK Web Push) and `EXPO` (mobile,
  Expo Push API over HTTP).

---

## 4. Notification document shape (in-app)

The canonical shape written by the backend and read by both clients:

notifications/{firebaseUid}/userNotifications/{autoId}
title: string
description: string
seen: boolean (default false)
createdAt: Timestamp (server timestamp)
type: NotificationType // NORMAL | SUCCESS | DANGER | WARNING
categoryId: NotificationCategoryId
data: map // category-specific

- `data` per category: `SAVED_PRODUCT` → `{ productId, productName }`; `NAVIGATION` →
  `{ navigate }`. Other categories carry no required `data` keys.
- The `data` navigation key is **`navigate`** everywhere. The backend review path's current use
  of `navigation` is corrected to `navigate` (§6).

### 4.1 Canonical category set

`NotificationCategoryId`: `MESSAGE`, `SAVED_PRODUCT`, `NAVIGATION`. The enum may retain
`PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`, `INFO` as stubs (no live producer), but they are not
part of this feature's contract. The canonical set in active use after this feature is:
`SAVED_PRODUCT` (favorite), `NAVIGATION` (follow, ban, unban), `MESSAGE` (push-tap nav only —
never an in-app doc).

**`MESSAGE` is a push-tap navigation target, not an in-app render category.** No message ever
produces an in-app document. Both clients keep the push-tap → `/messages` navigation but
remove `MESSAGE` from in-app-list rendering.

`NotificationType`: `NORMAL`, `SUCCESS`, `WARNING` are in use; `DANGER` is available.

---

## 5. Trust boundaries (Part 11)

### 5.1 Push-token endpoints — CRITICAL fix

`POST /api/secure/push/token` and `POST /api/secure/push/token/detach` currently trust a
client-supplied `userId` / token-string for the ownership key. This is a Part 11 violation: a
caller can register their device token under a victim's `userId` and receive the victim's
notifications.

**Fix:**

- Derive the owning user from `currentUserService.getCurrentUserIdStrict()`.
- Remove `userId` from `PushTokenDTO`. Both clients stop sending it.
- `detach` must verify the token row belongs to the authenticated user before deleting.

### 5.2 Recipient derivation (all events)

Every notification recipient is server-derived and must remain so:

| Event       | Recipient source                                                   |
| ----------- | ------------------------------------------------------------------ |
| Message     | recipient verified as chat participant via Admin SDK chat-doc read |
| Favorite    | `product.getOwner().getId()` (DB)                                  |
| Follow      | `targetUserId` path var (server-validated FK)                      |
| Ban / Unban | `product.getOwner().getId()` (DB)                                  |

### 5.3 Message-ping participant check

The message-ping endpoint must confirm, via Admin SDK read of the chat document, that the
authenticated caller is a participant of the chat they claim to have sent to. The client's
claim of recipient/chat is not trusted on its own.

---

## 6. Backend defects to fix (existing pipe)

1. **Null-`data` throw (medium):** `DefaultFirebasePushService` initializes `dataMap` to an
   immutable `Map.of()` then calls `.put()`, throwing `UnsupportedOperationException` (swallowed)
   whenever `data == null`. Result: web push silently sends nothing on null-`data`
   notifications. Fix: initialize `dataMap` mutable / handle null.
2. **Null-`categoryId` NPE (medium):** the review-disapprove notification sets no `categoryId`,
   NPE-ing the web-push path. Fix: set a non-null `categoryId` on that notification. (Combined
   with #1, a web recipient of a review-disapproval currently gets no push.)
3. **Hard-coded `"rs"` language (low):** several admin notifications pass literal `"rs"` to
   `getBackendTranslation` instead of the recipient's `preferredLanguage`. Fix to use the
   recipient's language.
4. **`data`-key inconsistency (low):** report uses `navigate`, reviews use `navigation`.
   Standardize on `navigate` (clients read `navigate`).

---

## 7. Firestore rules

The notifications rule is **not buggy**. `create: if false` (Admin-SDK-only) plus
recipient-only `read`/`update`/`delete` is correct, and confirmed by 9 passing tests.

**One change:** the `update` rule has no field constraints — the recipient can rewrite any
field, not just toggle `seen`. Tighten it to permit changing **only `seen`** — the recipient
may change only `seen`, enforced via the `request.resource.data.diff(resource.data).affectedKeys().hasOnly(['seen'])`
idiom plus a `seen is bool` type-guard. This `hasOnly` idiom was chosen over per-field equality
pinning (the idiom the `messages` update rule uses) because the notification doc's full field set
is owned by the backend Admin SDK and is cross-repo-unverifiable from the rules repo, so the two
sibling rules use different idioms by design (see [decisions.md](../decisions.md) 2026-06-02
`affectedKeys().hasOnly` entry). This is a correctness guard, not a security fix (self-only blast
radius today). New test for the field-lock.

No new index is required for a simple `createdAt desc` per-user list (served by automatic
single-field indexes). If a client adds a `seen`-filtered + `createdAt`-ordered query, a
composite index is owed — not in scope unless a client query requires it.

---

## 8. Web changes

1. **Delete the dead Firestore token sink.** Web registers the FCM token to two places: the
   Postgres `push_token` table (live, read by the backend send-side) and
   `users/{uid}.fcmToken` in Firestore (dead — the backend never reads it). Remove the
   Firestore path entirely: `fcmClient.getFcmToken`, `setUserFcmToken`, the
   `ensureUserInFirestore` token write, and the owner-settings toggle's Firestore write.
   Consolidate the two near-duplicate token-fetch implementations into one (the `devicePush`
   one). **This closes the long-standing fcmToken-location open issue — Postgres is the sink.**
2. **Stop sending `userId`** in the push-token attach call (§5.1).
3. **Owner-settings notifications toggle = push-only off-switch.** Off → detach (delete) the
   user's push tokens from the backend so no push is sent. In-app list keeps working. On →
   register the token via the existing backend attach path. The toggle no longer touches
   Firestore.
4. **Service-worker locale consistency.** The in-app and foreground handlers strip the routing
   locale from `data.navigate`; the service worker does not. Standardize: define whether
   `data.navigate` is locale-prefixed (backend contract), then make all three handlers
   (in-app, foreground, SW) consistent. `[inferred — backend should emit unprefixed
data.navigate; confirm in backend brief]`
5. **Remove `MESSAGE` from in-app-list rendering** (keep push-tap nav to `/messages`).
6. **Delete the dead test page** `app/[locale]/test/notifications/page.tsx` (POSTs to the
   already-deleted `/public/notification/test` endpoint).

---

## 9. Expo changes

1. **Fix the notification-list flicker.** Root cause: the Firestore listener re-subscribes on
   every `user` _object_ identity change (token refresh / foreground re-sync produce a fresh
   `user` object), and each re-subscribe replaces the notifications array wholesale with a
   fresh ≤20-doc page — so rows past the first page vanish ("visible one moment, gone the
   next"). Fix:
   - Subscribe keyed on `user.firebaseUid` (the stable string), not the `user` object.
   - Remove the screen's redundant second subscription (the app-wide `NotificationsInit`
     already subscribes; the screen subscribing again creates two live listeners on the same
     slice).
   - Make the realtime merge reconcile updates and deletes (`seen`-flips, removals from other
     devices), not only additions.
   - Fix the mixed page-size cursor corruption (`initialLoad`/`loadMore` use `limit(20)`; the
     realtime `subscribe` uses `limit(10)` and overwrites `lastDoc`). Use a dedicated first-page
     query (no `startAfter(null)`).
2. **Stop sending `userId`** in the push-token attach call (§5.1).
3. **Reconcile the category mismatch.** iOS registers a `NAVIGATE` category that is handled
   nowhere (the handler and in-app render use `NAVIGATION`). Fix to `NAVIGATION`.
4. **Remove `MESSAGE` from in-app-list rendering** (keep push-tap nav to `/messages`).
5. **Remove dead scaffolding:** `markNotificationAsShown`, the `nonShown` filter, and the
   unrendered `FirebaseNotification.link` field have no consumers. Drop them (or, if web ends
   up using `link`, reconcile — but current decision is drop, no live producer).

---

## 10. Out-of-repo: push credential provisioning (Igor, console)

The Expo client push path is fully coded (permission flow, `getExpoPushTokenAsync`, backend
registration, refresh listeners, tap handling) and `app.config.ts` declares the
`expo-notifications` plugin, iOS `remote-notification` background mode, per-tier
google-services files, and the EAS projectId. The genuine gap is **credential provisioning**,
not code:

- **APNs key** (iOS) registered against the Expo project.
- **FCM v1 credentials** (Android) uploaded to the Expo project.

Without these, `getExpoPushTokenAsync` throws on a real device. This is console work in the
EAS dashboard / Firebase console, owned by Igor — not an engineer brief. It is the item the
previous Mastermind deferred ("skip push config until this chat").

Web push depends on `NEXT_PUBLIC_FIREBASE_VAPID_KEY` and a registered service worker (both
exist). The committed generated `firebase-messaging-sw.js` carries stage Firebase config;
confirm the prod build regenerates it with prod config.

---

## 11. Translations

New `BACKEND_TRANSLATIONS` keys for: follow notification (title/body), ban notification,
unban notification, and the image-only message photo body (`notif.message.photo.body`, added by
B6 — EN final "📷 Photo"; placeholders RS/CNR "📷 Fotografija", RU "📷 Foto"). New `ERRORS` key for
the owner-activate-while-banned refusal. Text-message push needs no translated string (title =
sender name, body = the sender's own message text). EN final; RS/RU/CNR placeholder pending
native-translator review (the standing project precedent — 33 placeholder rows total; see
[state.md](../state.md) Risk Watch). Per conventions Part 6 Rule 3, seeded via the backend SQL
append; a dedicated SQL file if the count crosses the multi-namespace threshold.

---

## 12. Out of scope

- Email notifications (no email channel in v1).
- Account state-change notifications (banned/restored user) — the user is logged out
  immediately and cannot receive an in-app notification; deliberately dropped.
- Product approve/reject notifications — products auto-approve; admin only bans.
- The `PRODUCT_EXPIRATION` / `PRODUCT_EXPIRED` / `INFO` categories — stubs, no live producer,
  not part of this contract.
- Cloud Functions / Firestore triggers — the architecture stays Spring + Admin SDK.

---

## 13. Definition of done

- Push-token endpoints derive `userId` from auth; both clients stop sending it (CRITICAL).
- Follow and ban/unban events fire notifications; favorite unchanged.
- Message-send delivers a collapse-keyed push with message text (image-only → backend-supplied
  localized photo body), anti-spam via the per-chat OS collapse key, no in-app doc. Fired by the
  client emitter post-commit (web W4, expo E3/E5).
- Admin ban flips `BANNED` + `INACTIVE`; owner cannot re-activate while banned; admin unban
  restores `APPROVED`; owner-activate guard returns a coded error.
- Two backend push defects fixed; `data` key standardized to `navigate`; admin notification
  language uses recipient's `preferredLanguage`.
- Firestore `update` rule tightened to `seen`-only with a test.
- Web dead Firestore token sink removed; toggle drives backend detach; SW navigate handling
  consistent; dead test page deleted.
- Expo flicker fixed (subscribe on `firebaseUid`); category mismatch and dead scaffolding
  reconciled.
- Push credentials provisioned in EAS/Firebase (Igor) and a real-device push verified on iOS
  and Android.
- Translations seeded (EN final, others placeholder).

---

## 14. Session log

- **2026-06-02 — feature close-out (config-file application, Docs/QA).** All four engineering
  lanes reached code-complete and the close-out config writes were applied: two
  [decisions.md](../decisions.md) 2026-06-02 entries (message push-only + per-chat OS-collapse
  anti-spam; the Firestore-update field-lock idiom `affectedKeys().hasOnly([...])`), the
  [issues.md](../issues.md) resolutions (push-token sink ambiguity; ban/unban deep-link
  route-shape) and carry-forward items, the [state.md](../state.md) Notifications feature block +
  Expo-backlog row + Risk Watch rows, and the §2.1 mechanism correction above (server-unread-state
  wording replaced with the as-built per-chat OS-collapse mechanism). Status by lane: backend
  code-complete on `dev` (B1 push-pipe hardening + token trust fix; B2 follow; B3 ban/unban +
  owner-activate guard; B4 message push; B5 navigate-path standardization; + cleanup);
  firestore-rules code-complete on `stage` (seen-only update field-lock); web code-complete on
  `dev` (W1 token plumbing; W2 consumption; W3 follow-ups); expo code-complete on `new-expo-dev`
  (E1 flicker fix; E2 reconciles/cleanup). Web is `web-stable`; mobile is **not** `mobile-stable`
  — on-device push verification (Ψ) is owed, pending EAS/Firebase push-credential provisioning
  (APNs key + FCM v1; Igor, console — §10).
- **2026-06-02 — close-out corrections (Docs/QA, Mastermind-confirmed).** Applied the four edits the
  first close-out pass flagged but could not make without confirmation: (1) confirmed the backend
  cleanup session landed (consolidated data-key literals into `NotificationConstants`, removed dead
  Logger fields, folded `DefaultMessageNotificationService`'s local `NAVIGATE_DATA_KEY` into the
  central constant; 745 tests green, zero behavior change) — state.md's backend "+ cleanup" bullet
  was already correct, no remaining-gate wording existed in live config to drop; (2) flipped the open
  [issues.md](../issues.md) 2026-06-01 flicker tracking-item to code-fixed-but-Ψ-owed (expo E1); (3)
  corrected §7 above — the notifications rule uses the `affectedKeys().hasOnly(['seen'])` idiom, NOT
  "mirroring the messages update rule" (the two rules differ by design; see decisions.md Entry B); (4)
  filed the two optional low web hygiene items (W3) into the [issues.md](../issues.md) 2026-06-02
  carry-forward entry. The single remaining gate stays Igor's on-device Ψ verification → then the
  `mobile-stable` flip. The notifications engineer session summaries (backend + review-report, web,
  firestore-rules, expo, plus per-repo audits — 26 files) were archived to `sessions/` and the sources
  deleted from the sibling `.agent/` folders in this same pass.
- **2026-06-02 — FINAL close-out: feature COMPLETE + `mobile-stable` (Docs/QA).** Igor verified all
  four events on real devices + web — favorite, follow, ban/unban, and message (text **and**
  image-only) on iOS, Android, and web — so the held `mobile-stable` flip was released. The
  device-testing round surfaced and fixed a receiver-first gap: the message-ping was built backend-first
  (B4) and the client emitter was missing — added by **expo E3 + web W4** (post-commit fire-and-forget
  ping, `{chatId, messageText}` only); **backend B6 + expo E5** then added the image-only empty-text
  path (backend supplies localized `notif.message.photo.body`). Final lane set: backend B1–B6 + cleanup
  (748 tests); firestore-rules seen-only field-lock (70 tests); web W1–W4 (268 tests); expo E1–E3 + E5
  (403 tests). Applied this pass: status flip to `web-stable` + `mobile-stable` here, in
  [state.md](../state.md) (Notifications block + Expo-backlog row), and the closed/reframed Risk Watch
  rows; the [decisions.md](../decisions.md) 2026-06-02 message-notifications entry gained the emitter +
  empty-text-photo-body contract and a process note; new [issues.md](../issues.md) 2026-06-02 entries
  (emitter-never-built RESOLVED; image-only-no-push RESOLVED; iOS ATT launch-crash RESOLVED — pre-existing
  non-notifications; unused `expo-tracking-transparency` low/open; GA4 DebugView eyeball low/open;
  web/mobile trim divergence low/open) and the flicker tracking-item fully closed (device-confirmed);
  §2.1 / §11 / §13 above reconciled to the as-built emitter + image-only model; the native-review pool
  updated to 33 placeholder rows. Engineer summaries for B6, W4, E3, E5 archived to `sessions/`. Only
  non-feature hygiene remains (preview/prod push credentials, unused-dep removal, DebugView eyeball,
  native translation review) — tracked under Risk Watch; the feature chat can close.
