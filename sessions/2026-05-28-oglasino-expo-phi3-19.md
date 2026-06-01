# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev (the brief's `dev` slot — Igor's deliberate isolation per the 2026-05-25 "Expo foundation work is tentative" decisions.md entry)
**Date:** 2026-05-28
**Task:** Φ3 Brief 19 — D2 (RequireBaseSite render gate on four base-site-dependent screens) + D5 (ProductList `selectedLanguage` guard) + cleanup (remove `[DBG]` log in `api.ts`, optionally remove misspelled TODO in `AppVersionConfigInit.tsx`).

## Implemented

- **D2 — `RequireBaseSite` component created.** New `src/components/context/RequireBaseSite.tsx`, verbatim per the brief's 3a sketch. Reads `selectedBaseSite` from `useAppState` and renders `fallback` (default `null`) when undefined, `children` otherwise. Import path `@/components/context/AppContext` and field name `selectedBaseSite` both verified against the live `AppContext.tsx`.
- **D2 — four base-site-dependent screens wrapped.**
  - `app/(portal)/(public)/index.tsx` — wrapped `<View><FilteredProductList .../></View>` in `<RequireBaseSite>`. Hooks unchanged (component has none of its own).
  - `app/(portal)/(public)/catalog/[...categories].tsx` — wrapped the final `<FilteredProductList .../>` return in `<RequireBaseSite>`. Screen-level hooks (`useTranslations`, `useAppState`, `usePathname`, `useMemo`, `usePortalFilterStore`) stay above the boundary so they continue to obey Rules of Hooks across re-renders. The existing `if (!categoriesFromPath) return null;` early-return (which is itself gated on `selectedBaseSite`) is preserved; the boundary now formalizes the same intent at the JSX root.
  - `app/(portal)/(public)/product/[...productData].tsx` — the original `ProductScreen` function body holds `useEffect` mount-phase fetches (`getPortalProductDetails`, `getUserForId`). Wrapping the JSX alone would not have stopped those effects from firing because they live in the screen body, above the JSX. Split into outer `ProductScreen` (renders `<RequireBaseSite><ProductScreenContent /></RequireBaseSite>`) and inner `ProductScreenContent` (the full original body). When `selectedBaseSite` is undefined, `ProductScreenContent` is not mounted, so its `useEffect`s do not fire. Verbatim move of the original function body — no logic edits.
  - `app/(portal)/(public)/user/[...userData].tsx` — same split pattern as product detail. Outer `UserScreen` wraps `<UserScreenContent />`. Inner contains the original body with the `getUserForId` mount-phase fetch.
  - `fallback` was kept at the default `null` for all four screens — per Brief 17 the root overlay in `app/_layout.tsx` already covers the screen visually during non-`ready` states, so a per-screen placeholder would only be redundant.
- **D2 — `app/(portal)/(public)/blog/free-zone.tsx` also wrapped.** Per the brief's "if any DOES fire a base-site-dependent fetch, wrap it and note in 'For Mastermind.'" — this static-content screen renders `<ExtraProductsList extraSections={[MORE_FROM_FREE_CATEGORY]} />` which mounts `HorizontalExtraProductsView`, which fires `getPortalProducts(...)` (a `BACKEND_API` call carrying `X-Base-Site` / `X-Lang`) on mount. The narrowest correct boundary is around the `<ExtraProductsList>` block, leaving the marketing copy unaffected. The other four static-content screens (`about.tsx`, `pricing.tsx`, `privacy.tsx`, `terms.tsx`) were verified clean — no `BACKEND_API` calls on mount. `privacy.tsx` / `terms.tsx` `MarkdownViewer` does a `fetch()` to raw github (no `X-Base-Site`); `about.tsx` and `pricing.tsx` are pure UI. Flagged for Mastermind below.
- **Constraint 2 verification (returning-user path).** Confirmed in `src/components/context/AppContext.tsx` bootstrap success branch (lines 109-139): `setCodes(storedBaseSite.code, language.code)` (line 127) fires before `setStateWithStatusHook({ status: 'ready', ..., selectedBaseSite: storedBaseSite, selectedLanguage: language, ... })` (lines 132-139). Both `apiStore` codes and React `selectedBaseSite` are populated in the same logical step (same async branch, single render commit after `setStateWithStatusHook`). For a returning user with a stored base site, by the time any screen mounts the `<RequireBaseSite>` boundary sees `selectedBaseSite` defined and passes through immediately — no flash, no visible delay. Verified identically for the base-site-switch action (`setBaseSiteForCode` lines 187-220): `setCodes(site.code, lang.code)` precedes the `'ready'` flip.
- **D5 — `ProductList` mount effect guard.** `src/components/product/ProductList.tsx:101-103` changed from `useEffect(() => { onRefresh(); }, [fetchPage]);` to `useEffect(() => { if (!selectedLanguage) return; onRefresh(); }, [fetchPage, selectedLanguage]);`. `selectedLanguage` was already in scope from line 49 (`const { selectedLanguage } = useAppState();` — Brief 5). With the new `<RequireBaseSite>` boundary in place, this guard is a no-op in normal operation (the parent screen short-circuits before `ProductList` ever mounts in a `selectedBaseSite === undefined` state, and base-site + language are set together by `setCodes`). It earns its place as defense-in-depth per Brief 17 D5.
- **Cleanup 5a — `[DBG]` log block removed.** `src/lib/config/api.ts:44-53` (the `console.log('[DBG]', config.url, ...)` block inside the request interceptor) deleted. Grep `DBG` across `src/` and `app/` returns zero matches.
- **Cleanup 5b — misspelled TODO removed.** `src/components/internals/AppVersionConfigInit.tsx:29` (`// TODO: Translations PLUS after meintanence page do better check`) deleted. The TODO was vague, misspelled (`meintanence`), had no matching `issues.md` entry, and tracked no actionable scope — pure note-to-self that nobody will action. Cleaner to remove.

