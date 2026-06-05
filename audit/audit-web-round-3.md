# Audit: package.json loose major-only version constraints

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-27
**Task:** Read-only audit of major-only `^N` version constraints in `package.json`

---

## Q1 — Quote the relevant lines

Every entry in `dependencies` and `devDependencies` using a bare major-only `^N` constraint (no minor/patch):

**devDependencies:**

| Line | Entry | Constraint |
|------|-------|------------|
| 75 | `"@tailwindcss/postcss"` | `"^4"` |
| 79 | `"@types/node"` | `"^22"` |
| 81 | `"@types/react-dom"` | `"^19"` |
| 89 | `"tailwindcss"` | `"^4"` |
| 91 | `"typescript"` | `"^5"` |

**dependencies:** none. All entries in `dependencies` use `^MAJOR.MINOR.PATCH` form.

**Comparison to the issues.md 2026-05-16 list of five:**

The original five: `@tailwindcss/postcss`, `@types/react`, `@types/react-dom`, `tailwindcss`, `typescript`.

- `@types/react` (line 80): now `"^19.2.15"` — **resolved.** Already in `^MAJOR.MINOR.PATCH` form.
- The other four remain as bare-major.
- `@types/node` (line 79): `"^22"` — **not in the original five** but is bare-major. The issues.md entry said it was "tightened" from `^20` to `^22`, but that was a major-version bump, not a `^MAJOR.MINOR.PATCH` normalization. It still has the same loose-floor problem.

**Result:** 4 of the original 5 remain. 1 additional entry (`@types/node`) is the same shape. **5 total bare-major constraints exist today.**

---

## Q2 — Installed versions per lockfile

| Package | Lockfile entry | Installed version |
|---------|---------------|-------------------|
| `@tailwindcss/postcss` | `node_modules/@tailwindcss/postcss` (line 4442) | `4.3.0` |
| `@types/react` | `node_modules/@types/react` (line 4622) | `19.2.15` |
| `@types/react-dom` | `node_modules/@types/react-dom` (line 4631) | `19.2.3` |
| `tailwindcss` | `node_modules/tailwindcss` (line 11837) | `4.3.0` |
| `typescript` | `node_modules/typescript` (line 12121) | `5.9.3` |
| `@types/node` | `node_modules/@types/node` (line 4607) | `22.19.19` |

---

## Q3 — Lockfile committed

- `package-lock.json` exists at the repo root.
- `package-lock.json` is not in `.gitignore` (grep returned empty).
- `package-lock.json` is tracked in git (`git ls-files --error-unmatch package-lock.json` returned the filename, no error).

**Confirmed:** the lockfile is committed and pins the installed versions. The loose constraints are a real concern (different developers or CI running `npm install` without a lockfile would drift).

---

## Q4 — Documented reasons to keep major-only constraints

- **No comment fields:** `package.json` contains no `//`-prefixed fields. Standard JSON has no comments.
- **No `overrides`/`resolutions` relevance:** the `overrides` section contains only `"postcss": ">=8.5.10"` — unrelated to the five entries.
- **No dep-management config:** no `RENOVATE.md`, `dependabot.yml`, `.renovaterc`, `.renovaterc.json`, `renovate.json`, or `package.json5` found.
- **Git log:** 14 commits have touched `package.json`. None contain messages explaining the loose constraints. The constraints appear to be artifacts of initial installation (e.g., `npm install tailwindcss` without specifying a version resolves to `^4` when the current major is 4).

**Result:** no documented reason to keep major-only constraints.

---

## Q5 — Fix shape verification

| Package | Current | Installed | Proposed | Assessment |
|---------|---------|-----------|----------|------------|
| `@tailwindcss/postcss` | `"^4"` | `4.3.0` | `"^4.3.0"` | Purely mechanical. |
| `@types/react-dom` | `"^19"` | `19.2.3` | `"^19.2.3"` | Purely mechanical. |
| `tailwindcss` | `"^4"` | `4.3.0` | `"^4.3.0"` | Purely mechanical. |
| `typescript` | `"^5"` | `5.9.3` | `"^5.9.3"` | Purely mechanical. |
| `@types/node` | `"^22"` | `22.19.19` | `"^22.19.19"` | Purely mechanical. Not in original five, but same shape. |

