# Brief 2b — Spec validation: messaging (backend)

**Repo:** oglasino-backend
**Branch:** `dev` (not switched)
**Date:** 2026-05-30
**Mode:** READ-ONLY. No code changes, no DB writes, no git changes.
**Spec validated against:** `oglasino-docs/features/messaging.md`

All 16 claims marked below. Citations are real current `file:line` read from disk; seed-row IDs are read from the SQL files, not copied from the brief.

---

## Claims

### chatId algorithm

**1. `DefaultTrustReviewService` computes chatId as `sorted([uidA, uidB]).join('_')` — MATCHES.**
`src/main/java/.../service/impl/DefaultTrustReviewService.java:114-116`. The method body is the single line at `:115`:
```java
return Stream.of(uid1, uid2).sorted().collect(Collectors.joining("_"));
```
`Stream.sorted()` with no comparator is natural (lexicographic) `String` ordering; the join separator is exactly `"_"`. This matches the spec's `sorted([uid1, uid2]).join('_')` exactly — same sort, same separator. Used at `:59` to build the chatId before reading `chats/{chatId}/messages`. (Spec cited `:114-116`; the method spans those lines, the algorithm is on `:115`.)

### Image token membership checks

**2. Upload-token chat membership — MATCHES.**
Controller `src/main/java/.../images/controller/ImageTokensController.java:41` (`POST /api/secure/images/upload-tokens`, class-level `@RequestMapping("/api/secure/images")` at `:32`). Facade `src/main/java/.../images/facade/impl/DefaultImageTokensFacade.java:95-97`: when `scope == ImageScope.CHAT`, calls `verifyChatMembership(auth.getFirebaseUid(), request.chatId())` BEFORE issuing tokens (`:216-223`), which delegates to `chatService.isUserMemberOfChat(...)`. The check is a single Firestore read of `chats/{chatId}.users` at `DefaultFirebaseChatService.java:139-148` (`users.contains(firebaseUid)`). Server-side, firebaseUid taken from the authenticated principal — not the client body.

**3. View-token chat membership re-verified every request — MATCHES.**
`DefaultImageTokensFacade.java:138-171` (`issueViewToken`). Every call runs `verifyChatMembership(auth.getFirebaseUid(), request.chatId())` at `:151`. No server-side caching of the membership result — each request performs the Firestore read fresh via `isUserMemberOfChat`.

**4. R2 key prefix `private/chats/{chatId}/{uuid}.{ext}` — MATCHES.**
`src/main/java/.../images/path/ImagePaths.java:27` defines `PRIVATE_CHATS = "private/chats/"`. `chatKey(...)` at `:44-46` returns `PRIVATE_CHATS + chatId + "/" + uuid + "." + ext` → `private/chats/{chatId}/{uuid}.{ext}`. Used for `scope=chat` upload keys via `buildKey(...)` at `DefaultImageTokensFacade.java:229`. The view-token `keyPrefix` claim uses `chatPrefix(...)` at `ImagePaths.java:49-51` → `private/chats/{chatId}/` (`DefaultImageTokensFacade.java:160`).

**5. Admin view-token override (admins mint view tokens for arbitrary chats) — DRIFTED.**
Spec §3.6: "Admin viewing of chat images uses the same view-token endpoint; backend additionally permits admins to mint view tokens for arbitrary chats."
Real code: there is **no admin override**. `DefaultImageTokensFacade.issueViewToken` (`:138-171`) unconditionally calls `verifyChatMembership(auth.getFirebaseUid(), ...)` for the caller — no role/admin branch. `isUserMemberOfChat` (`DefaultFirebaseChatService.java:139-160`) has no admin short-circuit. `ImageTokensController` has no admin-specific mapping, and a full sweep of the `admin/` package found no view-token / `signViewToken` / `chatPrefix` usage. An admin who is not a participant of `chats/{chatId}.users` gets a 403 `NOT_CHAT_MEMBER`. The admin-arbitrary-chat view-token path described by the spec does not exist in the backend.

### Admin chat surfaces

**6. Admin endpoints exist, `hasRole('ADMIN')`, read-only — MATCHES (with path correction).**
`src/main/java/.../admin/controller/FirebaseChatController.java`. Class-level `@RequestMapping("/api/secure/admin/chats")` (`:16`) and `@PreAuthorize("hasRole('ADMIN')")` (`:17`). Two `@GetMapping`s:
- `:22` `@GetMapping("/user/{userId}/chats")` → **full mapping `GET /api/secure/admin/chats/user/{userId}/chats`** → `chatService.getChatsForUserPaged(...)`.
- `:32` `@GetMapping("/{chatId}/messages")` → **full mapping `GET /api/secure/admin/chats/{chatId}/messages`** → `chatService.getMessagesForChatPaged(...)`.

