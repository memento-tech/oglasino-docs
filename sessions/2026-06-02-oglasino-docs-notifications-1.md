# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-06-02
**Task:** Notifications feature CLOSE-OUT config-file writes — apply the decisions/issues/state/spec edits owed at feature close; HOLD the mobile→`mobile-stable` flip pending Igor's on-device Ψ.

## Implemented

- **decisions.md ×2 (top, newest-first).** (A) Message notifications are push-only with per-chat
  OS-collapse anti-spam — no in-app doc, no server unread-state; trigger is the participant-verified
  `POST /api/secure/notifications/message-sent` ping; message text is display-only. (B) Prefer
  `affectedKeys().hasOnly([...])` over per-field pinning when a Firestore doc's schema is
  cross-repo-unverifiable; messages-rule keeps per-field by design (the two sibling rules differ by
  design). Entry A notes it supersedes the spec's old §2.1 server-unread wording.
- **issues.md — 2 RESOLVED + 1 grouped NEW (3 items), all dated 2026-06-02 at top.** RESOLVED:
  push-token sink ambiguity (fcmToken Firestore-vs-Postgres → Postgres `push_token` is sole sink,
  `userId` dropped from attach); ban/unban deep-link route-shape mismatch (standardized to
  `/owner/*`). NEW (open, mixed-severity grouped entry): web `/notifications` plain-router locale
  path (low); web SW focus-match never matches → new tab per tap (low/medium); backend `shown`
  field possibly dead across the stack with an ACTION to confirm/drop (low).
- **state.md.** New `### Notifications (in-app + push)` active-features block (`web-stable` / mobile
  code-complete on `new-expo-dev`, Ψ-pending, **NOT** `mobile-stable`); new Expo-backlog row
  (mobile `in-progress`); two new Risk Watch rows (30-key RS/RU/CNR native-translator review; the
  mobile-notifications Ψ-pending row carrying the (a)–(g) device checklist + the EAS/Firebase
  APNs+FCM-v1 credential-provisioning gate). **Mobile flip to `mobile-stable` deliberately HELD.**
- **features/notifications.md §2.1.** Replaced the "first-unread-per-conversation" server-unread
  wording with the as-built per-chat OS-collapse mechanism (Android collapseKey/tag, iOS
  apns-collapse-id, Expo collapseId, Web Push Topic+tag; chatId hashed to ≤32-char Topic limit;
  best-effort; no in-app fallback). Folded the now-redundant "Collapse key" and "Best-effort caveat"
  bullets into the new bullet (they were subsumed and the old "Collapse key" bullet carried a now-
  resolved `[inferred]` placeholder). Kept the "Push banner content" bullet (in the brief's "stays"
  list). Also updated the spec **Status** line (`planned` → `web-stable` with mobile-code-complete/Ψ
  caveat) and added a §14 Session log — close-out hygiene per CLAUDE.md responsibility #3.

## Files touched

- decisions.md (+2 entries at top)
- issues.md (+3 entries at top: 2 RESOLVED, 1 grouped NEW/open)
- state.md (+1 active-features block, +1 Expo-backlog row, +2 Risk Watch rows)
- features/notifications.md (§2.1 rewrite + 2 bullets consolidated; Status line; +§14 Session log)

## Tests

- N/A — markdown-only repo, no test suite. Manual revalidation: README.md has no notifications
  references (no sync owed); relative links in new content point to existing files
  (decisions.md / issues.md / features/notifications.md §8.1, §10).

## Cleanup performed

- Consolidated two now-redundant §2.1 bullets ("Collapse key" + "Best-effort caveat") into the
  rewritten Anti-spam bullet — removes duplicate content and a stale `[inferred — exact field
  names …]` placeholder that the as-built text resolves (Part 4 cleanliness: duplicate consolidated,
  stale reference updated).

## Config-file impact

- conventions.md: no change.
- decisions.md: 2 new entries titled "Message notifications are push-only with OS-collapse anti-spam
  (no in-app doc, no server unread-state)" and "Prefer `affectedKeys().hasOnly([...])` over per-field
  pinning when a Firestore doc schema is cross-repo-unverifiable".
- state.md: new Notifications active-features block; new Expo-backlog row; 2 new Risk Watch rows.
  `Last updated` already 2026-06-02 (no change). **No `mobile-stable`/`adopted` flip (HELD).**
- issues.md: 3 new entries authored (2 fixed/RESOLVED, 1 open grouped with 3 sub-items). No existing
  entries amended — verified no pre-existing open entry matched any of the five items (grep).

## Obsoleted by this session

- The spec §2.1 "first-unread-per-conversation" server-unread-state mechanism description — deleted
  and replaced with the as-built OS-collapse mechanism in this session.
