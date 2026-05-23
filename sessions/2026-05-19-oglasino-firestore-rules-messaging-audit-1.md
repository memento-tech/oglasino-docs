# Session summary

**Repo:** oglasino-firestore-rules
**Branch:** stage (read-only; no changes made to source files)
**Date:** 2026-05-19
**Task:** Audit the Firestore Security Rules governing messaging — chat document creation, chat reading, message subcollection creation and reading. Identify why the rules reject the start-message flow, and inventory the rule shape comprehensively.

## Findings

Status legend: `correct` / `broken` / `suspicious` / `partial` / `missing` / `n/a`.
Severity legend: `critical` (relates to the production defect) / `high` (broken behavior unrelated to current defect) / `medium` (gap or weakness) / `low` (cosmetic / hardening).

### 1. Repo structure

The repo is small. Top-level inventory:

- `firestore.rules` — 72 lines, single source of rules. Last modified 2026-05-10 (bootstrap commit).
- `firestore.indexes.json` — two composite indexes:
  - `chats` collection: `users CONTAINS, lastUpdated DESC` (for the "my chats" list query).
  - `messages` collection group: `seen ASC, senderId ASC` (likely for an "unread for me" query).
- `firebase.json` — declares `firestore.rules` and `firestore.indexes.json`; emulator wired on `127.0.0.1:8080` with UI on `:4000` and `singleProjectMode: true`.
- `.firebaserc` — aliases `stage → oglasino-stage-49abb`, `prod → oglasino-prod-7e5db`. **No `default` alias** (per README, intentional).
- `package.json` — scripts: `emulator` (uses `--project demo-oglasino`), `test` (`vitest run --passWithNoTests`), `test:watch`, `deploy:stage`, `deploy:prod`. Devdeps: `@firebase/rules-unit-testing@^4.0.1`, `firebase-tools@^13.29.0`, `vitest@^2.1.8`, `typescript@^5.7.2`. Node `>=20`.
- `.github/workflows/`:
  - `deploy-stage.yml` — push to `stage` → deploys rules+indexes to `oglasino-stage-49abb` via service-account JSON.
  - `deploy-main.yml` — push to `main` → deploys to `oglasino-prod-7e5db`.
  - `validate-pr.yml` — PRs to `stage` or `main` run `npm test` then a dry-run deploy against stage.
- `README.md` — declares "Phase A: mirrors production rules verbatim" and "Phase B: tests TBD."
- `vitest.config.ts`, `tsconfig.json` — scaffolding only; **no `test/` or `tests/` folder exists**.
- `firestore-debug.log` — leftover from local emulator session (not deployable, cosmetic).
- `.agent/` — was already present at session start, containing only `brief.md`. **`.agent/` did not need to be created this session.**
- `CLAUDE.md` — present (untracked per `git status`). The brief said "if CLAUDE.md doesn't exist, do not create it — flag for Mastermind." It does exist (untracked); see "For Mastermind."

Findings:

- **Repo structure** — `correct` / `low`. No surprises. Two real Firebase projects bound via `.firebaserc`; emulator uses the `demo-oglasino` sentinel.

### 2. The `chats/{chatId}` document rules

Located at `firestore.rules:40-47`. Verbatim:

```
match /chats/{chatId} {
  allow read: if request.auth != null &&
    (resource == null || request.auth.uid in resource.data.users);
  allow create: if request.auth != null
    && request.resource.data.users.size() == 2
    && request.auth.uid in request.resource.data.users;
  allow update, delete: if request.auth != null
    && request.auth.uid in resource.data.users;
  match /messages/{messageId} { ... }
}
```

Per-rule notes (these are top-level `read` / `create` / `update` / `delete` operations on the chat document):

- **read** (`firestore.rules:41-42`):
  - Auth required.
  - `resource == null` short-circuit: **any authed user can read a not-yet-existent chat doc** at any `chatId`. Likely intentional for "the listener subscribes before the doc exists" startup flows, but worth flagging — the rule trusts that creation-side rules gate the actual write. Status: `partial` / `low`.
  - If the doc exists, caller must be in `resource.data.users` (array).
  - There is no `list` keyword; in `rules_version = '2'`, this `read` rule covers both single-doc gets and list/query results. The `users CONTAINS` index supports the canonical `where('users', 'array-contains', uid)` query, and each returned doc satisfies the rule because of the `array-contains` constraint. Status: `correct` / `n/a`.

