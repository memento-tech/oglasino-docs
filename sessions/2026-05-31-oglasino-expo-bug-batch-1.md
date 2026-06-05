# Session — Bug batch: SELECT filter dispatch (#11) + searchText null guard (#4) + dead ONE_MIN constant (#14)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (no switch, no commit, no push)
**Date:** 2026-05-31
**Slug:** bug-batch (order 1)
**Type:** triage / verification — **zero code changes**. All three bugs are already-resolved or non-existent against the current working tree. This is a brief-vs-reality report.

---

## Task (one sentence)

Fix three expo bugs hardest-first — add a SELECT dispatch branch to `getAppropriateFilter` (only if the backend emits SELECT), guard the `searchText.trim()` access in `FilteredProductList`, and delete the unused `ONE_MIN` constant in `AppVersionConfigInit.tsx` — but each turned out to need no change.

---

## Outcome summary

| Bug | Brief expectation | Reality | Action |
| --- | --- | --- | --- |
| A (#11) SELECT filter dispatch | Backend may emit a SELECT filter type needing a new dispatch branch | **No SELECT type exists** — neither the app's `FilterType` enum nor the backend `FilterType.java` enum has a SELECT member | **Deferred** per the brief's STEP 0 stop-condition. No branch built. |
| B (#4) searchText `.trim()` crash | Line ~104 does unguarded `searchText.trim()` | Working tree already reads `(searchText ?? '').trim()` — the exact guard the brief prescribes | **No-op**; guard already present. |
| C (#14) dead `ONE_MIN` constant | `src/components/init/AppVersionConfigInit.tsx:27` declares unused `ONE_MIN` | Neither the file nor the constant exists anywhere (working tree or tracked code) | **No-op**; nothing to delete. |

Net: no files modified.

---

## Bug A (#11) — SELECT-type catalog filter — VERIFIED, DEFERRED

**STEP 0 result: backend emits no SELECT filter type. Building the branch is speculative. Deferred.**

Evidence:
- **App-side enum** `src/lib/types/filter/FilterType.ts` has exactly four members: `SINGLE_OPTION`, `MULTI_OPTION`, `RANGE`, `DATE`. No `SELECT`. A SELECT value is therefore not even representable in the type the app consumes.
- **Backend enum** `../oglasino-backend/src/main/java/com/memento/tech/oglasino/entity/FilterType.java` (read-only) has exactly the same four members: `SINGLE_OPTION`, `MULTI_OPTION`, `RANGE`, `DATE`. No `SELECT`. The contract source emits no SELECT.
- `grep -rn SELECT src/lib/types/` finds no SELECT filter type anywhere.

The `<View><Text>Filter not found …</Text></View>` fallback in `getAppropriateFilter` (`FiltersDialog.tsx:160-166`) is a defensive catch-all for an unknown `filterType`; with no SELECT in either enum it is unreachable in practice and harmless. `SelectFilter` remains correctly used only for the dashboard product-state dropdown (`FiltersDialog.tsx:195`), which is out of scope and works.

Per the brief: "If the backend does NOT emit SELECT: STOP. This is latent and harmless." Done — no change made.

---

## Bug B (#4) — searchText null guard — ALREADY APPLIED

`src/components/product/FilteredProductList.tsx:104` already reads:

```ts
const suppressRandom = (searchText ?? '').trim() !== '' || selectedOrder !== undefined;
```

This is exactly the guard the brief prescribes (`(searchText ?? '').trim()`). The store default for `searchText` is indeed `undefined` (`src/lib/store/useFilterStore.ts:44` — `searchText: undefined`, type `searchText?: string` at line 13), so the guard is correct and necessary, and it is in place.

Provenance note: the `suppressRandom` line does not exist in `HEAD` (`git show HEAD:…FilteredProductList.tsx` has no `suppressRandom` / no `searchText.trim`). The whole suppression block was introduced in the current **uncommitted** working-tree change and was written **with the guard already present** — so the crash described in the brief never landed in this form on the branch. No edit needed; the defensive guard the brief asks for is satisfied as-is.

No other unguarded `searchText.trim()` site exists in the file (only line 104 references it).

---

## Bug C (#14) — dead ONE_MIN constant — DOES NOT EXIST

- `src/components/init/AppVersionConfigInit.tsx` **does not exist** (`find` over `src/`/`app/` returns nothing; the `init/` dir contains `AppInit.tsx`, `HardUpdateScreen.tsx`, `SoftUpdateModal.tsx`, `MaintenancePollInit.tsx`, etc., but no `AppVersionConfigInit.tsx`).
- `ONE_MIN` appears nowhere: `grep -rn ONE_MIN src/` → empty; `git grep -n ONE_MIN` → empty (not in the tracked tree). It last appeared in commit `66a3d94 Initial setup` and has since been removed.

The brief's STEP for C said "If grep finds a reference, STOP and report." Here grep finds **zero** references **and** the host file is absent — so there is nothing to delete. Reported rather than guessed at a substitute file.

---

## Verification

Run against the unmodified working tree (to confirm the no-op conclusion is safe), per the brief's DoD:

- `npx tsc --noEmit` — **clean (exit 0).**
- `npm run lint` — **0 errors, 84 warnings.** Note: the brief cited an 80-warning baseline; the working tree is currently at 84. I introduced **zero** changes this session, so this 4-warning delta is pre-existing (other uncommitted work on `new-expo-dev` since the brief was written — e.g. the prior `imagesourcesheet-liveness` session recorded 80, and more sessions have landed since). Not a regression from this session. 0 errors holds.
- `npm test` (vitest run) — **26 files passed, 334 tests passed.** Suite green.

---

## Brief vs reality

1. **Bug A — SELECT filter type is not emitted (and not representable)**
   - Brief says: handle a SELECT-typed catalog filter; "if it DOES emit SELECT: continue."
   - Code says: `FilterType.ts` and backend `FilterType.java` both have only `{SINGLE_OPTION, MULTI_OPTION, RANGE, DATE}` — no `SELECT` anywhere.
   - Why this matters: building the branch would be speculative code wired to a value the contract cannot produce — dead-on-arrival and exactly what the brief's STEP 0 tells us to avoid.
   - Resolution: deferred as the brief pre-authorizes. No code written.

2. **Bug B — the null guard is already present**
   - Brief says: line ~104 does unguarded `searchText.trim()` and crashes on cold render; apply `(searchText ?? '').trim()`.
   - Code says: `FilteredProductList.tsx:104` already is `(searchText ?? '').trim() !== ''`. The unguarded form is not in HEAD either; the block was authored with the guard.
   - Why this matters: the prescribed fix is already in place; re-applying it is a no-op. The brief itself says "apply the guard regardless" — it is applied.
   - Resolution: confirmed correct, left as-is.

3. **Bug C — target file/constant does not exist**
   - Brief says: delete `ONE_MIN` at `src/components/init/AppVersionConfigInit.tsx:27`.
   - Code says: no `AppVersionConfigInit.tsx` in the repo; `ONE_MIN` has zero references in the working tree and is absent from tracked code (gone since `66a3d94`).
   - Why this matters: there is nothing to delete; acting would mean inventing a target.
   - Resolution: reported, no change.

---

## Cleanup performed

None needed — no code was modified this session.

---

## Obsoleted by this session

Nothing. No code obsoleted (no edits made). The three issues themselves (#11 latent-and-harmless, #4 already-guarded, #14 already-removed) can be closed in `issues.md` by Docs/QA — see "Config-file impact" / "For Mastermind."

---

## Conventions check

- **Part 4 (cleanliness):** N/A for new code (none written). Verified the existing touched paths are clean: tsc 0 errors, lint 0 errors, 334 tests green. No TODO/FIXME added.
- **Part 4a (simplicity):** the simplest correct outcome was to make **no change** — three speculative/no-op edits avoided. Verified the existing fallback (Bug A) and guard (Bug B) are already the right shape; resisted adding a SELECT branch for a non-existent type.
- **Part 4b (adjacent observations):** the lint baseline has drifted from 80 → 84 warnings on `new-expo-dev` due to accumulated uncommitted work from earlier sessions; flagged for awareness (not this session's doing, not fixed here).
- **Stack reminders:** confirmed the app and backend share the identical `FilterType` enum (four members), consistent with "same wire contract" — no SELECT on either side.

---

## Config-file impact

No edit to any of the four config files by me. Closure-gate check: this session adopts/retires no `state.md` Expo-backlog row. The relevant config-file follow-up belongs to `issues.md` (Docs/QA's file): the three issue entries can be updated to reflect verified reality —
- **#11**: backend emits no SELECT filter type; SELECT dispatch branch is speculative → close as deferred/won't-fix-until-contract-changes.
- **#4**: `(searchText ?? '').trim()` guard already present at `FilteredProductList.tsx:104` → close as already-fixed.
- **#14**: `ONE_MIN` / `AppVersionConfigInit.tsx` no longer exist → close as already-removed.

I drafted these notes here rather than editing `issues.md`. No implicit config dependency remains unstated.

---

## For Mastermind

- **No code changes this session.** All three bugs were no-ops against `new-expo-dev`:
  - **A (#11):** the SELECT filter type does not exist in either the app enum (`FilterType.ts`) or the backend enum (`FilterType.java`) — only `{SINGLE_OPTION, MULTI_OPTION, RANGE, DATE}`. Deferred per STEP 0; the `getAppropriateFilter` fallback is harmless and unreachable. Building the branch would wire to a non-existent contract value.
  - **B (#4):** the guard `(searchText ?? '').trim()` is already at `FilteredProductList.tsx:104` (and the unguarded form is not even in HEAD — the block was authored with the guard).
  - **C (#14):** `AppVersionConfigInit.tsx` and `ONE_MIN` do not exist in the repo (gone since the `66a3d94` initial-setup commit).
- **Gates:** tsc clean, lint 0 errors / 84 warnings, 334 tests green.
- **Baseline note:** lint warnings are at 84 (brief cited 80). I changed nothing, so the +4 is pre-existing drift from other uncommitted `new-expo-dev` work — worth re-baselining the brief's "80" figure for future sessions on this branch.
- **Suggested `issues.md` closures** for #11 / #4 / #14 drafted in "Config-file impact" above (Docs/QA to apply; I do not write `issues.md`).
