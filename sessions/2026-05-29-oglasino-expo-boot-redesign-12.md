# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** expo-boot-redesign step 7c, Phase 1 — make the portal-mounts-only-in-`ready` invariant structural: gate the `<Stack>` in `app/_layout.tsx` on `bootStatus ∈ {ready, updating}`, keep AppInit on `ready` alone, do not re-introduce `<RequireBaseSite>`, plus the two in-session fixes (ProductList deps, bootStore AsyncStorage import). Phase 0 (`boot-redesign-11`) verified the loop hypothesis; Mastermind released Phase 1 with decisions locked.

## Implemented

- **Stack-level gate (the fix).** `app/_layout.tsx` — wrapped the portal mount point in `{(bootStatus === 'ready' || bootStatus === 'updating') && (<View className="flex-1"><Stack .../></View>)}`. The portal tree now mounts ONLY in `ready` or the `updating` freshness transient; on `booting`/`intro-picker`/`maintenance` it is unmounted, so no portal screen runs effects (e.g. the pre-Gate-1 `/product/search`) before a base site exists. `updating` is included so `pickBaseSite`/`setLanguage` (`ready → updating → ready`, no `booting` transit — Phase 0 §0f) keep the Stack mounted across the transient. This is the structural enforcement that replaces the `<RequireBaseSite>` per-screen guards deleted in 7b. Updated the two stale "always-mounted Stack" comments to describe the gate.
- **AppInit unchanged — `ready` alone.** `{bootStatus === 'ready' && <AppInit />}` left as-is (locked decision #3). Its effects (push registration, auth listener, chat init) are "first reach `ready`," not "portal is showing"; mounting on `updating` would re-register/double-subscribe on every freshness pass.
- **No `<RequireBaseSite>` / per-screen guards re-introduced** (locked). The Stack gate is the sole enforcement.
- **ProductList initial-load correctness (in-session fix — implemented as the *intent* of the locked instruction, not its literal form; see Brief vs reality).** `src/components/product/ProductList.tsx` — the mount-load effect's guard was `!selectedLanguage` while its deps were `[fetchPage]`, so a null-language mount would never re-fetch when language resolved. Added `selectedLanguage` to the deps so the load is correct regardless of mount timing, and added a `hasLoadedInitiallyRef` guard so the full `onRefresh()` (page-0 reset) fires **exactly once** — a later language *switch* does not reset the feed (that path stays owned by the dedicated language-change effect's `onRefreshCurrent`, which preserves pagination/scroll). This honours Mastermind's stated intent ("guard correct regardless of mount timing") without the scenario-6 regression the literal "add to deps" one-liner would cause.
- **bootStore AsyncStorage import — verified, nothing to remove.** `rg "AsyncStorage|async-storage" src/lib/store/bootStore.ts` → the **only** match is a doc-comment at `:208` ("read stored base site + language from AsyncStorage"); there is **no `import` of AsyncStorage** in the file (storage is delegated to the i18n/baseSites/checksum service modules). The Phase-0 flag was an artifact of a glitched file read that session; the real file never had the import. Item closed, no edit.

## Files touched

- `app/_layout.tsx` (+24 / -7) — Stack-level gate + two comment rewrites
- `src/components/product/ProductList.tsx` (+14 / -4) — `hasLoadedInitiallyRef` + reworked initial-load effect

(Both files remain untracked/uncommitted on `new-expo-dev` per state.md — Igor commits.)

## Tests

- `npx vitest run` → **185 passed** (11 files, 0 failed). Unchanged from 7b's 185; no test exercises `_layout.tsx`'s JSX conditional (the repo test env is `node`, no React renderer — per `MaintenancePollInit.tsx` note), and there is no `ProductList` test. Per the brief, no new test is required for the deps shore-up; the `_layout` render test (brief 1.4, optional) was skipped as impractical without a renderer — covered by the manual regression below.
- `npx tsc --noEmit` → clean (exit 0, no output).
- `npx expo lint app/_layout.tsx src/components/product/ProductList.tsx` → **exit 0, 0 errors**. `app/_layout.tsx`: clean. `ProductList.tsx`: **2 warnings, both pre-existing and NOT from my change** — `:63` (the favorites-refresh effect, untouched) and `:146` (the language-change effect / Effect C, untouched), each an `exhaustive-deps` warning that was already among the repo's ~70 standing warnings (7b summary). My reworked initial-load effect (`:102-116`) is warning-free: its `eslint-disable-next-line react-hooks/exhaustive-deps` covers only the deliberately-omitted `onRefresh` (identity changes with pagination state; the ref bounds the effect to one run). **Net new warnings introduced: 0.**
- `npx expo-doctor` — not run; no dependency changes (package.json/lock untouched).
- Did NOT run `npx expo start` (long-running dev server). Live device regression steps documented for Igor below.