All five are mechanical normalizations. None of the installed versions are far behind latest — these are actively maintained packages and the lockfile was recently updated (the `package-lock.json` is modified in the current working tree per `git status`). No upgrade discussion needed.

---

## Q6 — Side effects of the change

Per npm semver rules:

- `^4.3.0` allows `>=4.3.0 <5.0.0` — same upper bound as `^4` (`>=4.0.0 <5.0.0`). Only the floor rises.
- `^19.2.3` allows `>=19.2.3 <20.0.0` — same upper bound as `^19`. Only the floor rises.
- `^5.9.3` allows `>=5.9.3 <6.0.0` — same upper bound as `^5`. Only the floor rises.
- `^22.19.19` allows `>=22.19.19 <23.0.0` — same upper bound as `^22`. Only the floor rises.

**This is a metadata tightening, not a behavior change.** The lockfile continues to pin the exact resolved versions. Developers running `npm install` against a populated `node_modules` (or with the lockfile present) resolve to identical versions before and after the change. The tightened floor only matters if someone deletes `node_modules` AND `package-lock.json` — in which case the tightened constraint prevents npm from resolving to an older minor that might lack features the codebase depends on.

**Explicitly confirmed:** no runtime effect, no behavior change.

---

## Q7 — `@types/node` history

Current `package.json` line 79:

```json
"@types/node": "^22",
```

This is **bare `^22`**, NOT `^22.x.x`. The issues.md 2026-05-16 entry says:

> "Down from six since the 2026-05-16 dependency-upgrade brief tightened `@types/node ^20` to `^22`."

That was a **major-version bump** (from major 20 to major 22), not a `^MAJOR.MINOR.PATCH` normalization. The constraint went from one bare-major to a different bare-major. `@types/node` has the same loose-floor problem as the other four entries.

The actual precedent for "what tightening looks like in this repo" is **`@types/react`**, which shows `"^19.2.15"` — the correct `^MAJOR.MINOR.PATCH` form.

Installed version per lockfile: `22.19.19` (line 4607). Proposed: `"^22.19.19"`.

---

## Verdict for A11

**Smaller than five (one resolved) + one additional entry = five total.**

- `@types/react` has been resolved since the 2026-05-16 entry — now `"^19.2.15"`. Four of the original five remain.
- `@types/node` at `"^22"` is the same bare-major shape and should be included in the tightening pass.
- **Total: 5 entries to tighten** (`@tailwindcss/postcss`, `@types/node`, `@types/react-dom`, `tailwindcss`, `typescript`).
- **All five are mechanical normalizations** against installed versions. No upgrade discussion needed. Engineer can apply directly.

| Package | From | To |
|---------|------|----|
| `@tailwindcss/postcss` | `"^4"` | `"^4.3.0"` |
| `@types/node` | `"^22"` | `"^22.19.19"` |
| `@types/react-dom` | `"^19"` | `"^19.2.3"` |
| `tailwindcss` | `"^4"` | `"^4.3.0"` |
| `typescript` | `"^5"` | `"^5.9.3"` |

---

## Cleanup performed

None needed (read-only audit).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: the 2026-05-16 entry should be updated by Docs/QA to reflect that `@types/react` is resolved, `@types/node` should be added to the list, and the total remains at five. Draft in "For Mastermind" below.

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed (read-only audit, no code changes)
- Part 4a (simplicity): N/A (read-only audit)
- Part 4b (adjacent observations): see "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **issues.md update draft.** The 2026-05-16 entry "Five loose `^N` major-only version constraints remain in `package.json`" should be amended by Docs/QA:
  1. Remove `@types/react` from the list (resolved — now `^19.2.15`).
  2. Add `@types/node` to the list (still bare `^22`, same shape).
  3. Update the "Found in" line to: `@tailwindcss/postcss`, `@types/node`, `@types/react-dom`, `tailwindcss`, `typescript`.
  4. Note that `@types/node`'s tightening from `^20` to `^22` was a major-version bump, not a `^MAJOR.MINOR.PATCH` normalization.

- **Adjacent observation (Part 4b).** The brief expected `@types/node` to show `^22.x.x` as the "precedent shape." It does not — it shows bare `^22`. The actual precedent for the correct shape is `@types/react` at `^19.2.15`. Mastermind's model of the `@types/node` tightening was inaccurate — it was a major bump, not a floor normalization. Low severity; no user-facing impact; surfaced here so the A11 fix brief references the correct precedent.