- The §2.1 "Collapse key" and "Best-effort caveat" bullets — deleted in this session (folded into the
  rewritten Anti-spam bullet; their content, and a stale `[inferred]` placeholder, is subsumed).
- Nothing else.

## Conventions check

- Part 4 (cleanliness): confirmed — §2.1 duplicate content consolidated, stale `[inferred]`
  placeholder resolved, README revalidated (no notifications refs), new links checked against disk.
- Part 4a (simplicity) / Part 4b (adjacent observations): see "For Mastermind".
- Part 6 (translations): confirmed — recorded the 30-row RS/RU/CNR native-review obligation as a Risk
  Watch row (EN final), matching the standing placeholder-review precedent; no namespace invented.
- Other parts touched: Part 1 (doc style — ATX headings, relative links, kebab-case) confirmed;
  Part 3 (sole-writer of the four config files; all substantive edits came from the upstream Mastermind
  close-out brief) confirmed; Part 11 (trust boundaries) — the decisions/spec text records recipient
  is server-derived and message text is display-only, consistent with the rule.

## Known gaps / TODOs

- Mobile `mobile-stable` flip HELD per the brief — owed after Igor's on-device Ψ pass (and
  EAS/Firebase APNs+FCM-v1 credential provisioning). Closure gate respected: no flip applied.
- Notifications engineer session summaries (backend B1–B5 + cleanup, web W1–W3, firestore-rules,
  expo E1–E2) are NOT yet archived to `sessions/` — archival was not in this close-out's
  config-write scope and the source files were not surfaced. Noted in the Notifications block's
  Tasks-remaining and flagged below.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — markdown content only.
  - Considered and rejected: rejected adding the two OPTIONAL low hygiene items the brief listed
    (web `resolveNotificationAction` has no unit test; `router: any` in `notificationActions.ts`) —
    the brief explicitly marked them "NOT required to ship / optional, if Mastermind wants strict
    hygiene tracked." Left them out; flag below so Mastermind can opt in.
  - Simplified or removed: folded two redundant §2.1 bullets into one; removed a stale `[inferred]`
    placeholder.
- **Brief vs reality — three items (applied around them, none blocking):**
  1. **Backend "cleanup session": done vs. pending — internal brief inconsistency.**
     - Brief says: §3 feature row lists "+ cleanup session" inside backend's *code-complete* set,
       but §5 closure gate lists "once the backend cleanup session lands" as a *remaining* gate.
     - I see: the two statements conflict on whether the backend cleanup session has landed; no
       engineer session summaries were provided to verify either way.
     - Why this matters: state.md now records backend "+ cleanup" as complete per the feature row; if
       the cleanup session has not actually landed, that line is slightly ahead of reality.
     - Recommended resolution: Igor/Mastermind confirm whether the backend cleanup session landed; if
       not, I'll reword the backend bullet to "cleanup session pending."
  2. **Open flicker tracking-item not flipped (correctly, per hard rule).** issues.md
     `2026-06-01` "Mobile on-device UI/UX findings (batch)" carries an OPEN `[ ]` item: *"Notifications
     are unstable — visible at one moment, gone the next … expected to be resolved by the notifications
     feature."* This is exactly the spec §9.1 flicker the E1 fix closed (spec quotes the same phrase).
     A status flip is a *substantive* edit and was **not** in the brief, so per the hard rule I did
     **not** flip it. Recommend a follow-up brief (or Igor's go-ahead) to flip it to `[x]` "Fixed
     (code) 2026-06-02 … On-device confirmation owed (Ψ)" matching the sibling items in that batch.
  3. **Spec §7 drift from decisions Entry B (out of this brief's spec scope).** Spec §7 says the
     notifications update rule is tightened "(mirroring the `messages` update rule)", but decisions
     Entry B records the two rules use *different* idioms by design (notifications: `hasOnly`;
     messages: per-field). The brief scoped the spec edit to §2.1 only, so I did not touch §7. Recommend
     a one-line §7 correction in a future pass (drop "mirroring the messages update rule", note the
     `hasOnly` idiom) — substantive, needs the go-ahead.
- **Spec changes beyond the literal §2.1 instruction (flagged for transparency):** updated the spec
  Status line (`planned` → `web-stable`/Ψ caveat) and added a §14 Session log. Both are factual,
  sourced from the brief, and standard close-out hygiene (responsibility #3) — flagged in case
  Mastermind wanted the spec left at `planned`.
- **Archival owed:** the notifications engineer session summaries across the four sibling repos'
  `.agent/` folders still need archiving to `sessions/` (not in this config-write brief). Queue a
  Docs/QA archival pass once Igor surfaces / points at them.
- **Optional hygiene items (Mastermind's call):** the two low items in item 1 above — log them in
  issues.md or leave as accepted? Awaiting your call.