## Cleanup performed

- Rewrote the two `app/_layout.tsx` comments that described the now-removed "always-mounted Stack" so they describe the Stack-level gate accurately.
- No commented-out code, no `console.log`, no `TODO`/`FIXME` added.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** the feature-close entry (owed since 7b) gains spec amendment **#6** (portal mount predicate, structurally enforced) — full text drafted in "For Mastermind." Running list stays at **six** (amendment #7 was retracted last release; `pickBaseSite` does not transit `booting`).
- **state.md:** no change this session. At feature close, the boot-redesign status flips to `shipped` once Igor's manual regression passes (Docs/QA applies).
- **issues.md:** no change. (No new issue: the AsyncStorage flag was a false alarm; recorded here, not in issues.md.)
- **Closure gate:** no unstated config-file dependency. The only config edit this session enables is amendment #6's inclusion in the feature-close `decisions.md` entry, drafted below for Docs/QA.

## Obsoleted by this session

- The "always-mounted Stack" property of `app/_layout.tsx` is gone — the Stack is now conditionally mounted. The two comments asserting it were rewritten (not left stale).
- Nothing deleted (no files, no code paths removed — `<RequireBaseSite>` was already deleted in 7b).

## Conventions check

- **Part 4 (cleanliness):** confirmed — net change is small and additive-by-necessity (a gate + a ref guard); stale comments rewritten; tsc/lint/tests green.
- **Part 4a (simplicity):** structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** the two parked items (dormant-while-`ready` poll; A6 deep-link redundant fetches) restated in "For Mastermind" as Mastermind parked them; no new ones.
- **Part 6 (translations):** N/A — no keys touched.
- **Spec Part 1 invariants + structural defense:** re-confirmed in "For Mastermind."

## Known gaps / TODOs

