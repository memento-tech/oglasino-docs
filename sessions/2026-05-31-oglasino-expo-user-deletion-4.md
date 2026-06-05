# Session summary

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (HEAD `b67627c`)
**Date:** 2026-05-31
**Slug / order:** user-deletion / 4 (prior `*-user-deletion-*.md`: `-1` read-only audit, `-2` Brief 1 foundation, `-3` Brief 2 danger-zone+delete flow)
**Task:** Brief 3 — user-deletion mobile adoption (3 of 3, the closer): activate the dormant `accountJustDeleted` slot in `AccountStateDialogsInit`; verify restoration is wired (zero code); add the grace-period/banned chat badge + send-gate + notices that mobile's messaging adoption never wired.

After this session the feature is **code-complete on `new-expo-dev` across Briefs 1–3, pending on-device Ψ verification. It is NOT `mobile-stable` until Ψ passes.**

---

## Brief vs reality (resolved against the as-built backend translation seed, not the stale spec §15)

Before writing code I found the brief's translation-key namespaces/strings disagreed with both the actual mobile key convention and the spec, so I verified against the authoritative source mobile actually fetches at boot — the backend seed SQL (`oglasino-backend/src/main/resources/data/translations/0001-data-web-translations-{EN,RS,RU,CNR}.sql`, read-only). Igor's directive (relayed in-session) was: ship the BANNED badge/notice **only if the keys are confirmed seeded in all four locales**, else STOP and fall back to gate-only. **All keys were confirmed in all four locales** — so the full implementation shipped. Findings:

1. **Task 1 dialog content namespace — brief was RIGHT, spec is stale.**
   - Brief says `tDialog('account.deleted.dialog.*')` (DIALOG namespace).
   - Spec §14.4/§15.4 lists these under `DASHBOARD_PAGES` (`dashboard.pages.account.deleted.dialog.*`).
   - **Seed reality:** rows are `('DIALOG', 'account.deleted.dialog.title' | '...scheduled.date' | '...restore.instruction')` — DIALOG, 4/4 locales. The brief's `tDialog(...)` is correct; **spec §15.4 is stale**. Used DIALOG.

2. **Task 1 close-button key — brief/spec key does NOT exist; used the seeded one.**
   - Brief/spec §15.3 name `buttons.account.deleted.go.home.label`. **No such seed row anywhere.**
   - **Seed reality:** the post-deletion acknowledge button is `('BUTTONS', 'account.deleted.acknowledge.label', 'OK')` — 4/4 locales. Used `tButtons('account.deleted.acknowledge.label')`.

3. **BUTTONS keys carry NO `buttons.` prefix — and the existing banned slot has a latent bug.**
   - Seed stores BUTTONS keys un-prefixed (`banned.go.home.label`, `account.deleted.acknowledge.label`, `images.import`, …). ~35 `tButtons(...)` call-sites in the app use the un-prefixed form and match the seed.
   - The lone outlier — `AccountStateDialogsInit.tsx:40` `tButtons('buttons.banned.go.home.label')` (the banned slot, from Φ1) — uses a spurious `buttons.` prefix that matches **no** seed row; the seeded key is `banned.go.home.label`. On-device this would render a raw key string / fall through to the InfoDialog default. **Pre-existing, never device-verified (Φ1 is Ψ-pending), out of this brief's scope (Task 2 = "don't touch the banned slot logic"). Left untouched; flagged in "For Mastermind."** My new post-deletion slot deliberately uses the correct un-prefixed `account.deleted.acknowledge.label`, so it does not propagate the bug.

4. **Task 3 BANNED keys exist in the seed but NOT in spec §15 — implemented per Igor's verify-first directive.**
   - Brief Task 3 names COMMON `user.banned.label` + MESSAGES_PAGE `user.banned.notice`. Spec §15.1/§15.5 omit both; spec §14.8 (web chat gating) gates only PENDING_DELETION.
   - **Seed reality:** `('COMMON', 'user.banned.label', 'Banned')` and `('MESSAGES_PAGE', 'user.banned.notice', 'This user has been banned and cannot receive messages.')` — both 4/4 locales. Confirmed → shipped the full BANNED badge + notice.

