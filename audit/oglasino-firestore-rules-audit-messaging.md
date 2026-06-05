# Session summary

**Repo:** oglasino-firestore-rules
**Branch:** stage (read-only; no changes to source files)
**Date:** 2026-05-20
**Task:** Read-only audit of Firestore Security Rules and test scaffolding for the messaging surface (chats, messages, userchats, userblocks/Reverse, notifications, users). Output feeds a Mastermind chat planning the messaging feature. Per the brief: rules file is ground truth; do not consult any prior spec or design doc.

## Implemented

The audit is structured as the seven sections specified by the brief. All file references are line-anchored to `firestore.rules` (72 lines, single source of rules, last modified by commit `9248c9c` on 2026-05-10; not edited since).

### 1. Helper functions

There is **exactly one** top-level `function` declaration in the rules file.

#### `isBlocked(userA, userB)` — `firestore.rules:4-7`

```
function isBlocked(userA, userB) {
  return exists(/databases/$(database)/documents/userblocks/$(userA)/blocked/$(userB))
      || exists(/databases/$(database)/documents/userblocks/$(userB)/blocked/$(userA));
}
```

- **What it does:** returns `true` if either party has blocked the other, by probing both the forward index (`userblocks/A/blocked/B`) and the symmetric forward index from the opposite direction (`userblocks/B/blocked/A`). Despite the existence of a separate `userblocksReverse` collection (see §2), this helper does NOT use it — both probes go through the canonical `userblocks` path.
- **Where called from:** one call site only — the `messages` `create` rule at `firestore.rules:54-57`. Not used by `read`, `update`, or any other collection.
- **Correctness:**
  - Uses `exists()`, not `get()` or `getAfter()`. `exists()` returns a plain bool and does not throw when the document is missing — this is the right primitive for a "has the user opted in?" check.
  - **Cost:** each call to `isBlocked` triggers two `exists()` lookups, which count against the rule-evaluation read budget (10 doc accesses per single-doc request). With the one `get(/chats/{chatId})` already in the message-create rule (§2.6), each message-create costs 3 rule-evaluation document reads. Within budget, no current concern.
  - Helper is scoped at `match /databases/{database}/documents` level, so it is callable from any nested match block, but only `messages` `create` invokes it. Block-aware semantics therefore do **not** apply to message `read`, message `update`, chat `create`, chat `read`, or userchats writes.

### 2. Rule-by-rule inventory

Source order, each `match` block in the file. For each `allow` clause: predicate, the human-language summary, the trust-boundary classification, and concerns.

