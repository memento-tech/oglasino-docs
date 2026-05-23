# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit the 7 open Dependabot branches, verdict each on safety, then apply the safe ones (no commits, no pushes)

## Implemented

Read-only audit of all 7 Dependabot branches on `origin/`, then applied the 3 verdicted SAFE without merging the branches (stayed on `dev`, edited equivalent changes directly on disk so Igor can review one diff instead of seven PRs).

**Audit verdicts:**

| # | Branch                                          | Type        | Verdict                                                                        |
|---|-------------------------------------------------|-------------|--------------------------------------------------------------------------------|
| 1 | `actions/checkout` 4→6                          | CI major    | **SAFE — applied**                                                             |
| 2 | `actions/setup-node` 4→6                        | CI major    | **SAFE — applied**                                                             |
| 3 | `eslint` 9.39.4→10.4.0                          | npm major   | SKIP — `eslint-config-next@16.2.6` ships plugins not yet certified vs ESLint 10 |
| 4 | `lucide-react` 0.577.0→1.16.0                   | npm major (1.0 cut) | SKIP — ~110 import sites, ~80+ distinct icon names; needs code sweep    |
| 5 | minor-and-patch group (react/react-dom/@types-react/typescript-eslint) | 4 patches | **SAFE — applied** |
| 6 | `@types/node` 22→25                             | npm major   | SKIP — outpaces Node 22 runtime (`.nvmrc=22`, `engines: ">=22"`)               |
| 7 | `typescript` 5→6                                | npm major   | SKIP — TS 6 not yet supported by `typescript-eslint@8.x` / `eslint-config-next`|

**Applied changes (3 of 7):**

1. `actions/checkout@v4 → @v6` across 9 sites in 3 workflow files. Project uses the action with no inputs — same surface in v6.
2. `actions/setup-node@v4 → @v6` across 9 sites in 3 workflow files. Inputs used (`node-version-file: '.nvmrc'`, `cache: 'npm'`) are stable through v5/v6.
3. Patch group via `package.json` floor bumps + `npm install`:
   - `react`: `^19.2.0 → ^19.2.6` (resolved `19.2.6`)
   - `react-dom`: `^19.2.0 → ^19.2.6` (resolved `19.2.6`)
   - `@types/react`: `^19 → ^19.2.15` (resolved `19.2.15`) — Dependabot only touched the lockfile; I additionally tightened the floor from `^19` because the loose `^N` floor is already flagged in `issues.md` 2026-05-16 ("Five loose `^N` major-only version constraints"). This consumes one of the five.
   - `typescript-eslint`: `^8.59.3 → ^8.59.4` (resolved `8.59.4`)

Verification: `npx tsc --noEmit` clean, `npm run lint` 182 warnings (3 below the 185 baseline from the morning's eslint-pass1 session — `typescript-eslint@8.59.4` dropped three false positives we don't need to track), `npm test` 154/154 passing.

## Files touched

- `.github/workflows/ci-dev.yml` (+10 / -10)
- `.github/workflows/deploy-prod.yml` (+4 / -4)
- `.github/workflows/deploy-stage.yml` (+4 / -4)
- `package.json` (+4 / -4)
- `package-lock.json` (+75 / -75)

Total: 5 files, +97 / -97. All on `dev`, unstaged, awaiting Igor's commit.

## Tests

- Ran: `npx tsc --noEmit` — clean (no output).
- Ran: `npm run lint` — 182 warnings (0 errors). Down from 185 morning-baseline.
- Ran: `npm test` — **154 passed, 0 failed**.
- Ran: `npm audit` — **1 moderate** vulnerability (see Adjacent observations).

## Cleanup performed

- None needed. The session adds nothing to clean up: workflow files flipped two version tokens each, `package.json` flipped four floor strings, `npm install` regenerated the lockfile. No commented-out code, no debug logging, no TODOs/FIXMEs added.

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change.
- `issues.md`: no change in this session. Two pending hooks for Docs/QA after Igor commits, drafted in "For Mastermind":
  - The 2026-05-16 "Five loose `^N` major-only version constraints" entry should be amended to "Four loose…" — `@types/react` got tightened to `^19.2.15` in this session.
  - A new `npm audit` finding (`protobufjs <=7.5.7` via Firebase) needs an `issues.md` entry. Was 0 vulns after the 2026-05-16 dep-upgrade; the advisory has been published since.

## Obsoleted by this session

- Five of the seven open Dependabot branches are now in a known state per the verdicts:
  - `dependabot/github_actions/actions/checkout-6` — superseded; the workflow edits in this session are equivalent. Can be closed without merge.
  - `dependabot/github_actions/actions/setup-node-6` — same; can be closed.
  - `dependabot/npm_and_yarn/minor-and-patch-2f611235da` — same; the `package.json` + lockfile changes in this session subsume it. Can be closed.
  - `dependabot/npm_and_yarn/eslint-10.3.0` — verdicted SKIP this session; Dependabot will re-propose when next eligible. Igor can either close it (Dependabot reopens on next release) or add an `ignore` rule for `eslint` major bumps in `.github/dependabot.yml` until eslint-config-next is certified.
  - `dependabot/npm_and_yarn/lucide-react-1.14.0` — verdicted SKIP; needs a dedicated brief for the icon-name code sweep. Igor can leave it open as a reminder, or close it and reopen when ready.
  - `dependabot/npm_and_yarn/types/node-25.6.2` — verdicted SKIP; structurally won't be safe until Node version in `.nvmrc` moves. Recommend adding an `ignore` rule for `@types/node` semver-major in `.github/dependabot.yml`.
  - `dependabot/npm_and_yarn/typescript-6.0.3` — verdicted SKIP; reopen when typescript-eslint + eslint-config-next certify TS 6.