**All 8 keys used, verified 4/4 locales (EN/RS/RU/CNR):**
| Key | Namespace |
| --- | --- |
| `account.deleted.dialog.title` | DIALOG |
| `account.deleted.dialog.scheduled.date` | DIALOG |
| `account.deleted.dialog.restore.instruction` | DIALOG |
| `account.deleted.acknowledge.label` | BUTTONS |
| `user.scheduled.for.deletion.label` | COMMON |
| `user.banned.label` | COMMON |
| `user.pending.deletion.notice` | MESSAGES_PAGE |
| `user.banned.notice` | MESSAGES_PAGE |

---

## Implemented

- **Task 1 — `accountJustDeleted` slot activated** (`src/components/init/AccountStateDialogsInit.tsx`): replaced the dead `const accountJustDeleted: string | null = null;` + its stub comment with real store selectors (`accountJustDeleted`, `setAccountJustDeleted`); filled the empty effect mirroring the `restored`/`accountBanned` slots exactly — guard `if (!accountJustDeleted) return;`, open `INFO_DIALOG`, clear via `setAccountJustDeleted(null)` in the same effect (one-shot). No new dialog component (shared `INFO_DIALOG`, like the other two slots). No web-only `onClose` side effects (no `router.refresh`/`revalidateUserCache`). Title `tDialog('account.deleted.dialog.title')`; description = `account.deleted.dialog.scheduled.date` (`{ date }` interpolated) + `account.deleted.dialog.restore.instruction` joined `'\n\n'`; close button `tButtons('account.deleted.acknowledge.label')`.
- **Task 2 — restoration confirmed wired, ZERO code.** The success interceptor reads `x-account-restored === 'true'` → `authHooks.onAccountRestored?.()` (`api.ts:56-57`) → `setRestored(true)` (`authInterceptors.ts:10`); the `restored` slot in `AccountStateDialogsInit` is intact (`AccountStateDialogsInit.tsx:22-29`). Both hold → no changes. (No STOP-and-report condition: nothing missing/broken.)
- **Task 3 — chat badge + send-gate + notices** (`src/components/messages/Messages.tsx`, `src/components/messages/Chats.tsx`):
  - **3a peer-state derivation** (both files): `peerPendingDeletion = !peerDeleted && peer?.state === 'PENDING_DELETION'`, `peerBanned = !peerDeleted && peer?.state === 'BANNED'`. Peer accessor matches each file (`activeChat?.withUser` in Messages, `chatSummary.withUser` in Chats). The `!peerDeleted &&` guard mirrors the existing `showBlockBadge = !peerDeleted && …` convention in Chats — keeps the hard-delete `isDeletedPeer`/`user.deleted` path (§3.10 / Task 3e) authoritative and prevents a PENDING/BANNED badge on a hard-deleted peer.
  - **3b send-gate** (Messages): added `cannotSend = blocked || peerPendingDeletion || peerBanned`; the `MessageInput` `disabled` prop now consumes `cannotSend` (was `blocked`). Existing `blocked`/`blocking`/`blockedBy`/`peerDeleted` conditions untouched — only OR-extended.
  - **3c badge** (both, **inlined**): a small chip next to the display name — `bg-yellow-500` chip `user.scheduled.for.deletion.label` when pending, `bg-red-500` chip `user.banned.label` when banned. In Messages it sits in the header `Pressable` after the name; in Chats it sits in a new `flex-row` wrapper around the name (between name and `lastMessage`). Inlined (not a shared component) — see Decision.
  - **3d notices** (Messages only): two sibling notices added to the `ListFooterComponent`, beside the existing `blocking`/`blocked.by` notices, same `text-center italic text-primary-mild` style — `user.pending.deletion.notice` when pending, `user.banned.notice` when banned.
  - **3e** the `isDeletedPeer` + `tCommon('user.deleted')` hard-delete path is **untouched** in both files.

---

## Reported items (explicitly requested by the brief)