- **list** (no separate rule; covered by `read` above):
  - Snapshot listeners on a query like `where('users', 'array-contains', uid)` work because the docs returned all carry `uid` in `users`. Status: `correct` / `n/a`.

- **create** (`firestore.rules:43-45`):
  - Auth required.
  - `request.resource.data.users.size() == 2` — requires the field to exist, be an array, and have exactly two entries. If the web payload omits `users` or names the field differently (e.g. `participants`, `withUser`), this fails with permission-denied.
  - `request.auth.uid in request.resource.data.users` — caller must be one of the two entries.
  - **No `productId` check.** **No restriction on extra fields** (the rule does not lock down field shape — the chat doc may carry `lastMessage`, `lastUpdated`, `productId`, etc., and the rule does not care).
  - Status: `suspicious-critical` — the rule requires a `users` field of a very specific shape; a web-side payload-shape change is a plausible cause of the defect. See hypothesis 7.

- **update** (`firestore.rules:46-47`):
  - Auth required, caller must already be in `resource.data.users`.
  - **No field-shape constraint on the update.** A participant can mutate any field on the chat doc, including `users` (they could remove the other party, or add a third — though the create rule guards initial shape, update does not re-validate). Status: `partial` / `medium`. Not in scope for the current defect.

- **delete** (`firestore.rules:46-47`, shares the `update` rule):
  - Either participant can delete the chat doc. Cascade implications for the `messages` subcollection are not handled — child docs become unreadable (parent `get()` returns null). Status: `partial` / `medium`. Not in scope for the current defect.

Trust boundary check: the rule trusts only `request.auth.uid` and the contents of `request.resource.data.users` (for create) / `resource.data.users` (for everything else). For create, the contents come from the client — so the only thing the rule actually enforces is "you can create a chat document that includes you among exactly two users." It does NOT prevent the caller from creating a chat with an arbitrary other user against that user's will. Status: `partial` / `medium`. Not in scope.

### 3. The `chats/{chatId}/messages/{messageId}` subcollection rules

Located at `firestore.rules:48-62`. Verbatim:

```
match /messages/{messageId} {
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
}
```

Per-rule notes:

- **read** (`firestore.rules:49-50`):
  - Cross-document `get()` to the parent chat document.
  - **There is NO `resource == null` fallback** like the chat-doc read rule has. If the parent chat doc does not exist on the server when the rule evaluates, `get(...)` returns null and `null.data.users` errors → permission-denied. Status: `suspicious-critical` — this is the primary candidate for the snapshot-listener Error 1 if the client subscribes to messages before the parent chat exists.

- **list** (no separate rule; covered by `read` above):
  - Snapshot listeners on the messages subcollection trigger the per-doc read rule for each returned doc, and they trigger the `get()` against the parent. Status: `suspicious-critical`, same path.

- **create** (`firestore.rules:51-57`):
  - Auth required.
  - `request.resource.data.senderId == request.auth.uid` — caller must be the message's `senderId`. Status: `correct` / `n/a`.
  - Same `get(/databases/$(database)/documents/chats/$(chatId)).data.users` lookup — **parent chat doc must exist on the server at rule-evaluation time**. Status: `suspicious-critical` — this is the primary candidate for the message-send write error.
  - `isBlocked(senderId, receiverId)` — calls `isBlocked()` (top-level helper, see §4) which does two `exists()` checks. Costs two reads per message-create.
  - **No length/content/type validation on `text`/body fields.** No `productId` check, no `participants` check, no schema enforcement.

- **update** (`firestore.rules:58-60`):
  - Same parent `get()` (same `suspicious` failure mode if parent missing).
  - `request.resource.data.seen is bool` — the type check requires a `seen` field of type bool on the post-update document. **NOTE:** this rule lets either participant flip `seen` — including the message's own sender flipping `seen` on their own outgoing message. Likely intended as a read-receipt for the receiver, but the rule does not enforce "only the receiver may flip it." Status: `partial` / `medium`. Adjacent observation — see "For Mastermind."
  - Also: the rule does not enforce immutability of `senderId`, `receiverId`, `text` (or whatever body field exists) — a participant can rewrite any field on an existing message, including content. Status: `partial` / `medium`. Adjacent observation.