No navigator tree changes. Φ2's no-conditional-navigator constraint preserved — root `<Stack>`, portal `<Tabs>`, and public `<Stack>` all remain always-mounted. The boundary lives inside screen bodies. No `apiStore` / barrier changes (Brief 18 territory). No Cycle B chat-triangle changes (B chat territory).

## Files touched

- `src/components/context/RequireBaseSite.tsx` (new, +14 lines).
- `app/(portal)/(public)/index.tsx` (+4 / −2).
- `app/(portal)/(public)/catalog/[...categories].tsx` (+22 / −20 — wrap + import + indent shift on the JSX block).
- `app/(portal)/(public)/product/[...productData].tsx` (+9 / −1 — wrap + import + outer-function split).
- `app/(portal)/(public)/user/[...userData].tsx` (+9 / −1 — wrap + import + outer-function split).
- `app/(portal)/(public)/blog/free-zone.tsx` (+4 / −2 — wrap + import).
- `src/components/product/ProductList.tsx` (+2 / −1 — guard + dep array).
- `src/lib/config/api.ts` (−10 — `[DBG]` block removed).
- `src/components/internals/AppVersionConfigInit.tsx` (−2 — TODO removed).

## Tests

- Ran: `npx tsc --noEmit` — exit 0.
- Ran: `npm run lint` — 0 errors, 73 warnings (matches brief's ceiling of ≤73 exactly).
- Ran: `npm test` — `Test Files 7 passed (7) | Tests 109 passed (109)`. Meets the ≥109 floor exactly.
- New tests added: none. The `RequireBaseSite` component is ~14 lines of pure conditional render with a trivial branch; the screen-wrap work is integration-shaped (the right test is the manual smoke per Brief 17's smoke list). The `selectedLanguage` guard is two lines of defensive code. Considered adding a unit test for `RequireBaseSite` — see Part 4a "considered and rejected" — rejected as low-marginal-value.

## Cleanup performed

- Deleted the `console.log('[DBG]', …)` block in `api.ts:44-53` (per brief 5a).
- Deleted the misspelled `// TODO: Translations PLUS after meintanence page do better check` in `AppVersionConfigInit.tsx:29` (per brief 5b).
- No other cleanup required this session.

Adjacent observation noticed but NOT fixed (out of scope): `AppVersionConfigInit.tsx:27` declares `const ONE_MIN = 60 * 1000;` but `ONE_MIN` is unused. Flagged in "For Mastermind" Part 4b.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. Per Brief 17 / 18's pre-staged title, the close-out chat for the full Φ3 refactor (after the re-smoke) is the natural place for a single `decisions.md` entry summarizing Briefs 18+19. This Brief-19 session does not draft it.
- state.md: no change. Brief 19 is the final code step in the Φ3 in-flight trajectory; no backlog row movement until the post-Φ3 smoke clears.
- issues.md: no change. The cleanup items in this brief (DBG, TODO) were tracked in Brief 17 / 18 session summaries, not in `issues.md`.

## Obsoleted by this session

- Deleted in this session: the `[DBG]` `console.log` block in `api.ts`, and the misspelled stale `TODO` line in `AppVersionConfigInit.tsx`.
- Nothing left for follow-up from the Brief 17 D6 plan.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code introduced; no unused imports / variables / functions / files; no `console.log` introduced (the only one in scope was removed); no `TODO`/`FIXME` introduced; `tsc`, `lint`, `npm test` all green within the brief's ceilings.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one new observation flagged (unused `ONE_MIN` constant in `AppVersionConfigInit.tsx`) plus one carry-forward observation noted as resolved.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults) — the `<RequireBaseSite>` component is per-screen content gating, not a navigator constraint, so the "routes are reusable" intent is unaffected. Part 5 (session template) — followed below. Φ2's structural rule (no conditional navigator) preserved.

## Known gaps / TODOs

- The full Φ3 re-smoke checklist (Brief 17 D7) is the next step — this brief is the last code change before that. Smoke owner is Igor; smoke verifies all 12 cases listed in Brief 17.
- Behavior contract for the language-switch path on `ProductList`: see "For Mastermind" — adding `selectedLanguage` to the mount effect's deps introduces a subtle interaction with the dedicated language-change effect (lines 119-133). Both effects now react to language changes; the mount effect's `onRefresh()` (full reset to page 0) runs first and the language-change effect's `onRefreshCurrent()` (preserves pagination) is gated out by `loadingRef.current`. In normal operation this is consistent with the brief's intent (the brief says "current behavior maintained"), but the practical effect of `onRefresh` vs `onRefreshCurrent` on language switch is a user-visible change worth Mastermind's attention. Implemented as the brief specifies; flagged for transparency.

---

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):**
  - `src/components/context/RequireBaseSite.tsx` (~14 lines). Earned: the same boundary lives at four screens today and a fifth (`blog/free-zone.tsx`) was added in this session after verification. One wrapper component beats five inline guards; further screens that add base-site-dependent fetches in the future get the same one-line wrap. Verbatim per Brief 17 D2.
  - Outer-function split for `ProductScreen` / `UserScreen` (an extra function per file, ~4 lines each). Earned: the mount-phase `useEffect`s live in the screen body itself, above the JSX. Wrapping the JSX alone leaves those effects firing on every render where `selectedBaseSite` is undefined. The split is the minimal change that puts the effects below the `<RequireBaseSite>` boundary — they only mount, and only fire, when `selectedBaseSite` is defined.
- **Considered and rejected:**
  - **Hoisting `RequireBaseSite` into the public layout (`(portal)/(public)/_layout.tsx`)** so all six public screens get the boundary for free. Rejected — the brief is explicit ("Do NOT wrap the static-content screens") and a layout-level wrap would gate `about.tsx`, `pricing.tsx`, `privacy.tsx`, `terms.tsx` too, blocking them from rendering during `select-base-site`. Those screens have no base-site dependency and should remain visible. Per-screen wrapping is the correct shape.
  - **Conditionally rendering the inner `<Stack>` in `(portal)/(public)/_layout.tsx`** based on `selectedBaseSite`. Rejected — Φ2's decisions.md 2026-05-27 entry forbids conditionally rendering any navigator. The whole point of the per-screen boundary is to avoid this anti-pattern.
  - **Adding a unit test for `RequireBaseSite`.** Considered; rejected. The component is a single conditional render with two trivial branches (`!selectedBaseSite ? fallback : children`). A unit test would exercise the obvious without adding real coverage; the meaningful test is the manual smoke per Brief 17's checklist.
  - **Using a `LoadingOverlay` fallback on `ProductScreen` / `UserScreen` for the deep-link cold-start edge case** (a push notification taps directly into product/user detail before bootstrap finishes). Rejected for v1 — the root overlay in `app/_layout.tsx` already covers the screen visually during non-`ready` states, and the apiStore barrier (Brief 18 D1) holds any base-site-dependent request that slips through. `null` fallback is sufficient. If post-smoke reveals a visual gap on the deep-link path specifically, swap to `<LoadingOverlay />` per-screen later.
  - **Folding the `if (!selectedBaseSite) return null;` early-return in `catalog/[...categories].tsx` into the `RequireBaseSite` wrap by removing the existing `categoriesFromPath` null-guard.** Rejected — `categoriesFromPath` is null in two distinct cases (no base site OR pathname doesn't start with `/catalog`); only the first is covered by `RequireBaseSite`. The existing early-return is correct and stays.
  - **Removing the dedicated language-change effect in `ProductList.tsx` (lines 119-133) now that the mount effect re-fires on `selectedLanguage` change.** Considered; rejected — the dedicated effect calls `onRefreshCurrent()` (preserves pagination) while the mount effect calls `onRefresh()` (resets to page 0). They are not equivalent. The dedicated effect would be redundant in practice because `loadingRef.current` gates it out when the mount effect races first, but removing it would lock in the regression-on-language-switch behavior structurally. Left intact for now; flagged as a separate concern below.
- **Simplified or removed:**
  - Removed the `[DBG]` `console.log` block in `api.ts` (10 lines deleted).
  - Removed the stale misspelled `TODO` line in `AppVersionConfigInit.tsx` (1 line deleted).
  - Net: ~11 lines of dead-or-debug code removed; ~30 lines of new structural code added.

### Brief vs reality — ProductList language-switch behavior change

- **Brief says (Step 4 verification ask):** "Verify the dependency array change doesn't break the existing language-switch re-fetch behavior (adding `selectedLanguage` to deps means the effect re-runs on language change — confirm that's consistent with current behavior; the audit suggested language switches already trigger a refetch via a separate effect, so verify this doesn't cause a double-fetch)."
- **Code says:** Before this brief, `ProductList.tsx` had two effects relevant to language: the mount effect at lines 101-103 (deps `[fetchPage]`, calls `onRefresh()` — resets list, fetches page 0) and a dedicated language-change effect at lines 119-133 (deps `[selectedLanguage?.code]`, uses a `currentLanguageRef` to detect actual changes, calls `onRefreshCurrent()` — re-fetches already-loaded pages 0..N in parallel and merges, preserving pagination and scroll). On a language switch, only the dedicated effect fired — `onRefreshCurrent` preserved pagination.
- **After this brief:** the mount effect now also re-runs on language change (because `selectedLanguage` is in its deps). On a language switch, the mount effect's `onRefresh()` fires first; it sets `loadingRef.current = true` and starts fetching page 0. The dedicated language-change effect runs immediately after in the same React commit, sees `loadingRef.current === true`, and early-returns. Net: no double-fetch (per the brief's explicit concern), but the *behavior* on language switch shifts from `onRefreshCurrent` (preserve pagination) to `onRefresh` (reset to page 0). For a user scrolled to page 5 who switches language, the post-Brief-19 experience shows the list empty briefly then page 0 only, vs. the pre-Brief-19 experience that refetched pages 0-5 in place.
- **Why this matters:** the brief's hard rules include "No feature behavior change beyond the intended render gating." The language-switch fetch behavior is a feature behavior, not part of the render gate. The audit (Brief 17 D5) framed it as "current behavior maintained" — that holds for double-fetch prevention but not for pagination preservation.
- **Recommended resolution (Mastermind's call):**
  - **Option A — accept the regression.** Language switches are rare; losing scroll position once per switch is a small UX cost. Update Brief 17 / 19 commentary to reflect the actual semantic shift. The dedicated language-change effect at lines 119-133 becomes dead-code-on-paper (always early-returns) and can be deleted in a follow-up cleanup.
  - **Option B — keep the guard, change the body.** Replace `onRefresh()` with `onRefreshCurrent()` in the mount effect when the language change is the trigger. Requires distinguishing mount-vs-language-change inside the effect; non-trivial.
  - **Option C — keep the original mount effect deps (`[fetchPage]`).** Add the `selectedLanguage` guard without adding it to deps: `useEffect(() => { if (!selectedLanguage) return; onRefresh(); }, [fetchPage]);` — but eslint-react-hooks-exhaustive-deps will flag this. Could disable per-line, but defeats the linter's value.
  - **Option D — leave the dedicated language-change effect as the sole language responder, drop the dep add.** I.e., add the guard but keep deps at `[fetchPage]`. Same eslint concern as Option C. This preserves pre-Brief-19 behavior exactly. Closest to the brief's hard rule "No feature behavior change beyond the intended render gating."
  - I implemented as the brief specified (the literal Option A path). Flagging here so Mastermind can choose whether to revisit.

### Part 4b adjacent observations

1. **`ProductList.tsx:119-133` — dedicated language-change effect.** Likely now redundant in practice for normal language-switch paths, given the mount-effect dep change (see "Brief vs reality" above). Will always early-return because the mount effect wins the race. Severity: low (the early-return is benign; it's a structural smell, not a bug). **I did not fix this because it is out of scope for Brief 19 and is contingent on Mastermind's resolution of the regression question above.**
2. **`AppVersionConfigInit.tsx:27` — `const ONE_MIN = 60 * 1000;` is declared but unused.** Pre-existing dead code surfaced when I removed the adjacent TODO at line 29. Severity: low (cosmetic). **I did not fix this because pre-existing dead code outside this brief's scope was not in the brief's cleanup list; removing it would expand scope.**
3. **`blog/free-zone.tsx` quietly fires a base-site-dependent fetch.** The brief listed it as "static-content"; in fact `<ExtraProductsList extraSections={[MORE_FROM_FREE_CATEGORY]} />` → `<HorizontalExtraProductsView>` → `getPortalProducts(...)`. Wrapped in this session per the brief's "if any DOES fire a base-site-dependent fetch, wrap it" exception. Severity: low (a single unintentional fetch on the intro path was the cost before this wrap). **Documented here so the brief's static-screen list can be updated in Mastermind's next planning pass.**

### Confirmation requested in the brief

- **`useAppState` import path and field name** — Brief vs reality: clean. Import `@/components/context/AppContext`, hook name `useAppState`, field name `selectedBaseSite` — all match the live `AppContext.tsx` (lines 286-290 export the hook; type at line 24 has `selectedBaseSite?: BaseSiteDTO`).
- **`selectedLanguage` in scope in `ProductList`** — confirmed at line 49 (`const { selectedLanguage } = useAppState();`), exactly where the audit noted.
- **Constraint 2 ordering (returning user has `selectedBaseSite` before any screen mounts)** — confirmed. `setCodes(storedBaseSite.code, language.code)` at `AppContext.tsx:127` fires inside the bootstrap success branch BEFORE `setStateWithStatusHook` flips status to `'ready'` at lines 132-139 in the same synchronous follow-up. The state commit that brings `status === 'ready'` carries `selectedBaseSite: storedBaseSite` and `selectedLanguage: language` in the same render. `<RequireBaseSite>` will pass through on first render after `'ready'` — no flash.
- **No `[DBG]` references remain** — confirmed via grep across `src/` and `app/`: zero matches.

### Follow-up investigation: single-placement gate considered (and rejected, third time)

Igor asked mid-session whether a single Φ2-safe placement exists in place of the four per-screen wraps, to reduce author-memory fragility. Investigated all four candidates:

1. **Gate at `(portal)/(public)/_layout.tsx` wrapping `<Stack>`** — would mount/unmount the Stack on `selectedBaseSite` transitions. Even though in practice the transition is one-way per session (boot only), Φ2's rule per decisions.md 2026-05-27 is *"any future navigator must be always-mounted, not conditionally rendered"* — phrased as a constraint, not a guideline. Brief 17 D2 already considered and rejected this exact placement for the same reason. **Rejected.**
2. **Gate at `(portal)/_layout.tsx` wrapping `<Tabs>`** — same Φ2 violation. Also wrong scope (conflates base-site readiness with the secured-tabs auth guard). **Rejected.**
3. **Shared screen wrapper** — the four screens don't share a content shell. Creating one would be a real refactor that ends up looking like the per-screen wraps with different naming. **Rejected.**
4. **Per-screen wraps** — Φ2-safe (boundary inside screen body, navigators stay always-mounted). Fragile to author memory but no expo-router chokepoint exists to mechanically prevent the miss. The apiStore barrier (Brief 18 D1) catches misses as held requests per Brief 17 D4 (kept precisely as defense-in-depth). **Selected; matches Brief 17 D2's original choice.**

**The "one-way transition" question raised in the follow-up:** is a single undefined→defined mount/unmount of `<Stack>` the same hazard as repeated conditional rendering? Probably not in practice — the cycle in Φ2 closed because the gate's source-of-truth lived in the remount-reset path. Here `selectedBaseSite` lives in `AppContext` at the root layout, above any remount cascade. So theoretically a portal-layout-level gate might survive a one-time mount without looping. But Φ2's rule is conservative ("always-mounted, period") because the failure mode is opaque and reasoning about it is the same reasoning that gave us the loop in the first place. Per Igor's instruction "if uncertain, prefer the safest option even if it means more wraps" — kept per-screen.

No code change resulting from this follow-up. The per-screen wraps already implemented match the conclusion.

### Closure gate

No implicit config-file dependency. The next step is the full Φ3 re-smoke per Brief 17 D7. If smoke passes, the close-out chat opens with a single `decisions.md` entry covering Briefs 18 + 19 (pre-staged title from Brief 17 / 18: "Boot/HTTP layer refactor: codes-derived barrier, per-screen base-site render gate, api↔authStore cycle broken."). If smoke surfaces regressions, the regression bug(s) get their own session(s) before close-out.
