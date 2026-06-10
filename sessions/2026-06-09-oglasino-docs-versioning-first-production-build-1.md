# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-06-09
**Task:** Apply the config-file edits that close the "versioning for first production build" work (versioning convention + OTA finding), placing Mastermind's drafted intent against the live files.

## Implemented

- **decisions.md** — new dated block at the top (newest-at-top), `## 2026-06-09 — Versioning for first production build; OTA does not yet exist (force-update gate is a store-redirect)`. Records the two coupled decisions as one entry, each with its rejected-alternatives list per Igor's standing preference: (1) versioning scheme (backend/web/expo SemVer 1.0.0 independent; router/firestore-rules/image-router bare integer `v1` in `package.json` + `CHANGELOG.md`); (2) OTA does not exist yet, the force-update gate is a store-redirect, OTA wired before first production build with `runtimeVersion: { policy: "appVersion" }` deferred into that track.
- **conventions.md** — added a concise `### Versioning` durable rule as a subsection under **Part 9 — Stack reference** (verified Part 9 is the stack/standards reference, matching Mastermind's expectation; `###` subsection matches the file's Part 6/12 style).
- **state.md** — added a new `## Versions` section near the top recording the per-repo version state as a table (backend/web/expo `1.0.0`, router/firestore-rules/image-router `v1`); refreshed the `**Last updated:**` line to 2026-06-09; added a Risk Watch row "OTA not yet wired — now an active pre-launch feature track (high)"; refreshed the store-redirect Risk Watch row.
- **issues.md (follow-up, Igor-authorized this session)** — reformatted the pre-existing 2026-06-09 store-URL entry from a malformed bare `ISSUE (...)` block to the file's convention (`## 2026-06-09 — iOS force-update store URL is still the placeholder`, `**Repo:** oglasino-expo · **Severity:** high · **Status:** open`). Igor confirmed the entry is legitimate and that the Android force-update URL was wired 2026-06-09 (only iOS remains placeholder).

## Files touched

- decisions.md (new top entry, ~45 lines)
- meta/conventions.md (+~4 lines: `### Versioning` under Part 9)
- state.md (+`## Versions` section; `**Last updated:**` refreshed; +1 Risk Watch row added; 1 Risk Watch row corrected — iOS-only placeholder)
- issues.md (reformatted the existing 2026-06-09 entry to convention; content preserved, no new issue created)

## Tests

- N/A (markdown-only repo). Grep-verified: `### Versioning` present in conventions.md; both 2026-06-09 decisions.md headers present and ordered; `## Versions` present in state.md; OTA + store-redirect rows present with no duplicate; no bare `ISSUE (` left in issues.md; `memento-tech.com` now referenced consistently (state.md row = iOS-only placeholder, issues.md = canonical entry, decisions.md 2026-05-29 = dated history that delegates to Risk Watch).

## Cleanup performed

- Reformatted the malformed `issues.md` entry to match the file's `## YYYY-MM-DD —` + `**Repo:** · **Severity:** · **Status:**` convention (it was the only bare `ISSUE (...)` entry in the file).
- Corrected the state.md store-redirect Risk Watch row to remove the now-false "both screens placeholder" framing (Android wired 2026-06-09; iOS only).
- The store-redirect placeholder row already existed — refreshed in place rather than duplicating. No dead links, no superseded content.

## Config-file impact

- conventions.md: durable versioning rule added — Part 9 (Stack reference) → new `### Versioning` subsection.
- decisions.md: new entry titled "Versioning for first production build; OTA does not yet exist (force-update gate is a store-redirect)". No edit to the dated 2026-05-29 expo-boot-redesign entry (append-only history; it delegates live tracking to state.md Risk Watch, now corrected).
- state.md: new `## Versions` section; `**Last updated:**` refreshed to 2026-06-09; Risk Watch — 1 row added (OTA not yet wired, high), 1 row corrected (store-redirect URL placeholder is iOS-only).
- issues.md: existing 2026-06-09 entry reformatted to convention (Igor-authorized). No new entry; no status change.

## Obsoleted by this session