- **delete** (`firestore.rules:61`):
  - Hard `false`. Messages are immortal. Status: `correct` / `n/a`.

### 4. Other messaging-adjacent collections

Found in `firestore.rules`:

- `/users/{userId}` (`firestore.rules:9-12`)
  - read: open to anyone (auth not even required).
  - write: owner only.
  - Status: `partial` / `medium` for the wide-open read (outside messaging scope).

- `/userchats/{userId}` (`firestore.rules:14-15`)
  - `create, read, write`: owner only.
  - This is the parent doc of the per-user chat sidecar collection. Likely stores `lastUpdated` or similar.
  - Status: `correct` / `n/a`.

- `/userchats/{userId}/chats/{chatId}` (`firestore.rules:16-19`)
  - `create, read, update, delete` allowed if caller is `userId` OR `request.auth.uid == request.resource.data.withUserFirebaseUid`.
  - Stores a per-user sidecar entry pointing at the underlying `/chats/{chatId}` doc and naming the counterparty's Firebase UID via `withUserFirebaseUid`.
  - **Field-name asymmetry with `/chats/{chatId}`**: this sidecar uses `withUserFirebaseUid`, while the chat doc itself uses `users` (array). A confused web payload could easily route the wrong field shape to the wrong path.
  - **`request.resource` is undefined on `read` and `delete`** — so the second clause (`auth.uid == request.resource.data.withUserFirebaseUid`) only meaningfully applies to `create`/`update`. On read/delete, only the owner clause matters, which is fine but the rule's shape is misleading. Status: `partial` / `low`.
  - **Quirk:** the second clause allows the OTHER party to write into your sidecar doc as long as they declare themselves as `withUserFirebaseUid`. That means user B can create/update `userchats/<A>/chats/<chatId>` with arbitrary content as long as the doc's `withUserFirebaseUid` equals B's own uid. This is presumably to support reverse-side sidecar maintenance when a new chat is started. Status: `partial` / `medium`. Adjacent observation.

- `/userblocks/{userId}/blocked/{blockedId}` (`firestore.rules:22-29`)
  - read: owner of the block list only.
  - create: owner only, cannot block self.
  - delete: owner only.
  - No update.
  - Used by `isBlocked()` helper (top of file, `firestore.rules:4-7`) called from `messages` `create`. Status: `correct` / `n/a`.

- `/userblocksReverse/{userId}/blockedBy/{blockerId}` (`firestore.rules:31-38`)
  - read: by the blocked user (`userId`).
  - create/delete: by the blocker.
  - Reverse-index pattern: when user A blocks user B, A writes to `userblocks/A/blocked/B` AND to `userblocksReverse/B/blockedBy/A`. The reverse index lives at B's path, written by A. Status: `correct` / `n/a`. Note: the messaging path uses only the forward index via `isBlocked()`; the reverse index does not gate any messaging-related rule.

- `/notifications/{userId}/userNotifications/{notificationId}` (`firestore.rules:65-70`)
  - `read, update, delete`: owner only.
  - **`create: if request.auth == null || request.auth.token.admin == true`** — this allows **any unauthenticated client to create a notification document** for any user under any document id. **CRITICAL trust-boundary violation** if interpreted literally — `request.auth == null` is the unauthenticated case, not "no claim required." This is outside the messaging defect's blast radius but is the most serious thing in the rules file. Status: `broken` / `high` (out-of-scope for the messaging defect but flagged loudly in "For Mastermind").

- **Helper at top level** — `isBlocked(userA, userB)` (`firestore.rules:4-7`):
  - Two `exists()` checks (forward + reverse). Status: `correct` / `n/a`.
  - Top-level scoping means it is in scope across all collections, but only `messages.create` references it.

**Not found in the rules file**:

- FCM token / push-notification registration collections — `missing`. If the web/mobile clients write FCM tokens to Firestore (e.g. `/fcmTokens/{uid}`), the writes are default-denied. If they write via Cloud Functions or to the Firebase Auth user record / Firestore via Admin SDK, this is fine; if they write client-side, this would manifest as permission-denied at app startup, not at message-send. Severity: `medium` only if clients write client-side.
- Typing indicators, read receipts (other than the `seen` field on messages), chat preferences, muted-users, message reactions, attachment metadata — all `missing`. Same default-deny note.