Both are GET and read-only (the backing service only performs Firestore `.get()` reads). **Path correction for the mobile audit:** the real mappings carry the `/api` prefix — `/api/secure/admin/chats/...`, NOT `/secure/admin/chats/...`. Note also `userId` is a Postgres `Long` path var, and `{userId}` precision differs from the spec's prose path `GET /user/{userId}/chats` only in the `/api` prefix.

**7. Receiver-per-row fix — MATCHES.**
`src/main/java/.../admin/service/impl/DefaultFirebaseChatService.java`. In `getChatsForUserPaged`, the per-row loop passes the current `doc` to the helper: `:57` `var receiver = getReceiverFromChatDoc(doc, sender.firebaseUid());`. The helper `:162-170` reads `doc.get("users")` (that row's array), filters out the sender, and takes the other participant:
```java
CollectionUtils.emptyIfNull((List<String>) doc.get("users")).stream()
    .filter(uid -> !uid.equals(senderFirebaseUid))
    .findFirst().orElse(null);
```
This is per-doc resolution. The old bug (a `docs.stream().findFirst()` reusing the first doc's receiver for every row) is gone — the only `findFirst()` here operates on the individual doc's two-element `users` array to pick the counterparty.

### Cleanup cron + sentinel

**8. `DefaultMessagingCleanupService` + four config keys — MATCHES.**
Service `src/main/java/.../messaging/service/impl/DefaultMessagingCleanupService.java:35` (`@Service`), cron at `:82` `@Scheduled(cron = "${messaging.cleanup.cron}")`. Config bound via `MessagingCleanupProperties` (`@ConfigurationProperties(prefix = "messaging.cleanup")`). All four keys present in every active profile (`application-{dev,stage,prod}.yaml`); dev block at `application-dev.yaml:259-266`:
```yaml
messaging:
  cleanup:
    cron: "0 0 3 ? * SUN"   # default
    enabled: true            # default
    batch:
      size: 100              # default
    lookback:
      days: 30               # default
```
Defaults also hard-coded in `MessagingCleanupProperties.java` (`enabled=true`, `Batch.size=100`, `Lookback.days=30`). Cron default `"0 0 3 ? * SUN"` matches spec.

**9. Sentinel format — MATCHES. Exact string shape: `deleted:<original-firebase-uid>`.**
`DefaultMessagingCleanupService.java:51` `static final String SENTINEL_PREFIX = "deleted:";` and `:148` `String sentinel = SENTINEL_PREFIX + firebaseUid;`. The produced value is the literal prefix `deleted:` (lowercase, trailing colon, no space) immediately followed by the **full, untruncated** original Firebase UID. This sentinel is written into `chats/{chatId}.users[]` (`:219`) and into message `senderId`/`receiverId` (`:245-249`). **Mobile must treat any value starting with `deleted:` as a deleted user** and render "Deleted user" (`COMMON.user.deleted`). Shape: prefix `deleted:` + UID, e.g. `deleted:AbC123XyZ...`.

**10. `pending_chat_cleanup` + `messaging_cleanup_locks` tables + hard-delete write — MATCHES.**
`src/main/resources/db/migration/V1__init_schema.sql`: `pending_chat_cleanup` table `:2028` (PK `:2039`, `UNIQUE(firebase_uid)` `:2042`); `messaging_cleanup_locks` table `:2056` (PK `:2066`, `UNIQUE(job_name)` `:2069`). `DefaultUserDeletionService.runHardDelete` writes a `pending_chat_cleanup` row at `:404-405`: `pendingChatCleanupRepository.save(new PendingChatCleanup(ctx.firebaseUid, Instant.now()))`.

### `/api/public/notification/test` removal

**11. Endpoint gone — NOT FOUND (correct/expected).**
Searched `src/` for filenames `*NotificationsControllerTest*` and `*NotificationTestDTO*` (none), and grepped `src/` for the mapping string `notification/test` (no matches). Neither `controller/test/NotificationsControllerTest.java` nor `NotificationTestDTO` exists anywhere in the repo. The endpoint is removed as the spec §6.5 intended.

### Translation seeds

**12. `MESSAGES_PAGE.chats.load.more` in all four locales — MATCHES (key present); DRIFTED on the IDs the brief attributes to spec §5.11.**
Key exists in all four files. Real IDs read from disk:

| Locale | ID | Value |
| --- | --- | --- |
| EN | 3391 | `Load more chats...` |
| RS | 5491 | `Učitaj još razgovora...` |
| RU | 7591 | `Zagruzit'' bol''she chatov...` |
| CNR | 1291 | `Učitaj još razgovora...` |

Spec §5.11 (per the brief) states EN=3363, RS=5463, CNR=1263, RU=7563. **Those IDs do not match disk** — real IDs are EN=3391, RS=5491, CNR=1291, RU=7591. The key and its purpose are correct; only the spec's recorded ID values are stale. Flagged in Drift summary.

**13. `messages.send.failed.toast` + `messages.image.preview.fallback.label` in all four locales — MATCHES.**

| Key | EN | RS | RU | CNR |
| --- | --- | --- | --- | --- |
| `messages.send.failed.toast` | 3402 | 5502 | 7602 | 1302 |
| `messages.image.preview.fallback.label` | 3403 | 5503 | 7603 | 1303 |

Both in the `MESSAGES_PAGE` group, all four locale files. (e.g. EN `messages.send.failed.toast` = "Failed to send message. Try again."; EN `messages.image.preview.fallback.label` = "Sent an image".)

**14. Mobile-hardcoded strings — see "Translation key decision" below. All five already have backend keys → REUSE; no new seed required.**

**15. `COMMON.user.deleted` in all four locales — MATCHES.**

| Locale | ID | Value |
| --- | --- | --- |
| EN | 2371 | `Deleted User` |
| RS | 4471 | `Obrisani korisnik` |
| RU | 6571 | `Udalyonnyy polzovatel` |
| CNR | 271 | `Obrisani korisnik` |

Key `user.deleted` in the `COMMON` namespace, all four files. (Minor: EN value is "Deleted User" with a capital U vs spec prose "Deleted user" — label casing, not a defect.)

### Backend chat-data write boundary

**16. Backend never writes `chats/**` / `userchats/**` / `messages` except the cron — MATCHES.**
Swept `src/main/java` for Firestore write verbs (`WriteBatch`, `.batch()`, `.commit()`, `.set/.update/.delete/.add`) and collection references. Write sites into chat data:
- `DefaultMessagingCleanupService.java` — the cleanup cron. Anonymizes `chats` (`:152`, `:221-223`) and `chats/{chatId}/messages` (`:189`, `:196`, `:256-268`), deletes `userchats`/`userblocks` sidecars (`:173-174`, `:282-294`). **This is the allowed Admin SDK path.**

All other Firestore writers touch **non-chat** collections only:
- `DefaultFirestoreUserService.java:38-47` — `WriteBatch` deletes under `notifications/{uid}/userNotifications` (notification cleanup), not chat data.
- `DefaultNotificationsService.java:51+` — writes `notifications` docs (Admin SDK), not chat data.

Read-only chat consumers (no writes): `DefaultFirebaseChatService` (`.get()` only), `DefaultTrustReviewService` (reads `chats/{chatId}/messages`), `DefaultFirebaseStatsService` (aggregate `count()`). No backend write path into `chats/**`, `userchats/**`, or `messages` other than the cleanup cron.

---

## Drift summary (DRIFTED / NOT FOUND only)

- **Claim 5 — DRIFTED (functional).** Spec §3.6 says admins can mint view tokens for arbitrary chats; the backend has no admin override. `issueViewToken` always verifies the caller's own chat membership, and no admin view-token path exists anywhere. A non-participant admin is denied (403 `NOT_CHAT_MEMBER`). If admin chat-image viewing is a required surface, it is not implemented — either the spec is aspirational here or an admin path is missing. Worth a decision.
- **Claim 12 — DRIFTED (documentation).** `MESSAGES_PAGE.chats.load.more` exists in all four locales (functionally correct), but the IDs the brief attributes to spec §5.11 are stale: spec says EN=3363/RS=5463/CNR=1263/RU=7563; disk has EN=3391/RS=5491/CNR=1291/RU=7591. Spec §5.11 ID values should be corrected.
- **Claim 11 — NOT FOUND (expected/correct).** `NotificationsControllerTest.java`, `NotificationTestDTO`, and any `notification/test` mapping are absent. This is the intended result of spec §6.5, not a defect.

All other claims (1, 2, 3, 4, 6, 7, 8, 9, 10, 13, 15, 16) MATCH.

---

## Translation key decision (claim 14)

For each of the five strings mobile currently hardcodes in Serbian, a backend-seeded `MESSAGES_PAGE` key **already exists in all four locales**. Mobile can consume these directly. **No new seed rows are required for any of the five** → no backend translation-seed brief is needed for this set.

| # | Mobile-hardcoded (SR) | Decision | Key | IDs (EN / RS / RU / CNR) |
| --- | --- | --- | --- | --- |
| A | "Pretraži ćaskanja..." (search placeholder) | **REUSE EXISTING KEY** | `MESSAGES_PAGE.search.placeholder` | 3382 / 5482 / 7582 / 1282 |
| B | "Izaberite ćaskanje kako bi videli poruke..." (select chat) | **REUSE EXISTING KEY** | `MESSAGES_PAGE.messages.select.placeholder` | 3384 / 5484 / 7584 / 1284 |
| C | "Učitaj starije poruke" (load older messages) | **REUSE EXISTING KEY** (wording note ↓) | `MESSAGES_PAGE.messages.load.more` | 3392 / 5492 / 7592 / 1292 |
| D | "Ne možete poslati poruku ... jer ste blokirali ..." (you blocked them) | **REUSE EXISTING KEY** | `MESSAGES_PAGE.blocking.label` | 3393 / 5493 / 7593 / 1293 |
| E | "Ne možete poslati poruku ... jer vas je blokirao ..." (they blocked you) | **REUSE EXISTING KEY** | `MESSAGES_PAGE.blocked.by.label` | 3394 / 5494 / 7594 / 1294 |

Notes for Igor:
- **A, B, D, E** are exact functional/semantic matches — RS values mirror the mobile hardcodes (e.g. `search.placeholder` RS = "Pretraži ćaskanja..."; `messages.select.placeholder` RS = "Izaberite ćaskanje kako bi videli poruke..."; `blocking.label`/`blocked.by.label` are the you-blocked / blocked-by notices). Straight reuse.
- **C — wording difference only, your call.** `messages.load.more` is the existing "load more messages" affordance (the web Messages.tsx `loadMoreMessages` key). RS value is "Učitajte još poruka..." vs the mobile hardcode "Učitaj starije poruke" — same meaning, slightly different phrasing. Reusable as-is; if exact mobile wording must be preserved, that's a copy-edit of the existing row, not a new key. Either way **no new seed row is needed**.
- The earlier-considered fallback `chats.load.more` (claim 12) is for the chat-LIST load-more, distinct from the messages load-more in C. Don't confuse them.

**Bottom line:** mobile messaging adoption needs **zero new backend translation seeds** for these five strings. The only open copy question is whether C's existing wording is acceptable.

---

## Cleanup performed

- none needed — read-only validation, no files changed.

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change.
- `issues.md`: no change. (Two drift findings below are surfaced for Mastermind; whether they warrant an `issues.md` entry or a spec edit is Mastermind/Docs-QA's call, not mine to write.)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — read-only, no code touched.
- Part 4a (simplicity) / Part 4b (adjacent observations): see "For Mastermind".
- Part 6 (translations): N/A this session (validation only, no seeds added).
- Other parts touched: Part 11 (trust boundaries) — confirmed image-token membership and chatId are server-derived; Part 7 (error contract) — observed `NOT_CHAT_MEMBER` 403 path, no change.

## Known gaps / TODOs

- none. All 16 claims marked.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only session, no code added.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Claim 5 (DRIFTED, medium):** Spec §3.6 promises an admin view-token override for arbitrary chats; the backend does not implement it (`DefaultImageTokensFacade.issueViewToken` always verifies the caller's own membership; no admin branch anywhere). `images/facade/impl/DefaultImageTokensFacade.java:138-171`. A non-participant admin cannot view chat images. Decide: spec is aspirational (correct it) or an admin path is genuinely missing (new brief). I did not change anything — out of scope, read-only.
- **Claim 12 (DRIFTED, low):** Spec §5.11 records stale IDs for `chats.load.more` (EN=3363/RS=5463/CNR=1263/RU=7563); disk has EN=3391/RS=5491/CNR=1291/RU=7591. Suggest a spec ID correction by Docs/QA. Low severity — purely a documentation value.
- **Claim 14 result (the actionable output):** all five mobile-hardcoded strings already have backend keys (table above). **No backend translation-seed brief is required.** The only residual is a copy decision on string C (`messages.load.more` wording "Učitajte još poruka..." vs "Učitaj starije poruke").
- Branch confirmed `dev`; not switched. No prior `.agent/audit-messaging.md` exists in this repo (this brief is self-contained).