- **Close-button key used (Task 1):** `tButtons('account.deleted.acknowledge.label')` ("OK"). The brief's/spec's `buttons.account.deleted.go.home.label` does not exist in the seed; the seeded post-deletion button is `account.deleted.acknowledge.label` (BUTTONS, no `buttons.` prefix). See Brief-vs-reality #2/#3.
- **Date formatter used (Task 1):** `new Date(accountJustDeleted).toLocaleDateString()` — **no locale argument**, matching mobile's only existing `toLocale*` precedent (`app/(portal)/(secured)/notifications.tsx:69` `…toLocaleString()`, also no-arg). No new date library; no invented `cnr → sr` fallback — by passing no locale, formatting follows the device locale exactly as the rest of the app does, so the `cnr` case is handled the same way mobile already handles it. The flag carries the raw ISO `scheduledDeletionAt` string (Brief 1 convention); `{date}` is the ICU placeholder in `account.deleted.dialog.scheduled.date`.
- **Badge shared vs inlined (Task 3c):** **inlined** in each file. The chat components inline small status labels (Chats.tsx's block badge is inline JSX, `Chats.tsx:55-61`, not a factored component); per Part 4a and the brief's "match the codebase's convention," I inlined rather than introduce a new shared component.
- **Call button (out-of-scope note):** not encountered in either edited chat component — neither `Messages.tsx` nor `Chats.tsx` renders a call/phone button. Not touched. (Backend nulls the number for PENDING/BANNED regardless — audit Q5.)
- **Translation keys that failed to resolve:** none — all 8 keys verified present in the backend seed across all four locales (EN/RS/RU/CNR). Runtime resolution on-device is still the Ψ gate (no running app here), but the seed confirmation removes the "key absent" risk.
- **STOP-and-report conditions:** none hit. Task 2 found restoration intact (no STOP). The BANNED verify-first gate (Igor's directive) passed — keys confirmed seeded — so no STOP/gate-only fallback was needed.

---

## Files touched

- src/components/init/AccountStateDialogsInit.tsx (+15 / -5) — Task 1
- src/components/messages/Messages.tsx (+33 / -2) — Task 3a/3b/3c/3d
- src/components/messages/Chats.tsx (+19 / -2) — Task 3a/3c

(`userService.ts`, `authStore.ts`, the DTO types, `DeleteAccountConfirmationDialog.tsx`, `user.tsx` — Briefs 1+2, untouched this session.)

---

## Tests

- `npx tsc --noEmit` — **exit 0**.
- `npm run lint` (`expo lint`) — **0 errors, 80 warnings**. Baseline held (80/0 since `oglasino-expo-user-deletion-2`); none of the 80 warnings are in the three touched files. Zero new errors, zero new warnings.
- `npm test` (`vitest run`) — **24 files / 325 tests passed, 0 failed**. Same 325 baseline; no tests added (the new surface is Ψ-verified UI, per the brief — "you add none").
- `npx expo-doctor` — not run; no dependency changes this session.

---

## Cleanup performed

- Removed the dead `const accountJustDeleted: string | null = null;` and its two-line stub comment, and the empty stub effect body, from `AccountStateDialogsInit` (replaced by the live subscription + working effect). No dead `const` left behind.
- No commented-out code, debug logs, TODOs, or unused imports added. (`tDialog`/`tButtons`/`tCommon`/`View` were all already imported in their files.)

---

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no status flip asserted by me. The User-Deletion Expo-backlog row may now move to mobile **`in-progress` (code-complete across Briefs 1–3, pending Ψ)** — drafted in "For Mastermind." I assert **NOT `mobile-stable`** (Ψ on-device verification still owed; shares the pending iOS+Android rebuild dependency per Risk Watch).
- issues.md: no change asserted by me, but two items are surfaced for Docs/QA triage (see "For Mastermind"): the pre-existing banned-slot `buttons.`-prefix key bug, and the systemic spec §15 ↔ as-built-seed drift.

**Closure gate:** no implicit config-file dependency left unstated. The only state.md edit my work motivates is the Expo-backlog row move (drafted below). The spec §15 drift and the banned-slot bug are drafted for Docs/QA, not edited by me.

---

## Obsoleted by this session

- The `accountJustDeleted` stub (dead const + empty effect) in `AccountStateDialogsInit.tsx` — activated/replaced this session (its activation was explicitly Brief 3's job). The audit `-1`, Brief 1 `-2`, and Brief 2 `-3` summaries remain valid references.

---

## Conventions check

- **Part 4 (cleanliness):** confirmed — tsc/lint/test clean for touched paths; dead stub removed; no debug logs, no TODO/FIXME, no commented-out code, no unused imports.
- **Part 4a (simplicity):** no new abstractions. The post-deletion slot mirrors the two existing slots; the badges are inlined matching the existing inline block-badge convention; the gate is a one-line OR-extension. Considered and rejected a shared `ScheduledForDeletionBadge` component (web has one) — the chat components inline status labels, so a new component would add an abstraction the files don't use. Considered and rejected a `cnr → sr` locale map for the date — mobile passes no locale to `toLocale*`, so none is needed.
- **Part 4b (adjacent observations):** the banned-slot `buttons.banned.go.home.label` key bug (flagged, not fixed) and the spec §15 ↔ seed drift — both in "For Mastermind."
- **Part 6 (translations):** no new keys created. All 8 consumed keys verified seeded in all four locales against the backend seed. Reported the close-button and BANNED key corrections vs the brief/spec.
- **Part 7 (error contract):** N/A — no new service calls this brief.
- **Hard rules:** no commits/pushes/branch switches; no edits to `app.config.ts`/`eas.json`/`app.json`/native config; no cross-repo **edits** (the backend seed SQL was **read** read-only to verify the translation contract Igor asked me to confirm — no write); no writes to the four `oglasino-docs` config files; no new docs in this repo's `docs/`.

---

## Known gaps / TODOs

- None added. On-device Ψ verification of the post-deletion dialog, the grace-period badge/gate/notice, and the banned badge/notice is the remaining gate (owed across Briefs 1–3), depending on the pending iOS+Android rebuild.

---

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - *Added (earned complexity):* nothing beyond mirrors — the post-deletion slot mirrors `restored`/`accountBanned`; the chat derivations/badges/notices mirror the existing `peerDeleted`/`showBlockBadge`/block-notice patterns. No new components, hooks, or config.
  - *Considered and rejected:* a shared `ScheduledForDeletionBadge` component (inlined instead, per convention); a `cnr → sr` date fallback (no locale passed, per mobile precedent); routing PENDING/BANNED through the deleted-user path (kept separate per §3.2 / Task 3e).
  - *Simplified or removed:* the dead `accountJustDeleted` stub const + empty effect.

- **Two items surfaced for Docs/QA triage (I do not write issues.md):**
  1. **Pre-existing latent bug — banned-slot close-button key.** `src/components/init/AccountStateDialogsInit.tsx:40` uses `tButtons('buttons.banned.go.home.label')`, but the backend seeds the BUTTONS key un-prefixed as `banned.go.home.label` (4/4 locales). On-device this renders a raw key / falls back to the InfoDialog default close label. Introduced by Φ1, never device-verified. **One-line fix** (`buttons.banned.go.home.label` → `banned.go.home.label`); left untouched here as out of scope (Task 2 = don't touch the banned slot). Recommend an issues.md entry + a trivial follow-up so it's caught before Φ1/user-deletion Ψ. The new post-deletion slot already uses the correct un-prefixed form.
  2. **Spec ↔ as-built-seed drift in `features/user-deletion.md` §15 (stale, multiple keys).** Confirmed against the seed SQL: (a) §15.4 lists the post-deletion dialog keys under DASHBOARD_PAGES, but they're seeded under **DIALOG**; (b) §15.3 lists `buttons.account.deleted.go.home.label`, but the seeded button is `account.deleted.acknowledge.label` ("OK"), and the banned button is `banned.go.home.label` — neither carries a `buttons.` prefix; (c) §15.1 COMMON omits `user.banned.label` ("Banned"), which IS seeded; (d) §15.5 MESSAGES_PAGE omits `user.banned.notice`, which IS seeded; (e) systemically, §15 writes keys WITH namespace prefixes (`common.user.deleted`, `buttons.banned.go.home.label`) but the seed stores them WITHOUT the prefix (`user.deleted`, `banned.go.home.label`). Igor already noted §15 is stale (same class as the corrected sessionStorage drift). Recommend Docs/QA reconcile §15 against the seed at close-out. **Mobile was implemented against the seed (ground truth), so behavior is correct regardless.**

- **Drafted `state.md` Expo-backlog note (Docs/QA to apply, if Mastermind agrees):** User Deletion — mobile adoption **code-complete on `new-expo-dev` across Briefs 1–3** (auth-store flags + `deleteCurrentUser` + DTO fields + C-6 guard [B1]; Danger Zone + reauth/confirm dialog + delete→sign-out sequence [B2]; `accountJustDeleted` post-deletion dialog activated + grace-period/banned chat badge + PENDING_DELETION/BANNED send-gate + pending/banned notices [B3]). **NOT `mobile-stable`** — on-device Ψ verification owed (shares the pending iOS+Android rebuild dependency). Suggest the Expo-backlog row move to mobile `in-progress`.
