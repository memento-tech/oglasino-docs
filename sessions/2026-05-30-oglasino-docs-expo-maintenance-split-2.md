# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-05-30
**Task:** Closing docs sweep for expo-maintenance-split — PART 1 only (doc content updates to the two-key KV model; PART 2 ship-record gated on Igor confirming the stage rehearsal passed).

## Implemented

PART 1 of the brief — the eight live-doc groups from the sweep audit's update map, edited to the two-key model (`maintenance.web.active` + `maintenance.backend.active`, composed per-client at the worker). None of these assert the feature shipped. PART 2 (decisions.md new entry + state.md flip) was **not** run — it is gated on the stage rehearsal.

- **`infra/cloudflare/maintenance.md`** (live runbook): four-key KV table with a new per-client composition note; gate matrix rewritten to web (`webDown = web.active OR backend.active` + admin bypass) vs mobile (`backend.active OR probe-fail`, no admin bypass) paths; deploy diagram PUT shows web-deploy vs backend-deploy keys; deploy-flow step 3, restore step 3, and the droplet `maintenance-off.sh` row updated to "whichever flag(s) the deploy set"; `use.backend.check` section re-scoped to mobile-only probe against `/actuator/health/readiness`. Grep-clean of the `maintenance.active` literal.
- **`infra/overview/secret-inventory.md`**: the four deploy-workflow secret rows (web stage/prod, backend stage/prod) now say web workflows write `maintenance.web.active`; backend workflows write `maintenance.backend.active` + `maintenance.web.active` (+ admin.bypass).
- **`infra/vercel/deployments.md`**: flip steps name `maintenance.web.active`; added a note that a web deploy flips only the web flag (mobile unaffected).
- **`meta/devops-chat-bootstrap.md`**: trust-boundary KV-flag example names the two keys.
- **`features/version-checksums.md`**: deferred-bucket `MaintenancePageController.toggle()` line annotated as resolved by expo-maintenance-split (controller deleted; toggle moved to admin endpoint).
- **`features/expo-release-readiness.md`**: chat-I 5s maintenance-poll item (scope line + Risk Watch entry) struck and marked absorbed/superseded by expo-maintenance-split.
- **`future/db-overload-protection.md`**: the two literal auto-trip flag references → `maintenance.backend.active` (a saturation auto-trip is a backend signal); internal `auto.maintenance.*` config-flag names left alone.
- **`issues.md`** (append-only): appended `> Resolved by expo-maintenance-split` to the open 2026-05-29 redundancy entry; appended a `> See expo-maintenance-split…` cross-ref to the open 2026-05-28 Φ3 cold-start entry; added three new 2026-05-30 low-severity entries (no method-security test coverage; backend docs 01/02/04/06/11 stale on the single key; router maintenance HTML missing `X-Robots-Tag` on stage). Closed 2026-05-15 router-worker findings left untouched.

`infra/oglasino-devops-blueprint-v5.md` left untouched per the brief (frozen historical blueprint). `sessions/` and `.agent/` untouched.

## Files touched

- infra/cloudflare/maintenance.md (11 edits)
- infra/overview/secret-inventory.md (4 edits)
- infra/vercel/deployments.md (2 edits)
- meta/devops-chat-bootstrap.md (1 edit)
- features/version-checksums.md (1 edit)
- features/expo-release-readiness.md (2 edits)
- future/db-overload-protection.md (2 edits)
- issues.md (2 appends + 3 new entries)

## Tests

- N/A (markdown-only repo).
- Verification: `grep -rn "maintenance\.active"` across live docs (excluding `sessions/`, `.agent/`, and v5) returns only legitimate non-targets — explanatory mentions, designated-leave entries, and PART-2 scope (see "Known gaps").

## Cleanup performed

- Dead links: none introduced; the new vercel→runbook relative link verified to resolve.
- Stale references: every brief-scoped stale `maintenance.active` single-key reference in the eight target docs updated to the two-key model.
- Duplicate content: the three new issues.md entries were grep-checked against existing entries — no duplicates.
- Superseded prior work of mine: none (this is the first content sweep for this feature; the `-1` session was the read-only audit).

## Config-file impact

- conventions.md: no change.
- decisions.md: **no change in PART 1** (the 2026-05-15 entry is superseded, not rewritten — that is the PART 2 new-entry job, gated on the rehearsal).
- state.md: **no change in PART 1** (status flip planned→shipped is PART 2, gated).
- issues.md: 2 entries amended (append-only resolution/cross-ref notes on the 2026-05-29 and 2026-05-28 entries); 3 new low-severity entries authored.

## Obsoleted by this session