## Conventions check

- Part 4 (cleanliness): confirmed. tsc clean, lint count down, tests at baseline, no new TODOs/FIXMEs, no debug logging, no commented-out code.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one new flag — `protobufjs` audit finding (see "For Mastermind").
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- The 4 SKIP-verdict Dependabot branches remain open on `origin/`. Each carries a specific gate before re-evaluation:
  - `eslint 10`: gate = `eslint-config-next` certifying ESLint 10.
  - `lucide-react 1.x`: gate = dedicated icon-rename audit brief.
  - `@types/node 25`: gate = Node version moving in `.nvmrc`, or Dependabot `ignore` rule.
  - `typescript 6`: gate = typescript-eslint + eslint-config-next supporting TS 6.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing. No new abstractions, no new config values, no new patterns.
  - Considered and rejected:
    - Merging the three SAFE Dependabot PRs through the GitHub UI individually instead of consolidating on `dev`. Rejected because (a) the brief said "fix as you recommended, do not push or commit anything," which rules out merging; (b) one local diff is easier to review than three PRs and one is cleaner for Igor's commit anyway.
    - Bumping `@types/react` to `^19.2.15` strictly (using `npm install --save-dev`) vs lockfile-only (matching Dependabot's exact behavior). Chose tighter floor because the looser `^19` is already flagged as a known anti-pattern in `issues.md` 2026-05-16; tightening it here is one fewer entry for future cleanup.
    - Running `npm audit fix` for the `protobufjs` finding. Rejected — it sits behind `firebase@12.13.0 → @firebase/firestore@4.14.1 → @grpc/proto-loader@0.7.15 → protobufjs@7.5.6`. No safe fix without bumping Firebase, and the brief is about Dependabot's recommendations, not unrelated audit remediation. Routed to "Adjacent observation" below.
    - Adding an `overrides` block to force `protobufjs >=7.5.8`. Rejected for the same reason — would need verification that the override doesn't break Firestore's gRPC path, which is out of scope.
  - Simplified or removed: `package.json` `@types/react` floor tightened from `^19` to `^19.2.15`. One step toward closing the 2026-05-16 loose-floor `issues.md` entry.

- **Adjacent observation (Part 4b) — new `npm audit` finding: `protobufjs <=7.5.7` (moderate, DoS via unbounded recursive JSON descriptor expansion).** Surfaced by `npm audit` after `npm install` this session — was 0 vulns on the 2026-05-16 dep-upgrade. Chain: `firebase@12.13.0 → @firebase/firestore@4.14.1 → @grpc/proto-loader@0.7.15 → protobufjs@7.5.6`. Severity: moderate. Cannot be fixed at the Web layer without a Firebase upgrade (Firebase's `@grpc/proto-loader` pin needs to move first) or a deliberately tested `overrides` block. I did not fix this because the brief is scoped to Dependabot recommendations only. Suggested `issues.md` entry below.

- **Drafted `issues.md` entries (Docs/QA, after Igor commits):**

  Entry 1 — amend existing 2026-05-16 entry "Five loose `^N` major-only version constraints remain in `package.json`":

  ```
  Down to four since the 2026-05-20 Dependabot-apply session tightened `@types/react ^19` to `^19.2.15`.
  ```

  Entry 2 — new:

  ```
  ## 2026-05-20 — Moderate-severity `protobufjs` advisory surfaced in `oglasino-web` via Firebase transitive chain

  **Severity:** moderate
  **Status:** open
  **Detail:** `npm audit` reports `protobufjs <=7.5.7` (GHSA-jggg-4jg4-v7c6 — DoS via unbounded recursive JSON descriptor expansion). The vulnerable copy is pulled in transitively: `firebase@12.13.0 → @firebase/firestore@4.14.1 → @grpc/proto-loader@0.7.15 → protobufjs@7.5.6`. Fix requires either (a) Firebase upgrading its `@grpc/proto-loader` pin to one carrying `protobufjs >=7.5.8`, or (b) a tested `overrides: { "protobufjs": ">=7.5.8" }` block — the override must be verified against Firestore's gRPC path because protobuf-loader is on the wire. Was 0 vulns on the 2026-05-16 web dependency-upgrade session, so this is a freshly-published advisory. Pre-launch this is tolerable (no PII at risk on a non-launched site), but should clear before production. Found by the 2026-05-20 Dependabot-apply session.
  ```

- **Suggested `.github/dependabot.yml` follow-up.** Three of the four SKIP-verdict branches will keep re-proposing on every release until either the gate clears or an `ignore` is added. Recommend adding (in a separate, deliberate brief — out of scope here):
  - `ignore: dependency-name: "@types/node", update-types: ["version-update:semver-major"]` until Node bumps in `.nvmrc`.
  - `ignore: dependency-name: "eslint", update-types: ["version-update:semver-major"]` until eslint-config-next certifies it.
  - `ignore: dependency-name: "typescript", update-types: ["version-update:semver-major"]` until typescript-eslint + eslint-config-next certify it.
  - Leave `lucide-react` un-ignored — Dependabot's nag is useful as a reminder that the code sweep is owed.

- **Closure gate:** two drafted `issues.md` edits pending Docs/QA application after Igor commits. No `conventions.md`, `decisions.md`, or `state.md` drafts.
