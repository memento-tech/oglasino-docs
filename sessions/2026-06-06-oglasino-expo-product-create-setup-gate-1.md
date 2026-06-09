# Session summary

**Repo:** oglasino-expo
**Branch:** brief named `new-expo-dev`; working tree is on `main` (identical trees — see below)
**Date:** 2026-06-06
**Task:** READ-ONLY AUDIT — audit the product-creation entry path on mobile and the missing "user basic data" setup gate.

## Implemented

Read-only audit, no code changed. Findings written to `.agent/audit-product-create-setup-gate.md` answering the seven inventory items plus a Seams section. Headlines:

- **Four** entry points open `NEW_PRODUCT_DIALOG`, not one: BottomBar (`BottomBar.tsx:144`), DashboardSidebar (`DashboardSidebar.tsx:63`), and two free-zone CTAs (`free-zone.tsx:62,118`). None gates on baseSite/regionAndCity.
- `openDialogSafe` (`BottomBar.tsx:36`) checks only auth-store hydration + `user` presence; checks nothing about user content. It is local to BottomBar; the other three sites don't use it.
- The dialog registry (`dialogRegistry.ts`) has **no** `USER_BASIC_DATA_SELECTOR` equivalent.
- **Crash point identified:** `AddUpdateProductDialog.tsx:85` — `userBaseSite.code === selectedBaseSite.code` dereferences a null `user.baseSite` (effect guards `user`/`selectedBaseSite` at line 81 but not `user.baseSite` at line 82). Thrown from a `useEffect` → red-screen. Null `regionAndCity` does **not** crash (defensively read at `BasicInfoProductDialog.tsx:58,261`).
- Mobile has a region/city edit surface (`app/owner/user.tsx` → `updateUser`) but **no baseSite-setup surface at all** (`UpdateUserDTO` has no `baseSite` field). No setup gate anywhere.
- `AuthUserDTO.baseSite`/`regionAndCity` are typed **non-nullable** (`AuthUserDTO.ts:9-10`); the crash implies the backend emits null for a setup-incomplete user — a type-vs-reality contradiction.
- Trust boundary: the create payload (`toCreateWirePayload`) deliberately omits baseSite/regionAndCity (server-derived). The client gate is **UX-only**; the server is load-bearing. Exact backend reject behavior is not verifiable from this repo (backend-audit scope).

## Files touched

- `.agent/audit-product-create-setup-gate.md` (new, audit output)
- `.agent/2026-06-06-oglasino-expo-product-create-setup-gate-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten, exact copy of this summary)
- No source files touched.

## Tests

- Not run. Read-only audit, zero source changes — no touched paths to lint/type-check/test.

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change required by this audit. (This is a Phase-2 audit; the feature has no `state.md` Active-feature row or Expo-backlog row yet — Mastermind will author the spec/row in later phases. Nothing to add or remove now.)
- issues.md: no change (findings are captured in the audit + flagged below for Mastermind; not authoring issues from an engineer session).

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; no dead code/imports/logs introduced.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no translations touched).
- Other parts touched: Part 8 (architectural defaults) and Part 11 (trust boundaries) — applied to inventory item 7; confirmed the create payload omits server-derived fields, server is the boundary.

## Known gaps / TODOs

- Backend behavior for create-by-setup-incomplete-user (reject code/status vs. location-less product) is unverified — needs the paired backend audit. Stated explicitly in audit §7 and Seam 4.
- Web's actual `USER_BASIC_DATA_SELECTOR_DIALOG` flow was taken from the brief, not read (mobile scope) — treated as a seam/assumption, not confirmed fact.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Branch discrepancy (process note):** brief said `new-expo-dev`; the checked-out branch is `main`. I did **not** checkout (hard rule). Verified `git diff --name-only main new-expo-dev` = 0 files (identical trees; `main` is one no-op "initialize main branch" commit ahead), so the audit reads the same code. Flagging so you know which branch the working tree was on.

- **Adjacent observation 1 (Part 4b) — HIGH:** `AddUpdateProductDialog.tsx:82-85` dereferences `user.baseSite` with no null guard; combined with the `AuthUserDTO.baseSite` non-null type, this is the user-facing crash. The defensive fix (guard `user.baseSite` and gate the wizard) plus a web-parity basic-data collector is the likely Phase-5 work. Not fixed — out of scope (read-only).

- **Adjacent observation 2 (Part 4b) — MEDIUM:** `AuthUserDTO` declares `baseSite`/`regionAndCity` non-nullable while `BasicInfoProductDialog.tsx:58,261` already treats `regionAndCity` as optional — the type and its consumers disagree internally. The type likely needs to become nullable to force guards across the create path. Not fixed — out of scope.

- **Suggested next step:** pair this with a backend audit confirming `/auth/firebase-sync` emission for `wasRegister:true` users and `/secure/products/create` enforcement when baseSite/region are unset — that determines whether the mobile fix is "gate + non-null type correction" or also needs a backend contract change. This is the §3 seam-analysis input.

- Config-file closure gate: no config-file edits required this session (all four: "no change" above). No pending drafts.