- The chat-I 5s maintenance-poll line item in `features/expo-release-readiness.md` (scope + Risk Watch) — struck in place, marked absorbed by expo-maintenance-split.
- The single-`maintenance.active` model in the eight target docs — superseded by the two-key wording; old wording replaced, not left behind.
- Nothing deleted as a whole file.

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): N/A (no abstractions/config introduced — markdown content edits only). See "For Mastermind" structured evidence.
- Part 4b (adjacent observations): two flagged in "For Mastermind" (boot-redesign gate table; maintenance.md line-5 wording).
- Part 6 (translations): N/A this session.
- Part 3 (config-file writes): confirmed — issues.md edits are this upstream-briefed substantive change; PART 2 config writes deliberately withheld pending the rehearsal gate (closure gate honored — see below).
- Part 5 (session archival): this session is markdown content edits in-repo; no sibling-repo session files archived.

## Known gaps / TODOs

- **PART 2 not run** (decisions.md new entry + state.md planned→shipped) — correctly deferred until Igor confirms the stage rehearsal passed. The session is intentionally open on PART 2 per the brief's explicit gate; this is not a missed config-file dependency.
- Residual `maintenance.active` literals in live docs, all intentional / out of PART-1 scope:
  - `decisions.md` 2026-05-15 entry (lines ~1265/1267/1269/1277) and `decisions.md:1352` allowlist line — PART 2 supersedes via a new entry; history not rewritten.
  - `state.md:128` — the "Expo Maintenance Split" entry correctly *describes* the split (explanatory); the status flip is PART 2.
  - `issues.md` closed 2026-05-15 router findings (955/966/968) — brief says leave (accurate historical records); plus my new entry that intentionally names the stale key.
  - `features/expo-maintenance-split.md` — the spec itself (brief: leave entirely).
  - `features/expo-boot-redesign.md:135-136` — see "For Mastermind."

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — content edits only; no new abstraction, config value, or pattern.
  - Considered and rejected: a full structural rewrite of `infra/cloudflare/maintenance.md` (rejected — surgical edits preserve the runbook's shape and the genuinely model-neutral prose, per concise>comprehensive); flipping the `Status:` fields of the annotated issues.md entries to `fixed` (rejected — the brief said "append a line," and PART 1 must not assert the feature shipped, which is right since state.md still has the feature as `planned`).
  - Simplified or removed: struck the absorbed chat-I 5s-poll item from `expo-release-readiness.md`'s open queue.

- **Brief-vs-reality (one item, resolved with Igor):** the brief's item 8 told me to append a `> Fixed by…` note to a HIGH `issues.md` entry for the unauthenticated `POST /api/public/maintenance/toggle` defect. **That entry does not exist** in `issues.md` (grepped exhaustively; the spec §10 claims it was "logged" but it never was). Igor's call (2026-05-30): **skip it** — do not create an open-then-fixed entry on the same day; the closure is recorded in `features/expo-maintenance-split.md` §10 and will be recorded in the PART 2 decisions.md entry. Applied the rest of item 8 as written.

- **Adjacent observation 1 (low):** `features/expo-boot-redesign.md:135-136` — the boot-gate table still encodes `maintenance.active === true/false` as gate 1's condition. The sweep audit classified this doc 0-edits / model-neutral (it documents the *shipped* boot-redesign gate behavior), and the brief did not list it, so I left it. But once expo-maintenance-split ships (PART 2), gate 1's dedicated-poll condition is replaced by the worker `X-Oglasino-Maintenance` signal (spec §5.3), at which point this table becomes historically stale. Recommend a one-line update or a "superseded by expo-maintenance-split" note when the feature ships. Did not fix — out of PART-1 scope and would assert an unshipped change.

- **Adjacent observation 2 (low):** `infra/cloudflare/maintenance.md:5` — "The gate is split into two flags…" predates this change and now reads ambiguously next to the two *maintenance* flags (it actually refers to the maintenance + admin-bypass lockdown step). Left as-is (not a brief target, substantive rewording would need a drafter). Flag if you want it clarified.

- **For PART 2 (when the rehearsal passes):** the residual `decisions.md` single-key references (incl. `decisions.md:1352` `/api/public/maintenance/*` allowlist line) and `state.md:128`/`:336` are handled by the supersession + status flip, not by in-place rewrite. The PART 2 decisions.md entry should explicitly note that the `MaintenancePageController` deletion makes the `decisions.md:1352` allowlist reference moot.

## Closure note

PART 1 is content-complete and grep-verified. The session remains open on PART 2 by design (the brief's hard gate: do not record the feature as shipped until Igor confirms the stage rehearsal passed). No upstream draft for PART 1 is left un-applied. If the rehearsal does not pass, report back to Mastermind rather than running PART 2.