### 5. Recent git history on the rules file(s)

```
$ git log --all --format='%H %ai %s' firestore.rules
9248c9c893786d183df94d703d705947cf7d6e06 2026-05-10 16:30:56 +0200 chore: bootstrap repo with rules, deploy workflows, PR validation
```

- **One commit, ever, on `firestore.rules`**: `9248c9c` on 2026-05-10 (nine days before today, 2026-05-19).
- The second commit on this repo, `ee8e54f Added indexes` (also 2026-05-10), modified only `firestore.indexes.json` — verified via `git show --stat ee8e54f`.
- Branches `stage`, `main`, and local `HEAD` are all at `ee8e54f` (verified via `git branch -a`).
- README states the rules file "mirrors production rules verbatim" as exported from the Firebase Console at bootstrap.

**Cross-reference verdict:** the rules file has NOT been edited in the last nine days, including the chat-creation and chat-read paths. Status: `n/a` — there is no recent rules change to suspect. The defect is either (a) a pre-existing rule the bootstrap export inherited, (b) deploy drift between repo and stage Firebase project, or (c) a client-side payload-shape change that finally collided with a long-standing rule shape. Hypotheses 6–8 below disambiguate.

### 6. Hypothesis check — `productId` on chat creation

**Verdict: DISCONFIRMED from the rules side.**

A direct text search of `firestore.rules` for `productId`, `product_id`, or `product` returns nothing. The chat-creation rule (`firestore.rules:43-45`) validates only:

1. `request.auth != null`
2. `request.resource.data.users.size() == 2`
3. `request.auth.uid in request.resource.data.users`

It does NOT require `productId`, `product_id`, `product`, or any product reference. The message-creation rule (`firestore.rules:51-57`) similarly does not reference `productId`. The rules do not restrict extra fields either — `productId` may be present or absent.

**Implication:** even if the web-side `MessageRequest` payload dropped `productId` from the first message or from the chat-creation payload, that change cannot by itself produce a permission-denied error from these rules. The hypothesis must be confirmed or refuted by the web audit on its own terms (e.g. is the dropped `productId` causing the web side to reshape OTHER fields, like `users`?).

### 7. Hypothesis check — participant validation

**Verdict: PLAUSIBLE.** The rule expects a very specific field shape on chat creation.

- Rule (`firestore.rules:44-45`):
  - `request.resource.data.users.size() == 2` — array field literally named `users`.
  - `request.auth.uid in request.resource.data.users` — the caller's UID is one of the entries.