Trust-boundary tags (per Part 11):
- **client-trusted** — value is read from `request.resource.data.*` (whatever the client sent) and the rule uses it directly without cross-validation.
- **auth-derived** — value comes from `request.auth.*` (Firebase Auth token; backed by Firebase Auth's signing).
- **server-read** — value is read from another doc on the server via `get()` / `getAfter()` / `exists()`.
- **path-bound** — value is a URL path parameter (e.g. `{userId}`), which the client chose but is fixed by the request URL.

---

#### 2.1 `match /users/{userId}` — `firestore.rules:9-12`

```
match /users/{userId} {
  allow read: if true;
  allow write: if request.auth != null && request.auth.uid == userId;
}
```

- **`read`** — predicate: `true`. Permits anyone (including unauthenticated) to read any user doc. Forbids: nothing.
  - Trust check: no inputs trusted; rule has no inputs. **No auth check at all.**
- **`write`** — predicate: `request.auth != null && request.auth.uid == userId`. Permits the owner to create, update, or delete their own user doc. Forbids any cross-user write.
  - Trust check: `userId` is `path-bound`; `request.auth.uid` is `auth-derived`. No field-level constraints on what the owner may write to their own doc.
- **Notable concerns:**
  - `allow read: if true` is intentionally permissive (public profile read), but the rule is dependent on the assumption that `users/{userId}` documents contain only fields that are safe to expose to unauthenticated callers. **The actual field set in `users/{userId}` is the central question of §5 below**, and the audit cannot fully answer it without console access. If those docs contain email/phone/address/etc., this rule leaks PII to anyone with a uid.
  - `write` is the single combined verb — it permits `create`, `update`, AND `delete` by the owner. No field-shape, no immutability of `uid`, no rate limit, no schema check. The owner could overwrite their entire profile with arbitrary fields.

---

#### 2.2 `match /userchats/{userId}` — `firestore.rules:14-15`

```
allow create, read, write: if request.auth != null && request.auth.uid == userId;
```

- Permits: the owner (and only the owner) to create / read / write their per-user chats container doc. Forbids: any cross-user access.
- Trust check: `userId` is `path-bound`; `request.auth.uid` is `auth-derived`.
- Notable concerns:
  - `write` already encompasses `create` and `update` — listing `create, read, write` is redundant but not broken. Pattern repeats from the Firebase Console's verbose export style.
  - No `list` verb is declared; the implicit `read` covers list. A list query that includes a `where(documentId, ...)` filter pinning to one's own uid would pass, but a naked `collection('userchats').get()` from any signed-in caller would be denied per-doc unless the caller is each `userId`. In practice, listing the entire `userchats` top-level collection is not a meaningful client query.

---

#### 2.3 `match /userchats/{userId}/chats/{chatId}` — `firestore.rules:16-19`

```
allow create, read, update, delete: if request.auth != null &&
  (request.auth.uid == userId || request.auth.uid == request.resource.data.withUserFirebaseUid);
```

- Permits: the owner of the userchats container, OR a third party who declares themselves to be `withUserFirebaseUid` on the doc they're writing.
- Trust check: `userId` is `path-bound`; `request.auth.uid` is `auth-derived`; `request.resource.data.withUserFirebaseUid` is **client-trusted**. The rule asks the writer to assert "I am the counterparty for this sidecar entry" — and accepts that assertion as long as the asserted uid matches the writer's own.
- Notable concerns:
  - **`request.resource` is undefined for `read` and `delete` operations.** So the second clause (`request.auth.uid == request.resource.data.withUserFirebaseUid`) can only meaningfully gate `create` and `update`. On `read` and `delete`, the rule effectively collapses to "owner only" — which is fine but the rule shape is misleading.
  - **The second clause permits a non-owner to write into another user's sidecar collection.** If user B writes `userchats/A/chats/{chatId}` with `withUserFirebaseUid: <B's uid>` in the payload, the rule passes. This is presumably intentional for the "start a chat" flow (the initiator mirrors a sidecar entry on the counterparty's side so the counterparty's chat list updates), but it is a deliberate-trust decision worth Mastermind's attention: B can create / update / delete arbitrary sidecar entries on A's side as long as B identifies themselves as the counterparty. No field-shape constraints either.
  - No `getAfter()` to verify the corresponding `/chats/{chatId}` doc actually has both A and B in its `users` array. The sidecar is decoupled from the chat doc at the rule layer.
  - Field-name asymmetry with `/chats/{chatId}` (which uses `users: [a, b]` array — see §2.5). The sidecar uses `withUserFirebaseUid` (singular string). Clients must produce both shapes correctly.

---

#### 2.4 `match /userblocks/{userId}/blocked/{blockedId}` — `firestore.rules:22-29`

```
allow read:   if request.auth != null && request.auth.uid == userId;
allow create: if request.auth != null && request.auth.uid == userId && blockedId != userId;
allow delete: if request.auth != null && request.auth.uid == userId;
```

- Permits: the owner of the block list to read / create / delete their own block entries. Self-block is forbidden.
- Forbids: cross-user reads or writes of someone else's block list; updates of existing block entries (no `update` verb declared).
- Trust check: all inputs are `path-bound` or `auth-derived`. No client-trusted fields.
- Notable concerns:
  - No `update` is intentional and correct — a block entry has no mutable state worth changing; the document key encodes who is blocked.
  - No field-shape constraints on the block document body. Clients can write arbitrary fields into `userblocks/{userId}/blocked/{blockedId}`, but since `isBlocked()` checks only `exists()` (not field contents), the body is semantically meaningless to the rules. Backend may put metadata (timestamp, reason) here; the rule does not validate it.

---

#### 2.5 `match /userblocksReverse/{userId}/blockedBy/{blockerId}` — `firestore.rules:31-38`

```
allow read:   if request.auth != null && request.auth.uid == userId;
allow create: if request.auth != null && request.auth.uid == blockerId;
allow delete: if request.auth != null && request.auth.uid == blockerId;
```

- Permits: the blocked user to read their "who has blocked me" index; the blocker to create or delete entries that name themselves as the blocker, in the blocked user's reverse-index path.
- Forbids: any update; the blocker reading the index (they can read the forward index instead); the blocked user mutating the index.
- Trust check: `userId`, `blockerId` are `path-bound`; `request.auth.uid` is `auth-derived`.
- Notable concerns:
  - **Reverse index is not consulted by `isBlocked()`** — see §1. The reverse index exists for "I want to know who has blocked me" query support (useful for client-side filtering of incoming messages, hiding users from search, etc.). If no client / backend code actually reads `userblocksReverse`, the reverse index is dead weight maintained by the blocker on every block/unblock. The audit cannot say whether clients use this index; that is a cross-repo seam question.
  - The blocker must write to **two** paths atomically when blocking (forward + reverse) to keep the index consistent. The rules do not enforce this dual-write — a blocker can write only the forward entry and skip the reverse, and `isBlocked()` will still work, but `userblocksReverse` will silently be incomplete. **Adjacent observation flagged** in §7.

---

#### 2.6 `match /chats/{chatId}` — `firestore.rules:40-47`

```
allow read: if request.auth != null &&
  (resource == null || request.auth.uid in resource.data.users);
allow create: if request.auth != null
  && request.resource.data.users.size() == 2
  && request.auth.uid in request.resource.data.users;
allow update, delete: if request.auth != null
  && request.auth.uid in resource.data.users;
```

- **`read`** — permits any authed user to read a non-existent chat doc (`resource == null` short-circuit), and permits chat participants to read existing chat docs. Forbids unauthenticated reads.
  - Trust check: `resource.data.users` is `server-read` (from the persisted doc); `request.auth.uid` is `auth-derived`.
  - Notable: the `resource == null` short-circuit lets any signed-in user probe arbitrary `chatId`s and observe existence/non-existence. Chat IDs are presumably random / derived from sorted UIDs, so the exposure is limited but real. More importantly: this rule supports the "subscribe before the doc exists" pattern (snapshot listener mounted before chat creation completes — won't emit `permission-denied` for the parent doc). The messages subcollection `read` rule does NOT have an equivalent fallback (see §2.7) — that is the principal asymmetry in the rules file.
- **`create`** — permits any authed user to create a chat doc with `users: [string, string]` of size exactly 2 including their own uid. Forbids any other shape.
  - Trust check: `request.resource.data.users` is **client-trusted**. The rule enforces `size == 2` and "caller is in the array," but does NOT verify the OTHER party consents, exists, or is not blocked by the caller. No field-shape on any other field (clients can attach `lastMessage`, `lastUpdated`, `productId`, etc., freely).
  - **No cross-check against `isBlocked()` here** — `isBlocked` is invoked only on `messages` `create`. So a blocked user can still create a chat document naming their target as the counterparty; the target will only experience block enforcement once the first message tries to send.
- **`update`** / **`delete`** — permits any participant (in `resource.data.users`) to mutate or delete the entire chat doc.
  - Trust check: `resource.data.users` is `server-read`.
  - **Field-immutability gap:** the update rule does not freeze `users`. A participant can rewrite the `users` array — add a third party, remove the other party, swap themselves out. The create rule's `size() == 2` constraint is not echoed in update. **Adjacent observation flagged** in §7.
  - **Cascade gap:** delete removes the chat doc but does NOT touch the `messages` subcollection. The messages remain in Firestore as orphaned docs whose `read` rule (§2.7) will return `permission-denied` because `get(/chats/{chatId})` returns null. Backend cleanup or a `delete: if false` posture would be safer. **Adjacent observation flagged**.

---

#### 2.7 `match /chats/{chatId}/messages/{messageId}` — `firestore.rules:48-62`

```
allow read: if request.auth != null &&
  request.auth.uid in get(/databases/$(database)/documents/chats/$(chatId)).data.users;
allow create: if request.auth != null
  && request.resource.data.senderId == request.auth.uid
  && request.auth.uid in get(/databases/$(database)/documents/chats/$(chatId)).data.users
  && !isBlocked(
      request.resource.data.senderId,
      request.resource.data.receiverId
  );
allow update: if request.auth != null
  && request.auth.uid in get(/databases/$(database)/documents/chats/$(chatId)).data.users
  && request.resource.data.seen is bool;
allow delete: if false;
```

- **`read`** — permits any participant of the parent chat to read messages. Forbids non-participants and unauthenticated reads.
  - Trust check: `get(/chats/{chatId}).data.users` is `server-read`. `request.auth.uid` is `auth-derived`.
  - **Critical failure mode:** there is **no `resource == null` fallback and no null-guard on the parent-chat `get()`.** If the parent chat doc does not exist at rule-evaluation time, `get(...)` returns null, dereferencing `.data.users` raises a rule-evaluation error, and the rule denies. A snapshot listener that mounts on `/chats/{chatId}/messages` before the chat doc is created will detach with `permission-denied`. This is the asymmetry with §2.6's chat `read` rule, which DOES guard `resource == null`.
- **`create`** — permits any participant (per parent chat's `users`) to insert a message whose `senderId` matches the writer's uid and where neither party has blocked the other.
  - Trust check: `request.resource.data.senderId` is **client-trusted but constrained** (must equal `request.auth.uid`); `request.resource.data.receiverId` is **client-trusted with no constraint** — the rule does NOT verify `receiverId` is the OTHER element of `chat.users`. A sender can write a message with an arbitrary `receiverId` (not necessarily their chat counterparty), and `isBlocked()` is evaluated against that arbitrary uid. If `receiverId` is set to a uid neither party in the chat, the block check is meaningless and the message still lands.
  - `get(/chats/{chatId}).data.users` is `server-read`. **Same null-deref failure mode** as `read` if parent chat doc is missing.
  - **`get()` (not `getAfter()`) is used.** This is the load-bearing decision for atomic chat-creation flows. A batched/transactional write of `{create chat, create first message}` will fail because the message rule's `get()` sees the pre-batch snapshot (chat does not yet exist). The only client pattern that works against these rules today is: sequential — create chat, await commit ack, then create message.
  - **No content-shape constraints** on `text` / body / timestamps / mime / attachments. Client can write any extra fields.
- **`update`** — permits any participant to update an existing message provided the post-update `seen` field is of type bool.
  - Trust check: `request.resource.data.seen` is **client-trusted** (only its type is checked). All other fields including `senderId`, `receiverId`, `text` are **unconstrained** — a participant can rewrite the message content arbitrarily as long as they leave a bool in `seen`.
  - **Field-immutability gap:** the rule does not preserve `senderId` (a participant could rewrite who appears to have sent a message), `receiverId`, or message body. **Adjacent observation flagged** in §7.
  - **Authorization gap:** the rule lets the SENDER flip `seen`, not just the receiver. If "seen" is meant as a read-receipt, only the receiver should flip it. **Adjacent observation flagged**.
  - Same null-deref failure mode on parent-chat `get()`.
- **`delete`** — hard `false`. Messages are immutable on deletion. Sender, receiver, admin — nobody can delete.

---

#### 2.8 `match /notifications/{userId}/userNotifications/{notificationId}` — `firestore.rules:65-70`

```
allow read, update, delete: if request.auth != null
  && request.auth.uid == userId;
allow create: if request.auth == null
  || request.auth.token.admin == true;
```

- **`read` / `update` / `delete`** — permits the recipient (only) to read, modify, or delete their notifications. Forbids cross-user access and unauthenticated access. Correct.
- **`create`** — **CRITICAL trust-boundary violation.** The clause `request.auth == null || request.auth.token.admin == true` reads "anonymous OR admin-claim holder" — meaning **any unauthenticated client can create notification documents for any user under any document id**. This is almost certainly a transcription bug. The intent was almost certainly `request.auth != null && request.auth.token.admin == true` (admin only) or simply `request.auth.token.admin == true` (which implicitly requires auth non-null because referencing `request.auth.token` on a null auth raises an error, denying the rule).
  - Trust check: `request.auth` checked but inverted; `request.auth.token.admin` is `auth-derived` (custom claim). Custom claims are signed by Firebase Auth and trustworthy when the writer is authenticated; the `null` branch bypasses this entirely.
  - **Severity: high.** Outside the messaging defect's blast radius but the most serious thing in the rules file. Notification spam, impersonation, and notification-as-tracker abuse are all unlocked. Repeated from yesterday's audit; still present today. **Flagged loudly** in §7.

---

### 3. Test scaffolding

- **Directory:** there is no `tests/` directory in the repo. Verified by `find ... -type d` and `find ... -name "*.test.*"` — both return nothing. The `vitest.config.ts` `include: ['tests/**/*.test.ts']` glob therefore matches zero files.
- **Framework configured:** **vitest** (`vitest@^2.1.8` in devDependencies). Config is `vitest.config.ts`: node environment, `testTimeout: 15000`, `hookTimeout: 30000`, `fileParallelism: false` (sequential — appropriate for an emulator-backed suite where parallel test files would race on Firestore state).
- **`@firebase/rules-unit-testing` version:** `^4.0.1` (installed; no test code consumes it yet).
- **Existing tests:** **zero.** No file in the repo invokes `describe`, `it`, `test`, `assertSucceeds`, `assertFails`, or `initializeTestEnvironment`.
- **Does `npm test` actually run anything?** `npm test` → `vitest run --passWithNoTests`. The `--passWithNoTests` flag means vitest exits 0 when no test files match. So `npm test` runs successfully, produces no assertions, and gates nothing. Confirmed by the test script in `package.json:8`.
- **Emulator wiring:** `npm run emulator` → `firebase emulators:start --only firestore --project demo-oglasino`. The `demo-` prefix is a Firebase convention signaling an emulator-only project that the CLI will run without authentication. The emulator config in `firebase.json` binds Firestore to `127.0.0.1:8080` and the UI to `:4000`. There is also `npm run test:watch` → `firebase emulators:exec --only firestore --project demo-oglasino "vitest"`, which starts the emulator and runs vitest in watch mode against it. **Wiring is correct and would work**, but with zero test files it doesn't do anything yet.
- **What `validate-pr.yml` gates on:** PRs targeting `stage` or `main` trigger:
  1. `npm ci`
  2. `npm test` (passes trivially with no tests)
  3. Auth to Firebase via `FIREBASE_SA_STAGE` service-account JSON
  4. `npx firebase deploy --only firestore --project stage --dry-run --non-interactive`
  The dry-run is the only meaningful gate today — it catches rules **syntax** errors. It does NOT catch semantic / behavioral regressions. **A rule change that compiles but allows or denies the wrong thing would pass the PR gate.**

---

### 4. Deploy surface

- **Firebase projects:**
  - Stage: `oglasino-stage-49abb` (aliased `stage` in `.firebaserc`).
  - Prod: `oglasino-prod-7e5db` (aliased `prod` in `.firebaserc`).
  - There is intentionally **no `default` alias** — every firebase command must specify `--project stage` or `--project prod` explicitly.
- **Workflows present:**
  - `.github/workflows/deploy-stage.yml` — deploys to stage.
  - `.github/workflows/deploy-main.yml` — deploys to prod. (Filename uses the git branch name `main`, not the firebase alias `prod`.)
  - `.github/workflows/validate-pr.yml` — PR gate. See §3.
- **Triggers:**
  - `deploy-stage.yml`: `on: push: branches: [stage]` **AND** `workflow_dispatch: {}` (push-triggered + manual). Concurrency group `deploy-firestore-stage`, `cancel-in-progress: false` (later pushes queue rather than preempt).
  - `deploy-main.yml`: `on: push: branches: [main]` **AND** `workflow_dispatch: {}` (push-triggered + manual). Concurrency group `deploy-firestore-main`, `cancel-in-progress: false`.
- **What gets deployed:** `npx firebase deploy --only firestore --project <env> --non-interactive`. This deploys both `firestore.rules` AND `firestore.indexes.json` (because firebase.json declares both under the `firestore` target, and `--only firestore` includes both rules and indexes).
- **Auth strategy:** service-account JSON via repo secrets (`FIREBASE_SA_STAGE` / `FIREBASE_SA_PROD`), written to disk at runtime, pointed to by `GOOGLE_APPLICATION_CREDENTIALS`. No `firebase login:ci` token.
- **Deploy-drift question (carryover):** the rules file in this repo has been edited exactly once (commit `9248c9c` on 2026-05-10). Both `origin/stage` and `origin/main` are at `ee8e54f` (the index-only commit, 2026-05-10) — meaning the rules content was pushed to both branches on 2026-05-10, triggering both deploy workflows. **Assuming nobody has edited rules in the Firebase Console out-of-band since 2026-05-10**, the live rules on stage and prod are identical to `firestore.rules` in this repo. The audit cannot inspect the live rules from this environment (would require `firebase firestore:rules:get` with an authenticated session). **Recommended cross-check:** Igor opens the Firebase Console for `oglasino-stage-49abb`, Firestore → Rules, and verifies the "Last published" timestamp matches 2026-05-10. If later, drift exists.

---

### 5. The `users/{userId}` collection question

#### Rule (verbatim)

`firestore.rules:9-12`:

```
match /users/{userId} {
  allow read: if true;
  allow write: if request.auth != null && request.auth.uid == userId;
}
```

- **`read`: `if true`** — open to anyone, including unauthenticated callers. Any client (web, mobile, scraper, anonymous browser) can fetch any user document by uid.
- **`write`: owner only** — only the user themselves can mutate their own doc. No field-shape, no immutability of `uid`, no schema, no audit fields preserved.

#### What fields are actually stored in `users/{userId}` on the stage Firebase project

**Cannot answer from this agent environment.** The brief explicitly anticipated this case and instructed: "If you cannot access the Firebase console from the agent environment, say so explicitly and note that this must be resolved separately. Do not guess."

Why I cannot:

- I have no browser access — I cannot open the Firebase Console at `console.firebase.google.com/project/oglasino-stage-49abb/firestore/data/~2Fusers`.
- The `firebase` CLI on this machine could in principle query Firestore via the Firebase Admin SDK or `firestore:export`, but those calls require active Google credentials and would issue authenticated reads against the stage project's live data. That is outside the read-only-audit posture of this brief (the brief authorizes inspection of the rules file and the test scaffolding, not authenticated reads against live Firestore data).
- I could also try to use `gcloud firestore export` or call the REST API directly with a service account, but I do not have stage-project service-account credentials staged in this environment, and minting them would (a) require Igor's cooperation and (b) constitute a privileged action better done deliberately by Igor.

**This must be resolved separately.** Two options for Igor / Mastermind:

1. **Console sample (recommended for speed):** Igor (or the docs/QA agent with console access) opens the stage Firebase Console → Firestore → `users` collection, expands 3–5 documents picked at random (or the most recent), and lists every distinct field name observed. Note which look like PII (email, phoneNumber, address, dateOfBirth, etc.) vs public-profile (displayName, photoURL/imageKey, createdAt, etc.).
2. **CLI export (more thorough but heavier):** authenticated `gcloud firestore export` to GCS, downloaded and inspected — overkill for a field-name inventory but the right tool if the question grows into "what's the full schema and value distribution."

#### Why this question matters now

The chat-creation rule (`request.auth.uid in request.resource.data.users`) does NOT depend on the `/users/{userId}` doc — but several plausible *future* messaging features would:

- Per-user messaging preferences (mute, do-not-disturb, blocked-strangers-only) stored on `users/{userId}` would need to be read by message-create rules.
- Per-user public-profile snapshots embedded into chat docs (denormalized `displayName`, `photoURL`) for offline chat-list rendering need a stable, public `users` shape.
- Account-deletion / GDPR flows need to know every collection that references a uid; `users/{userId}` is one of them.

Mastermind cannot scope the messaging feature confidently without knowing whether `users/{userId}` is a thin public-profile doc (safe to leave `allow read: if true`) or a fat auth/PII doc (in which case the current rule is a critical leak that must be fixed before the messaging surface lands).

---

### 6. Cross-repo seams

Each entry: the rule's assumption, what we'd need from the counterpart audit to verify it.

1. **`/chats/{chatId}` create — `users` field must be `[string, string]` of size exactly 2 including the caller's uid.**
   - Counterpart: web client (and mobile client, when it lands) chat-creation payload code.
   - To verify: read the exact request payload the web client constructs for chat creation. Field name must be `users` (not `participants`, not `withUser`); type must be an array of two Firebase UID strings (not objects, not a map). Note: the SIDECAR at `/userchats/{userId}/chats/{chatId}` uses a DIFFERENT field name (`withUserFirebaseUid`, singular). A field-name mix-up between the two paths is a plausible bug source.

2. **`/chats/{chatId}/messages/{messageId}` create — chat doc must exist on the server before the first message can be created.**
   - The message-create rule uses `get()` (not `getAfter()`). Therefore a batched/transactional write of `{create chat, create first message}` will fail (the message rule's `get()` sees the pre-batch snapshot). The only client pattern that works today is sequential: write the chat doc, await commit acknowledgment, then write the first message.
   - Counterpart: web client's "start a new conversation from a product page" flow.
   - To verify: read the client code that handles "first message in a new chat." Does it use `writeBatch`/`runTransaction`? Or does it `await setDoc(chatRef, ...)` and then issue the message write? If the former, this is the root cause of any "first-message fails" defect today. If we want to allow atomic creation, the rules must change to `getAfter()` (out of scope here; in scope for any fix spec).

3. **`/chats/{chatId}/messages/{messageId}` create — `senderId` must equal `request.auth.uid`; `receiverId` is unconstrained at the rule layer.**
   - Counterpart: web/mobile client message-write payload.
   - To verify: the client always stamps `senderId: currentUser.uid` and stamps `receiverId` correctly as the OTHER user in the chat (not, e.g., the message recipient's display name, or the chat ID, or undefined). The rules trust `receiverId` blindly except as input to `isBlocked()`, so a wrong `receiverId` will not cause a permission-denied — it will silently break the block check.

4. **`/chats/{chatId}/messages/{messageId}` create — block check uses `isBlocked(senderId, receiverId)` against `/userblocks/`.**
   - Counterpart: backend or web client that writes block entries when a user blocks another.
   - To verify: clients write to `/userblocks/{blockerUid}/blocked/{blockedUid}` (forward index) when blocking. Optionally also write to `/userblocksReverse/{blockedUid}/blockedBy/{blockerUid}` (the rules permit it but `isBlocked` does not read it).

5. **`/chats/{chatId}/messages/{messageId}` update — only the `seen is bool` check; everything else is unconstrained.**
   - Counterpart: web/mobile client that flips "seen" / "read" state on incoming messages.
   - To verify: the client only writes the `seen` field and never patches other fields (otherwise it can rewrite content). Also: the client correctly handles the fact that the rule does NOT distinguish sender from receiver — if the client UI assumes only the receiver flips `seen`, that's a client-side convention, not a rule guarantee.

6. **`/userchats/{userId}/chats/{chatId}` — counterparty may write into the owner's sidecar by declaring themselves `withUserFirebaseUid`.**
   - Counterpart: web client's "start a new chat" flow — does it mirror the sidecar on the counterparty's side?
   - To verify: read the chat-creation flow end-to-end. If the client writes `userchats/A/chats/{chatId}` (own sidecar) AND `userchats/B/chats/{chatId}` (counterparty sidecar, payload includes `withUserFirebaseUid: A`), this confirms the rule's permissive design is intentional. If only one side is written, the rule's design is over-permissive (the counterparty branch is unused) and could be tightened.

7. **`/notifications/{userId}/userNotifications/{notificationId}` create — currently allows unauthenticated callers.**
   - Counterpart: backend repo (whichever service writes notifications).
   - To verify: does the backend mint a Firebase Auth custom claim `admin: true` on its service identity and use that to write notifications? If yes, the rule's `|| request.auth.token.admin == true` branch is the intended path and the `request.auth == null` branch is a bug. If the backend writes notifications via the Firebase Admin SDK (which bypasses security rules entirely), the entire `create` rule predicate is dead and can be tightened to `if false` without breaking anything.

8. **Backend identity model — custom claim `admin` on Firebase Auth tokens.**
   - Counterpart: backend repo (Firebase Admin SDK setup).
   - To verify: the backend mints `admin: true` custom claims on whatever Firebase Auth identity it uses to make Firestore calls (typically a dedicated service identity, NOT real user accounts). Clients can never set custom claims; if the backend uses the Admin SDK directly (no Auth token), rule evaluation is bypassed and the claim is moot — but then any rule that checks `request.auth.token.admin` is dead code. Worth confirming so we know which rules are load-bearing.

9. **Field-name convention across the messaging surface.**
   - Counterpart: web client, mobile client, backend (any code that constructs chat or sidecar payloads).
   - To verify: clients consistently use `users` (array) on chat docs and `withUserFirebaseUid` (singular) on sidecar docs. The asymmetry is a known foot-gun for any refactor.

10. **Public-profile data on `/users/{userId}`.**
    - Counterpart: any code (web, mobile, backend, signup flow) that writes to `/users/{userId}`.
    - To verify: what fields are actually written. See §5 — this is the blocker question for whether `allow read: if true` is safe.

---

### 7. Adjacent observations

Items surfaced by the audit that are out of scope for this brief but worth Mastermind's attention. Repeated from yesterday's audit where still present; **new** marker on anything not flagged 2026-05-19.

#### Severity: high

1. **`notifications` create rule permits unauthenticated callers.** `firestore.rules:68-69`. `allow create: if request.auth == null || request.auth.token.admin == true`. Any anonymous client can create notification documents under any user's path with any id and any body. Almost certainly a transcription bug. Suggested fix (out of scope here): `allow create: if request.auth != null && request.auth.token.admin == true`. **Repeated from 2026-05-19 audit — still present.**

#### Severity: medium

2. **`/users/{userId}` reads are open to unauthenticated callers and the actual stored fields are unknown to the audit.** §5 above. If `users/{userId}` carries any PII (email, phone, address, etc.), this rule leaks it. Cannot resolve without console access. The fix space is small: tighten read to `request.auth != null` (still public to all signed-in users, blocks scrapers) or to `request.auth != null && fields-restricted` (requires moving PII to a sibling subcollection with its own rule). **The right fix depends on §5's answer.**

3. **`/chats/{chatId}` update rule has no field-immutability constraints.** Any participant can rewrite the `users` array — add a third party, remove the other party. The create rule's `size() == 2 && caller in users` constraint is not preserved on update. Suggested fix: add `&& request.resource.data.users == resource.data.users` (or equivalent).

4. **`/chats/{chatId}/messages/{messageId}` update permits anyone in the chat to flip `seen`, including the sender on their own outgoing message.** If `seen` is a read-receipt, only the receiver should flip it. Suggested fix: `&& request.auth.uid == resource.data.receiverId` on the update rule.

5. **`/chats/{chatId}/messages/{messageId}` update permits arbitrary rewrite of `senderId`, `receiverId`, and message body.** The rule only requires `seen is bool` post-update; everything else is unconstrained. A participant can rewrite who appears to have sent a message and the message content. Suggested fix: add explicit immutability for `senderId`, `receiverId`, and `text`/`body`/`createdAt` fields.

6. **`isBlocked()` does not consult `userblocksReverse`; the reverse index is maintained by clients but only useful if clients/backend read it explicitly.** If nobody reads it, it's dead weight. **New observation** — yesterday's audit noted the reverse index pattern but didn't surface this dead-weight question. Worth Mastermind's call: keep + document the use case, or drop the reverse-index rules and trust forward-only lookups.

7. **No tests at all.** `tests/` directory does not exist; `vitest run --passWithNoTests` passes trivially; `validate-pr.yml`'s only meaningful gate is rules-syntax dry-run. A rule change that compiles but allows or denies the wrong thing would pass the PR gate. Known Phase B item per README, but worth confirming as a hardening priority before any rule change ships. **Repeated from 2026-05-19.**

8. **No `getAfter()` anywhere in the rules file.** `get()` is used three times (messages read / create / update, all against the parent chat doc). For any future feature that wants atomic batched creation (e.g. "create chat + first message in one transaction" — which clients today must serialize because of `get()`), this is the load-bearing decision. **Repeated from 2026-05-19 as part of the cross-document-lookup hypothesis.**

#### Severity: low

9. **`/userchats/{userId}/chats/{chatId}` second clause grants writes to a self-declared counterparty.** Likely intentional (start-chat flow mirrors a sidecar entry on the counterparty's side), but it's a deliberate trust decision Mastermind should explicitly bless or revoke. The audit cannot tell whether the client uses this path or whether it's dead code. **Repeated from 2026-05-19.**

10. **`chats/{chatId}` `delete` orphans the `messages` subcollection.** Children remain in Firestore but become unreadable (`get(parent)` returns null → read rule denies). Either declare `delete: if false` (messages are immortal — see §2.7 — so the chat doc arguably should be too), or arrange a backend cleanup. **Repeated from 2026-05-19.**

11. **`chats/{chatId}` `read` permits any authed user to probe arbitrary chat IDs and observe existence via the `resource == null` short-circuit.** Chat IDs are likely random / derived from sorted UIDs, so the exposure is bounded but real. **Repeated from 2026-05-19.**

12. **CLAUDE.md exists but is empty (0 bytes).** Untracked, present in working tree. The session-start primer says "Follow your CLAUDE.md (already in your context). The hard rules there are non-negotiable" — but the primer ALSO enumerates the hard rules itself, so the empty file does not block the session. Yesterday's audit flagged it as untracked-but-present; today it is still untracked AND empty. Mastermind should decide: commit a populated CLAUDE.md, leave the working file as a per-clone artifact, or delete it. **Updated from 2026-05-19 — was 0 lines yesterday too based on file size.**

13. **The `chats` composite index `users CONTAINS, lastUpdated DESC`** (in `firestore.indexes.json`) supports the canonical `where('users', 'array-contains', uid).orderBy('lastUpdated', 'desc')` chat-list query. Healthy. The `messages` composite index `seen ASC, senderId ASC` is a collection-group index suggesting an "unread for me" query (`collectionGroup('messages').where('seen', '==', false).where('senderId', '==', otherUid)`). If the client mounts such a query without first ensuring read-rule access via the parent chat doc lookup, it will return per-doc `permission-denied`. **New observation** — not flagged 2026-05-19.

14. **`origin/HEAD -> origin/feature/bootstrap`** rather than `origin/main`. Cosmetic — the default remote branch on GitHub is `feature/bootstrap`, not `main` (which is the prod deploy branch). Worth pointing the default to `main` (or `stage`) so `git clone` lands developers on a sane working branch. **New observation.**

## Files touched

- `.agent/audit-messaging.md` (new) — this audit, per brief.
- `.agent/last-session.md` (new, replaces empty 0-byte file) — exact copy.

**Not touched** (verified): `firestore.rules`, `firestore.indexes.json`, `firebase.json`, `.firebaserc`, `package.json`, `package-lock.json`, `tsconfig.json`, `vitest.config.ts`, `README.md`, `.github/workflows/*`, `start-session.sh`, `CLAUDE.md`. No `tests/` directory created. No `firebase deploy` invoked. No live-Firestore reads attempted.

## Tests

- n/a — read-only audit. There are no tests in the repo regardless (see §3).

## Cleanup performed

- none.

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change.
- `issues.md`: no change. Candidate `issues.md` entries (for Mastermind / Docs/QA to triage): adjacent observations 1, 2, 3, 4, 5 in §7 (high + medium). Audit does not write to `oglasino-docs/` per hard rules.

## Obsoleted by this session

- Nothing in `oglasino-docs/`. The earlier `.agent/2026-05-19-oglasino-firestore-rules-messaging-audit-1.md` is **not obsoleted** — it covers a different brief (defect-triage hypotheses for a chat-creation bug) and remains the right reference for that question. Today's audit is a structured inventory feeding Mastermind, with the seven explicit sections the brief specified. Overlap on the rule-by-rule content is intentional (rules file is unchanged since 2026-05-10).

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see "For Mastermind" below. Read-only audit; all three categories are "nothing."
- Part 4b (adjacent observations): §7 covers them; categorized by severity.
- Part 5 (session summary template): applied; audit content placed inline in "Implemented" per the brief's explicit instruction.
- Part 11 (trust boundaries): applied per-rule in §2 with explicit `client-trusted` / `auth-derived` / `server-read` / `path-bound` tags. The `notifications.create` rule is a CRITICAL trust-boundary violation (§7.1).
- Other parts touched: Part 9 (stack reference) — confirmed for rule-language semantics; no other touchpoints this session.

## Known gaps / TODOs

- **§5 (users field inventory) cannot be closed by this agent.** Requires Firebase Console access or authenticated CLI read against `oglasino-stage-49abb`. Defer to Igor or whoever has console access. This is a blocker for confidently deciding whether `users/{userId}` `allow read: if true` is safe.
- **Deploy-drift cross-check** (§4) cannot be closed by this agent. Requires Igor opening Firebase Console → Firestore → Rules on `oglasino-stage-49abb` and verifying the "Last published" timestamp matches 2026-05-10. One-minute check.
- **Client behavior questions** (§6 seams 2, 3, 5, 6, 9) require web / mobile / backend audits. This audit cannot answer them from the rules file alone.

## For Mastermind

### Part 4a simplicity evidence (required)

- Added (earned complexity): nothing — read-only audit.
- Considered and rejected: nothing — read-only audit.
- Simplified or removed: nothing — read-only audit.

### Brief vs reality

1. **The brief says "every top-level `function` declaration in the rules file. For each: ..."** — implies plural. **There is exactly one**: `isBlocked(userA, userB)`. Not a contradiction — just want to call out that the helper surface is minimal, which itself is information (Mastermind may want to decide whether to introduce more helpers for `isParticipant(chatId, uid)`, `isAdmin()`, etc. before the messaging rules grow further).

2. **The brief asks for `getAfter()` usage check.** **Not used anywhere.** Three `get()` calls in §2.7, no `getAfter()`. This is load-bearing for any client pattern that wants to atomically create chat + first message.

3. **The brief asks "what fields are actually stored in `users/{userId}`."** Cannot answer from this environment. Flagged in §5 with two options for Igor.

4. **The brief implies a `userchats/{userId}/chats/{chatId}` rule may use a different path or field shape than memory suggests.** Confirmed shape: path is exactly `/userchats/{userId}/chats/{chatId}`; key field on the sidecar doc is `withUserFirebaseUid` (singular). The chat doc itself uses `users` (array of 2 string UIDs). Asymmetry documented in §2.3 and §6.9.

5. **The brief mentions both `userblocks/**` and `userblocksReverse/**`.** Confirmed: both exist. Forward index `/userblocks/{userId}/blocked/{blockedId}` is the one `isBlocked()` reads. Reverse index `/userblocksReverse/{userId}/blockedBy/{blockerId}` is permitted by rules but not consulted by any rule — its consumer (if any) is in client or backend code. **New question for Mastermind: is the reverse index actually read anywhere? If not, drop it.**

6. **The brief mentions `notifications/**`.** Confirmed at `/notifications/{userId}/userNotifications/{notificationId}`. The create rule is broken (§2.8 / §7.1).

7. **CLAUDE.md.** The session-start primer says "Follow your CLAUDE.md (already in your context). The hard rules there are non-negotiable." CLAUDE.md is 0 bytes. I proceeded on the hard rules enumerated in the primer itself (no commits, no pushes, no cross-repo edits, no writes to the four oglasino-docs config files, calibrated challenge, session summary to two files). If Mastermind wants CLAUDE.md populated, that's a separate task. Not blocking this audit.

### Suggested next steps

- **Highest priority:** answer §5 (what fields are in `users/{userId}` on stage). Without this, Mastermind cannot confidently scope whether the messaging rule changes need to touch `users` at all.
- **Second priority:** decide whether the messaging feature will require atomic chat-create + first-message-create. If yes, switch the three `get()` calls in `messages` rules to `getAfter()`. If no, document the "must create chat first, await ack, then first message" client invariant explicitly in the messaging reference doc.
- **Third priority:** fix the `notifications` `create` rule (§7.1) — it's not in the messaging blast radius but it's a live high-severity bug.
- **Cross-repo audits needed:** web client (chat-creation payload shape, first-message flow, snapshot-listener ordering), backend (notification-creation identity and Admin-SDK posture), and the question of whether anyone reads `userblocksReverse`.
- **Test scaffolding (Phase B):** before any rule changes ship, write at least the regression tests for the existing rule shape. The scaffolding is in place; populating `tests/` should be a discrete pre-fix task.