- No `TODO`/`FIXME` added.
- The exact component that issued the pre-Gate-1 `/product/search` was not identified (Mastermind confirmed from access-log timestamps that the 400 fired ~2ms before Gate 1's response, i.e. a portal mount before boot). The Stack gate makes its identity moot — no portal screen mounts pre-`ready`. Igor's scenario-1 access-log check is the confirmation.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** the Stack-level conditional (one predicate — the entire fix); `hasLoadedInitiallyRef` in ProductList (one ref — the minimum needed to make the initial load correct-regardless-of-mount-timing *without* regressing language-switch state preservation).
  - **Considered and rejected:** the literal "add `selectedLanguage` to the deps array" one-liner (rejected — it regresses scenario 6; see Brief vs reality); a per-screen guard / re-introducing `<RequireBaseSite>` (rejected — locked out, and the Stack gate subsumes it); a `_layout` render test (rejected — no React renderer in the node test env; manual regression covers it).
  - **Simplified or removed:** nothing removed; the gate *replaces* the structural role `<RequireBaseSite>` used to play (already deleted in 7b).
- **Three invariants — re-confirmed:**
  1. **One effect, empty deps:** `app/_layout.tsx:39-41` is still the single boot effect (`[]` → `start()`); the gate is a render-time JSX conditional, not an effect, and lists nothing. The other effect (Android nav-bar) touches no machine state.
  2. **Machine writes status, view reads it:** unchanged — the gate only *reads* `bootStatus`; no new status writer introduced (writers remain confined to `bootStore.ts`, Phase 0 §0d).
  3. **Re-entry destroys nothing:** unchanged — the gate doesn't touch store state. On `maintenance`/`booting` the Stack unmounts (screens reset on the next `ready`, which is correct and expected — scenario 5); on the `ready ↔ updating` switcher transients the Stack stays mounted, so screen state is preserved (scenario 6/7).
- **Structural defense — re-confirmed:** `bootStore.ts` imports zero `authStore` / chat-store (the only such mentions are the doc-comment at `:57-58`); `rg AsyncStorage` in the file is empty. The fix added no imports to the boot store.
- **Brief vs reality (the one deviation — needs your awareness, not a blocker):**
  - **The locked "add `selectedLanguage` to ProductList's deps, tiny one-line edit" conflicts with this brief's own scenario-6 DoD.** The code's existing comment states it: *"Re-running this effect on language change would reset the feed to page 0."* Effect A (`ProductList.tsx:101-115`) does a full `onRefresh()` (page-0 reset, scroll dropped); the deliberate language-change handler is the *separate* Effect C (`:126-140`) which calls `onRefreshCurrent()` to refetch loaded pages **preserving pagination/scroll**. Literally adding `selectedLanguage` to Effect A's deps fires the page-0 reset on every switch → violates scenario 6 ("Stack stays mounted, **screen state preserved**"). Under the gate, ProductList never mounts with null language anyway, so the literal change would fix an *unreachable* case while introducing a *reachable* regression.
  - **What I did instead:** kept `selectedLanguage` in the deps (so the load is correct regardless of mount timing — your intent) **and** added `hasLoadedInitiallyRef` so the full reset fires exactly once; switches stay with Effect C. This delivers the correctness shore-up with zero scenario-6 regression. If you'd prefer the strict literal form despite the page-0-reset-on-switch behaviour, say so and I'll change it — but I'd recommend against it.
- **Six spec amendments owed at feature close (running list — unchanged count):**
  1. Part 3: "dev seed" → "safe-default-all-envs seed" (B2).
  2. Part 5: `translations_<NS>` → `translations_<NS>_<lang>` payload key.
  3. Part 4: namespace count 20 (mobile) vs 22 (backend) — deliberate omissions.
  4. Part 8 step-6 test wording reconciliation (register-full-map clarification).
  5. Gate 4 register step includes `i18n.changeLanguage(activeLang)` (7b finding).
  6. **NEW — Portal mount predicate is `bootStatus ∈ {ready, updating}`, structurally enforced at the Stack-level gate in `app/_layout.tsx`.** `<RequireBaseSite>` was the prior per-screen enforcement (deleted 7b, replaced by this structural gate in 7c). The "no mount→status-write path" structural-defense claim is amended to the convergent form: the only such path (product deep-link auto-switch, `product/[...productData].tsx:96-129`) is `ready`-guarded and convergent (idempotent once the base site matches), not self-reinforcing.

  (Amendment #7 from the previous release stays **retracted** — `pickBaseSite` writes no status and goes `ready → updating → ready`, so the spec transition table is correct as-is.)
- **Parked (recorded, not in this brief — per your GO):**
  1. The dormant-while-`ready` maintenance poll (spec table row "ready → maintenance detected" not wired; only the maintenance-clear edge is). Decide later: implement runtime detection from `ready`, or amend the spec table.
  2. The A6 deep-link redundant-fetch churn (a few extra `getPortalProductDetails` while a foreign-base-site product deep-link converges). Efficiency only, no correctness impact.
  3. The exact identity of the pre-Gate-1 `/product/search` source. The gate kills it whichever component it is.

### Manual verification — for Igor (run after commit, on a real device; backend access log open)

1. **Fresh install / cleared storage → intro picker.** Access log shows **NO `/product/search` (or any portal fetch) before `/maintenance/active`**. The 400 is gone because the request is never made. *(critical retest)*
2. Pick a base site → header + product list render, no restart.
3. Cold-restart with stored site → reaches `ready` with zero translation/catalog fetches.
4. Cold-restart with maintenance ON → maintenance screen; no portal fetches during boot.
5. Toggle maintenance OFF → returns to `ready` within ~5s. Stack remounts → fresh screen state on return (expected — unmount/remount).
6. Language switch via PortalConfigDialog → new language registers AND activates; **Stack stays mounted, screen state preserved** (no splash flash; the feed is *not* reset to page 0 — confirms the ProductList ref-guard behaves).
7. Base-site switch via PortalConfigDialog/UserMenu → `updating` transient; **Stack stays mounted**; new content renders.

If any `/product/search` (or any portal-screen fetch) fires before Gate 1's response in scenario 1's access log, the fix is incomplete — report which fetch.

### decisions.md — amendment #6 text (for the feature-close entry, Docs/QA to fold in)

> **Spec amendment #6 (expo-boot-redesign, step 7c):** the portal mount predicate is `bootStatus ∈ {ready, updating}`, enforced structurally by a single Stack-level conditional in `app/_layout.tsx` (not per-screen). This replaces `<RequireBaseSite>` (deleted in 7b) as the enforcement of the portal-mounts-only-in-`ready` invariant; `updating` is included so freshness/language/base-site transients (`ready → updating → ready`) do not unmount the portal. The Part 1 "no mount→status-write path" structural-defense claim is amended to its convergent form: the sole such path (the product deep-link base-site auto-switch) is `ready`-guarded and idempotent once the base site matches, so it cannot drive a boot loop.

- Nothing else flagged.