- If the web payload uses any of these alternative shapes, the rule rejects creation:
  - `participants: [a, b]` — `users` does not exist → `.size()` errors.
  - `users: [{ id: a }, { id: b }]` — `request.auth.uid in users` is `false` for an array of objects.
  - `users: { a: true, b: true }` map shape — `.size()` undefined.
  - `withUser: { id: ... }` only (the sidecar's `withUserFirebaseUid` field reused) — `users` missing.
- The sidecar collection (`userchats/{userId}/chats/{chatId}`) uses `withUserFirebaseUid` (singular, the counterparty's UID). The `/chats/{chatId}` collection uses `users` (array of two). A web-side refactor that swapped the two would manifest exactly as the observed error.

**Implication:** the web audit should confirm whether the payload posted to `/chats/{chatId}` has a `users` field that is a string-array of size 2 containing the caller's UID. If it does, this hypothesis is disconfirmed. If the payload sends nested objects, a different field name, or a wrong-cardinality array, this is the cause of the chat-creation write error.

### 8. Hypothesis check — cross-document lookups

**Verdict: STRONGLY SUSPECTED as the root cause.**

Cross-document calls in messaging rules:

1. `messages` **read** (`firestore.rules:50`):
   - `get(/databases/$(database)/documents/chats/$(chatId)).data.users`
   - Returns: chat document.
   - Field read: `users` (array of UIDs).
   - Expected value: the caller's UID is contained in the array.
   - **Failure mode**: if the chat doc does not exist on the server at rule-evaluation time, `get(...)` returns null and `null.data.users` raises a rule-evaluation error → permission-denied. There is **no `resource == null` fallback** here unlike the chat-document `read` rule.

2. `messages` **create** (`firestore.rules:53`):
   - Same `get(...)` as above.
   - Same failure mode if the chat doc does not exist.

3. `messages` **update** (`firestore.rules:59`):
   - Same `get(...)` as above.
   - Same failure mode.

4. `isBlocked(senderId, receiverId)` helper (`firestore.rules:5-6`), called from `messages` **create**:
   - `exists(/databases/$(database)/documents/userblocks/$(userA)/blocked/$(userB))` (forward)
   - `exists(/databases/$(database)/documents/userblocks/$(userB)/blocked/$(userA))` (reverse)
   - `exists()` returns false (not null/error) when the doc is missing, so this is safe by design.

**Why this is the strongly-suspected root cause of the start-message defect:**

The "Start a new conversation from a product page" flow has a defining characteristic: **the chat document does not yet exist on the server when the first message is sent.** The client must create the chat doc and the first message essentially together. There are three common implementation patterns and all three have a known failure mode against these rules:

- **Pattern A — batched write (`writeBatch`):** the client writes both the chat doc and the first message in a single batched commit. Rule evaluation is per-write, evaluated against the pre-batch server state. The message-create rule's `get(/chats/$(chatId))` returns null because the chat doc doesn't yet exist in the visible snapshot → permission-denied.
- **Pattern B — transaction (`runTransaction`):** same issue. Rule `get()` does not see in-flight transaction writes.
- **Pattern C — sequential writes (chat first, await acknowledgment, then message):** this should succeed. The chat write fires first (chat-create rule does not depend on the message), commits, and the subsequent message write sees the now-existing chat doc.

**Snapshot-listener Error 1 (the read):** if the client mounts a snapshot listener on `/chats/{chatId}/messages` (or a single-message ref) BEFORE the chat doc exists — for example, the UI optimistically subscribes when the user clicks "Send Message" — the listener fires the messages `read` rule, whose `get()` returns null, and the listener detaches with `permission-denied`. This matches the observed error sequence (Error 1 on listener mount, Error 2 on the send write).

**The rules-side fix space** (not implementing — read-only audit, but documenting for the fix-spec):

1. Make the message-create rule use `getAfter()` instead of `get()`. `getAfter()` sees in-flight transaction/batch writes, so an atomic `{create chat, create first message}` batched write would pass. Requires client coordination — both writes must be in the same batch/transaction.
2. Drop the `get()` from message-create and instead require the client to include `request.resource.data.participants` (or read it from the message doc itself), then verify caller is one of them. Loses the "must be a real chat" invariant.
3. Add a `resource == null` fallback to the message-read rule for safety, matching the chat-doc read rule, so listeners-before-chat-exists don't fire permission-denied error events. (This is purely a UX papering-over; the underlying write issue remains.)

The right fix depends on the web client's actual flow (does it batch? sequential? listener-then-write?), which the web audit must answer.

### 9. Tests, if any

**Verdict: no tests exist at all.** Status: `missing` / `medium`.

- No `test/` or `tests/` directory in the repo (verified via `ls`).
- `package.json` `test` script is `vitest run --passWithNoTests` — passes trivially with zero test files.
- README explicitly says: "Phase B (TBD) — no tests committed yet."
- Devdeps for `@firebase/rules-unit-testing@^4.0.1` and `vitest@^2.1.8` are installed and ready, but unused.

**Implication for the messaging defect:** there is no automated coverage of the start-message flow, the message `get()` cross-document dependency, or any rule in the file. The defect could not have been caught by the test suite because the suite is empty. Adjacent observation: the CI `validate-pr.yml` workflow runs `npm test`, which trivially passes — there is no rule-correctness signal at PR time.

### 10. Deployment state

**Verdict: no obvious deploy drift on `stage`. Repo HEAD matches the only deployed rules-file revision in repo history.**

- Deploy mechanism: GitHub Actions auto-deploy on push.
  - `deploy-stage.yml` — push to `stage` branch → `firebase deploy --only firestore --project stage --non-interactive`. Service-account JSON from `FIREBASE_SA_STAGE` secret.
  - `deploy-main.yml` — push to `main` → same against `prod` project.
- Git state at audit time:
  - Local branch: `stage` at `ee8e54f`.
  - `origin/stage`: at `ee8e54f`.
  - `origin/main`: at `ee8e54f`.
  - `origin/feature/bootstrap`: at `9248c9c` (one commit behind).
- The `Added indexes` commit (`ee8e54f`) on 2026-05-10 pushed to both `stage` and `main`, which would have triggered both deploy workflows. The push deployed `firestore.rules` as it stands today (the rules content has not changed since the bootstrap commit).
- The README says rules in the repo were exported "verbatim" from the Firebase Console at bootstrap. So unless someone has edited rules in the Console *after* the 2026-05-10 deploy (out-of-band), the rules in stage and prod are identical to `firestore.rules` in this repo.

**Could there be drift?** Theoretically yes, in one direction:

- The Firebase Console allows manual editing. If anyone has edited rules in the Console for stage or prod since 2026-05-10, the deployed rules would diverge from the repo. The repo would not know.
- The audit cannot confirm or refute this — confirming requires reading the deployed rules via `firebase firestore:rules:get` or the Firebase Console, which (a) requires login, (b) is forbidden by the agent's hard rules (no firebase login, no live calls).

Status: `partial` / `low` for the drift question — the rules-repo side is internally consistent; the stage-Console side cannot be inspected by this agent. Igor or the Docs/QA agent can confirm by checking the Firebase Console "Rules" tab on `oglasino-stage-49abb` for the deployed revision's "Last published" timestamp. If the timestamp is 2026-05-10 (when the indexes commit was pushed), the repo and stage match. If it is later, someone edited via the Console.

## Files touched

- `.agent/2026-05-19-oglasino-firestore-rules-messaging-audit-1.md` (new) — this summary.
- `.agent/last-session.md` (new) — exact copy.

No edits to `firestore.rules`, `firestore.indexes.json`, `firebase.json`, `.firebaserc`, `package.json`, workflows, README, or any source-tree file. Read-only audit.

## Tests

- n/a (read-only audit; no test coverage exists in the repo regardless — see finding 9).

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change. (Several items below in "For Mastermind" are candidate `issues.md` entries for Mastermind / Docs/QA to decide on after the cross-repo seam analysis.)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see structured evidence in "For Mastermind." Read-only audit; all three categories are explicitly "nothing."
- Part 4b (adjacent observations): several flagged in "For Mastermind" (notification create rule, message-update rule scope, sidecar field-name asymmetry, user-read open, no tests).
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): applied throughout; the notification-create rule is a CRITICAL trust-boundary violation; flagged.
- Other parts touched: Part 9 (stack reference) — confirmed for stack identification; Part 5 (session summary template) — applied.

## Known gaps / TODOs

- The deploy-drift question (finding 10) cannot be closed by this agent. Confirming requires reading the deployed rules from the Firebase Console for the stage project. Defer to Igor or Docs/QA.
- The "is there a batched/transactional write in the web start-message flow?" question (finding 8) is answerable only by the web audit. This audit names the rules-side failure modes for each pattern.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — read-only audit.
  - Simplified or removed: nothing — read-only audit.

### Brief vs reality

1. **The brief's "withUser.id == otherParticipant"-style hypothesis assumes a different rule shape than actually exists.**
   - Brief says (hypothesis 7): "the chat-creation rule validates that `request.auth.uid` is one of the two participants on the chat document being created (a `withUser.id == otherParticipant` check, or similar)."
   - Rules say: chat-creation uses `request.resource.data.users.size() == 2 && request.auth.uid in request.resource.data.users`. The field is `users` (array), not `withUser.id`. No `participants` either.
   - There IS a `withUserFirebaseUid` field on the `/userchats/{userId}/chats/{chatId}` sidecar — a different collection, a different rule.
   - Why this matters: when seam analysis runs, the web side needs to know that the `/chats/{chatId}` doc requires a top-level `users` array of size exactly 2 with both Firebase UIDs as strings (not objects, not a map). Any other shape on the chat doc creation rejects with permission-denied. The sidecar has its own (different) field-name expectation.
   - Recommended resolution: when the messaging-reference doc is drafted, document the two field-name shapes side by side and call out the asymmetry explicitly. The fix spec should confirm web-side payload shape against this.

### Hypothesis verdicts (the three the brief asked for)

1. **`productId` hypothesis (brief §6): DISCONFIRMED from the rules side.** No rule references `productId`/`product_id`/`product`. Dropping `productId` from the web payload cannot, on its own, cause permission-denied at the rules layer. If the defect's cause is `productId`-shaped, the failure is happening before the request reaches Firestore (likely in client-side validation or in a payload-construction step that conditionally reshapes `users` when `productId` is missing).

2. **Participant validation (brief §7): PLAUSIBLE / unconfirmed pending web audit.** The rule requires `users` (array of 2, both string UIDs, caller is one). If the web payload uses a different field name, a different cardinality, or nested objects, the chat-creation rule rejects it. The sidecar collection uses `withUserFirebaseUid` (singular) — a field name a web-side refactor could plausibly confuse with `/chats/{chatId}`'s expected `users` array.

3. **Cross-document lookups (brief §8): STRONGLY SUSPECTED.** The `messages` `read`, `create`, and `update` rules all `get(/databases/$(database)/documents/chats/$(chatId)).data.users`. If the parent chat doc does not exist at rule-evaluation time, `get()` returns null and `null.data.users` errors → permission-denied. The start-message flow is exactly the case where the chat doc does not yet exist. This explains BOTH errors:
   - **Error 1 (snapshot listener `read`):** the UI mounts a listener on `/chats/{chatId}/messages` (or similar) before the chat doc is created → `get()` is null → listener detaches with `permission-denied`.
   - **Error 2 (message `create`):** the client batches `{chat, first message}` and the message rule's `get()` still does not see the in-flight chat → write rejected.

### Cross-repo seams (for the messaging reference / fix spec)

The rules embody the following assumptions about other repos. Each is a seam that needs explicit verification:

1. **The `/chats/{chatId}` document carries a top-level `users` array of exactly 2 string UIDs, including the writer's UID.** Web and mobile clients must produce this shape on initial chat creation. Verify against the web audit.
2. **The `/chats/{chatId}/messages/{messageId}` document carries `senderId` (string, matching the writer's UID) and `receiverId` (string).** Clients must produce both. The rules do not enforce a `text` or `body` field shape at all.
3. **For message `update`, the document must have a `seen` field of boolean type after the update.** Clients flipping `seen` must write a bool, not a string. (Adjacent observation: either participant — sender or receiver — can flip `seen`. This is permissive; intended semantics likely "receiver only.")
4. **The `/userchats/{userId}/chats/{chatId}` sidecar carries `withUserFirebaseUid` (string, the counterparty's UID).** This is the OTHER side of the chat-list lookup. Sidecar field-name does NOT match the chat doc's `users` array.
5. **The chat document must exist on the server BEFORE any message can be written or read.** The rules do not allow atomic-create of chat + first message because the message rules use `get()` (sees only pre-transaction state), not `getAfter()`. Web/mobile must therefore (a) write the chat doc first and await acknowledgment, then (b) write the first message — OR — switch the rules to `getAfter()` (out-of-scope for this audit, in-scope for the fix spec).
6. **User blocks are stored in `/userblocks/{userId}/blocked/{blockedId}` (forward index).** Blocked-by relations are stored in `/userblocksReverse/{userId}/blockedBy/{blockerId}` (reverse index, written by the blocker into the blocked user's path). Backend must keep both in sync if it writes via Admin SDK; clients must use the forward path for their own block lists.
7. **Notification documents at `/notifications/{userId}/userNotifications/{notificationId}` are created by the backend, never by the client.** See adjacent observation 1 below — the create rule is currently catastrophically permissive and should be tightened to require an admin custom claim, not the absence of auth.
8. **Custom claim `admin` exists on Firebase Auth tokens for backend service accounts.** The notification-create rule and (potentially elsewhere) reference `request.auth.token.admin`. The backend repo must mint this claim via Firebase Admin SDK; clients can never set it. Verify against backend audit.

### Adjacent observations (per Part 4b)

1. **CRITICAL: `notifications` create rule allows unauthenticated clients to create notifications for any user.** `firestore.rules:68-69` — `allow create: if request.auth == null || request.auth.token.admin == true`. The `request.auth == null` branch means an unauthenticated caller passes the create rule. This is almost certainly a bug — the intent was probably `request.auth != null && request.auth.token.admin == true` (admin only) or `request.auth.token.admin == true` (admin only, implies auth non-null). Severity: high (could allow notification spam or impersonation). Outside the messaging defect scope; flagged for separate fix. I did not fix this because it is out of scope.

2. **`chats/{chatId}` update rule has no field-shape constraint.** `firestore.rules:46-47` — any participant can mutate any field, including `users` (potentially removing the other party). Severity: medium. Outside the messaging defect scope.

3. **`chats/{chatId}/messages/{messageId}` update rule allows the sender (not just receiver) to flip `seen`.** `firestore.rules:58-60` — the rule does not constrain who flips the bit, only that the result is a bool. Senders can mark their own messages "seen." If the read-receipt UI relies on this, it could produce inconsistent receipts. Severity: medium. Out of scope.

4. **`chats/{chatId}/messages/{messageId}` update rule does not preserve immutability of `senderId`, `receiverId`, or message body.** Any participant can rewrite any field on a message (so long as the update leaves a bool in `seen`). Severity: medium. Out of scope.

5. **`/users/{userId}` document read is open to unauthenticated callers.** `firestore.rules:10` — `allow read: if true`. Public-profile semantics likely intended (display names on listings, etc.), but if user docs carry email addresses, phone numbers, or other PII, this is a leak. Severity: medium (cosmetic if user docs are intentionally public, high if PII is present). Out of scope — fix-spec for messaging shouldn't touch this.

6. **The sidecar `/userchats/{userId}/chats/{chatId}` allows the OTHER party to write entries into your sidecar collection by self-declaration.** `firestore.rules:17-18` — caller can be the owner OR can declare themselves as `withUserFirebaseUid`. Likely intentional for the "start a chat" flow (your counterparty mirrors a sidecar entry on your side), but worth documenting as a deliberate trust decision. Severity: low.

7. **No tests at all.** Finding 9 above. Severity: medium. Existing CI runs `npm test` which trivially passes — there is currently no rule-correctness gate at PR time. This is a known Phase B item per the README, but should be noted as a hardening priority once the messaging fix is decided.

8. **`firestore-debug.log` is checked into the repo.** `ls` shows it's tracked-or-untracked at the repo root. If checked in, it's noise that should be `.gitignore`d. (Did not run `git ls-files` to confirm tracking status to avoid file-system writes.) Severity: low.

9. **`CLAUDE.md` is present but untracked.** The brief said "if CLAUDE.md doesn't exist, do not create it — that's a separate decision flagged for Mastermind." It does exist (locally), but is not in git history. Mastermind should decide whether to commit it or remove it. Severity: low (process-level, not security).

10. **The chat-doc `read` rule allows authed users to read ANY non-existent chat doc** (`firestore.rules:41-42`, `resource == null` short-circuit). This is presumably to support snapshot listeners that subscribe before the chat exists, but it does expose chat-id existence: a caller can probe arbitrary chat IDs and see which return data vs which return "not found." Low severity (chat IDs are random / derived from UIDs in practice). Severity: low.

### Deploy-drift question (finding 10)

The repo and `stage` branch agree on rules content; stage and main have the same git SHA. But the audit cannot read the live deployed rules. **Recommended cross-check:** Igor opens the Firebase Console for `oglasino-stage-49abb`, Firestore → Rules tab, and confirms (a) the "Published" timestamp is 2026-05-10 (matching the `Added indexes` push), and (b) the rules text matches `firestore.rules` in this repo. If the Console shows a later timestamp or different text, there is drift and the on-disk file is not what's running on stage. This single check resolves the question.

### Suggested next steps

- The web audit must answer: (a) what is the exact payload shape posted to `/chats/{chatId}` for chat creation, (b) is the first message written in the same batch/transaction as the chat doc, or sequentially, and (c) does the snapshot listener mount before or after the chat doc create resolves. With those three facts plus this audit, seam analysis can identify the single root cause and pick the fix.
- The backend audit (if any messaging touchpoint exists) should confirm whether `admin` custom claim is minted on the Firebase Auth tokens used for `/notifications` writes — see seam 8 and adjacent observation 1.
- The messaging reference doc should explicitly document seam 5 (chat-must-exist-before-message-write) since it is the highest-impact, least-obvious rule in this file. The fix spec should choose between (a) client-side sequencing or (b) switching rules to `getAfter()`.