- Nothing obsoleted. The store-redirect Risk Watch row was corrected/refreshed (not superseded); the 2026-06-06 Admin App Version Control decision and the 2026-05-29 expo-boot-redesign decision both stay accurate as history.

## Conventions check

- Part 4 (cleanliness): confirmed — malformed entry reformatted, stale "both placeholder" framing corrected, no duplicate row, no dead links.
- Part 4a (simplicity): confirmed — see "For Mastermind." Conventions rule kept to one paragraph; the OTA decision records that applying `runtimeVersion` now (inert) was rejected on Part 4a grounds.
- Part 4b (adjacent observations): the four brief-listed non-blocking follow-ups recorded in "For Mastermind" (not forced into the files).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 9 (Stack reference) — extended with the Versioning subsection; Part 3 (config-file write authority) — observed; decisions.md append-only nature — respected (no rewrite of dated history).

## Known gaps / TODOs

- The four non-blocking follow-ups in the brief were NOT written into state.md/issues.md (no clean backlog/Risk Watch home; each is a cross-repo engineering item needing an upstream draft to be substantive) — listed below for Mastermind to triage, per the brief's "list in your summary and do not force them in."

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): the `## Versions` table in state.md (six rows) — earns its place: state.md is the "where are we" source of truth and there was no existing home for current version state; a table shows six repos at a glance.
  - Considered and rejected: folding version facts into the Admin App Version Control feature section (rejected — versioning is cross-cutting across all six repos); a multi-paragraph conventions narrative (rejected — kept to one paragraph per "a rule, not a narrative").
  - Simplified or removed: removed the now-false "both screens placeholder" framing from the state.md store-redirect row; collapsed a malformed multi-bullet issues.md entry into the conventional prose form.

- **RESOLVED — the unattributed issues.md entry from the prior turn.** Igor confirmed it is legitimate and that the **Android** force-update URL was wired 2026-06-09 (only **iOS** remains placeholder, blocked on the App Store Connect Apple ID). Per Igor's instruction I (a) reformatted the malformed entry to convention in issues.md, and (b) fixed the contradiction in state.md (the store-redirect Risk Watch row now reads iOS-only). I did **not** alter the dated 2026-05-29 expo-boot-redesign decision in decisions.md, whose "both URLs placeholder" prose is accurate as-of-date and explicitly delegates live tracking to state.md Risk Watch — flagging this as a deliberate append-only-history call; say the word if you want that historical line annotated.

- **Brief's four non-blocking follow-ups — listed, not written into the config files** (no clean Risk Watch/backlog home; each needs an upstream draft to be a substantive entry):
  1. backend `/actuator/info` is exposed but unpopulated — setting `pom` `1.0.0` does **not** make any runtime endpoint report `1.0.0`; needs a `build-info` goal on `spring-boot-maven-plugin` if runtime version visibility is wanted later.
  2. `oglasino-image-router` CLAUDE.md describes a different worker than reality (says JOSE 6 asymmetric JWT + method-routing + `src/handlers/`; reality is HS256 symmetric in `src/auth/jwt.ts` + path-based routing). Provisional per its own note; needs an audit-rewrite. Outside my write boundary (that repo's own CLAUDE.md) — flag only. **It also makes me suspect our conventions.md Part 9 "Image | … JOSE 6 for JWT" stack row is stale** — worth a Mastermind decision on whether to correct Part 9.
  3. `oglasino-firestore-rules` `package.json` `scripts.test` is a bare `vitest run` with no emulator wrapper, contradicting its CLAUDE.md's "npm script handles emulator wiring"; only `test:watch` wraps with `emulators:exec`. Mirrors the wording in our conventions.md Part 4 cleanliness list for Firestore Rules — that line may also be inaccurate.
  4. `oglasino-image-router` `.gitignore` has a tooling-added `.agent/` line (not made by the engineer) that, if committed, would git-ignore the session summaries Docs/QA reads — flagged for Igor, not commit-fixed (cross-repo + the brief said do not commit-fix).

- **One naming note:** the brief's draft rule text named "image-worker"; the actual repo per conventions Part 2 is `oglasino-image-router`. I used the real repo name throughout (conventions, decisions, state).
