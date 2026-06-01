# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** expo-boot-redesign step 7b — the atomic cleanup: re-point all AppContext consumers to bootStore (Option A), delete the old boot path (AppContext, AppVersionConfigInit, RequireBaseSite, the i18n burst chain, the `_bootstrap` plumbing, diagnostic logs), and extract the hard/soft-update UI. After this session bootStore is the sole source of truth.

## Phase 0 (report-only) — outcome

Reported 0a (locale/setLocale) and 0b (hard/soft-update extraction) before touching code; Mastermind released Phases 1–6 with three locks, all applied:
- **0a lock:** `locale`/`setLocale` is dead → deleted with AppContext, no replacement (react-i18next drives consumer re-renders).
- **0b lock:** clean two-component + helper extraction; strings stay Serbian-hardcoded; placeholder store URL kept.
- **Gate 4 lock:** `runFreshnessGate` now calls `await i18n.changeLanguage(activeLang)` after the register step (was the gap that would have made runtime language-switch register bundles without switching the active language). This is **spec amendment #5** (see decisions.md draft).

## Implemented

- **Phase 1 — bootStore grew three pieces + the Gate-4 fix.** Added `latestVersion: string | null` slot (populated by Gate 2's version DTO, on both the force and optional/advance branches). Populated the existing `config` slot by fetching `getAppConfiguration()` inside Gate 1's success path (wrapped in `withGateTimeout`; failure/timeout → warn + `config: {}`, never maintenance — config is non-critical). Added `setLanguage(code)` action (resolve lang from `selectedBaseSite.allowedLanguages`, persist, set slot, re-run freshness). Added `await i18n.changeLanguage(activeLang)` to Gate 4's register step (idempotent; makes "active language loaded AND active" the gate's invariant). Exported `useConfiguration(key)` selector.
- **Phase 2 — re-pointed 25 consumers** from `useAppState`/`useAppActions` to bootStore selectors (the 26th consumer, RequireBaseSite, was deleted not re-pointed). `selectedBaseSite`→`useBootStore((s)=>s.selectedBaseSite)` (24 sites); `selectedLanguage`→`s.language` (ProductList, PortalConfigDialog); `baseSites`→`s.baseSiteOverviews` (PortalConfigDialog — verified the switcher needs only code/flag/label/domain, all on the overview DTO); `status`→`s.status` (product screen); `setBaseSiteForCode`→`pickBaseSite` (UserMenu, PortalConfigDialog, product screen); `setLanguageForCode`→`setLanguage` (PortalConfigDialog); `getConfiguration`→`useConfiguration` (BasicInfoProductDialog, owner product screen). Dropped the now-invalid `withLoading=false` 2nd arg at the product screen's `pickBaseSite` call (pickBaseSite takes only a code; the old withLoading is subsumed by the uniform `updating` status).
- **Phase 3 — deleted `RequireBaseSite`** and unwrapped its 5 portal screens (the portal-mounts-only-in-`ready` invariant makes `selectedBaseSite` non-null at render).
- **Phase 4 — extracted the update UI.** `HardUpdateScreen.tsx` (reads `latestVersion`, local `colorScheme`, rendered by the `_layout` overlay's new `update-required` branch). `SoftUpdateModal.tsx` (reads `softUpdate`+`latestVersion`, self-gates via the dismissal helper, mounted at layout level). `softUpdateDismissal.ts` (the 24h-per-version memory ported verbatim — same `dismissed_optional_update_version_data` key + shape so existing dismissals carry forward; dropped the dead `ONE_MIN` constant and the redundant `if-if` nesting).
- **Phase 5 — deletions.** `AppVersionConfigInit.tsx`, `AppContext.tsx`, `i18n/i18n.ts`, `i18n/loader.ts`, `i18n/loadNamespaces.ts` deleted; the two now-empty dirs removed. Removed `<AppContextProvider>`+`<AppVersionConfigInit>` wrappers, `setLocale` state, and the `[MOUNT-EFFECT]`/`[OVERLAY]` logs from `_layout`. Removed the `[BOOT-START]` log from `bootStore.start()`. Removed the `_bootstrap?: boolean` type augmentation from `api.ts` and every `_bootstrap: true` call-site flag (6 services). Rewrote the stale "apiStore bootstrap barrier" comment in `versionsService.ts`. (The duplicate maintenance poll died with AppContext.)
- **Phase 6 — verification** below.

## Files touched

**Created (3):**
- `src/components/init/HardUpdateScreen.tsx`
- `src/components/init/SoftUpdateModal.tsx`
- `src/lib/store/softUpdateDismissal.ts`

**Deleted (6 files + 2 empty dirs):**
- `src/components/context/AppContext.tsx`, `src/components/context/RequireBaseSite.tsx` (+ dir `src/components/context/`)
- `src/components/internals/AppVersionConfigInit.tsx` (+ dir `src/components/internals/`)
- `src/i18n/i18n.ts`, `src/i18n/loader.ts`, `src/i18n/loadNamespaces.ts`

**Modified — boot infra:**
- `src/lib/store/bootStore.ts` (latestVersion slot, config population, setLanguage, Gate-4 changeLanguage, useConfiguration export, [BOOT-START] removed, 3 stale comments tidied)
- `src/lib/store/bootStore.test.ts` (i18n.changeLanguage mock + configurationService mock; 2 Invariant-3 tests updated for the now-gate-owned config slot; +9 new tests)
- `app/_layout.tsx` (teardown + new component wiring + log removal)
- `src/lib/config/api.ts` (`_bootstrap` type augmentation removed)
- `src/lib/init/versionsService.ts`, `src/lib/init/baseSitesService.ts`, `src/lib/services/maintenanceService.tsx`, `src/lib/services/configurationService.tsx`, `src/lib/services/appVersionConfigService.tsx` (`_bootstrap` flags removed)
- `src/i18n/fetchNamespace.ts` (3 debug `console.log` removed — adjacent boot-path cleanup, see Cleanup)

**Modified — 25 re-pointed consumers:**
- Simple `selectedBaseSite` (19): `src/components/filters/CurrencySelector.tsx`, `src/components/ConsumerProtectionBanner.tsx`, `src/components/SearchInput.tsx`, `src/components/filters/ProductOrder.tsx`, `src/components/filters/RegionCityFilter.tsx`, `src/components/navigation/CategoryNavigation.tsx`, `src/components/navigation/Footer.tsx`, `src/components/navigation/Filters.tsx`, `src/components/basic/CategorySelector.tsx`, `src/components/dialog/dialogs/CurrencySelectionDialog.tsx`, `src/components/basic/CitySelector.tsx`, `src/components/navigation/TopBar.tsx`, `src/components/product/ProductBreadcrumb.tsx`, `src/components/dialog/dialogs/FiltersDialog.tsx`, `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx`, `src/components/dialog/dialogs/PreviewProductDialog.tsx`, `app/(portal)/_layout.tsx`, `app/owner/_layout.tsx`, `app/(portal)/(public)/catalog/[...categories].tsx`
- `src/components/product/ProductList.tsx` (selectedLanguage→language)
- Complex: `src/components/dialog/dialogs/PortalConfigDialog.tsx`, `src/components/user/UserMenu.tsx`, `app/(portal)/(public)/product/[...productData].tsx`, `src/components/dialog/dialogs/product-creation/BasicInfoProductDialog.tsx`, `app/owner/dashboard/products/[productId].tsx`

**Modified — RequireBaseSite unwrap only (3, beyond catalog/product already listed):**
- `app/(portal)/(public)/index.tsx`, `app/(portal)/(public)/user/[...userData].tsx`, `app/(portal)/(public)/blog/free-zone.tsx`

(All boot-redesign source files remain untracked on `new-expo-dev` — the feature is uncommitted per state.md. Igor commits.)

## Tests

- `npx vitest run` → **185 passed** (11 files, 0 failed). Up from 176: +9 new bootStore tests (3 Gate-1 config tests, 2 latestVersion-capture tests, 1 Gate-4 changeLanguage test, 3 setLanguage tests). 2 existing Invariant-3 tests updated (not weakened) to reflect config now being Gate-1-owned (re-populated, not "no gate owns it").
- `npx tsc --noEmit` → clean.
- `npx expo lint` → **0 errors, 70 warnings**, all pre-existing or same-idiom-as-existing. The only new line, `bootStore.ts:496` (`i18n.changeLanguage` named-as-default-member), is the identical class to the pre-existing `i18n.use` warning (line 468) and matches the file's established i18n-default-import idiom.
- `npx expo-doctor` — not run; no dependency changes this session (package.json/lock untouched).
- Did NOT run `npx expo start` (long-running dev server). Live device regression steps documented below for Igor.

## Cleanup performed

- Removed the `[BOOT-START]` log (`bootStore.start()`), `[MOUNT-EFFECT]`/`[OVERLAY]` logs (`_layout.tsx`).
- Removed 3 debug `console.log` in `src/i18n/fetchNamespace.ts` (logged full payloads/errors on every namespace fetch — the central Gate-4 fetcher). Not a primary brief target and not in my re-point set, but this is the feature-close whose explicit goal (spec Part 9) is a diagnostic-free boot path, and conventions Part 4 forbids `console.log`; the removal is zero-risk (the gate already handles the swallowed-undefined return). Switched `catch (error)` → `catch {} ` so no unused binding is introduced.
- Deleted the dead `ONE_MIN` constant and the redundant `if (res.optionalUpdate) { if (res.optionalUpdate)` nesting when porting the soft-update dismissal logic.
- Deleted 6 obsolete files + 2 now-empty directories; removed all `_bootstrap` plumbing; rewrote the stale apiStore-barrier comment in versionsService.ts; tidied 3 bootStore comments that referenced the deleted AppContext/initI18n.
- No commented-out code, no `TODO`/`FIXME` added.

## Config-file impact

- **decisions.md:** new feature-close entry owed — drafted in "For Mastermind." Includes the 5 spec amendments (the original 4 + the new Gate-4 register-step amendment #5).
- **state.md:** edits owed (Φ3 boot-loading → resolved; Version-checksums backlog row → mobile-stable; Risk Watch rows) — drafted in "For Mastermind."
- **issues.md:** edits owed (`[BOOT]` interceptor obligation → closed; always-mounted-Stack pre-bootstrap entry → updated) — drafted in "For Mastermind." Also one new low-severity entry (fetchNamespace had debug logs — now fixed; logging only for the record) and one optional `.env.*` paths-filter nit carry-over.
- **conventions.md:** no change.
- **Closure gate:** no unstated config-file dependency. The Version-checksums backlog row flip is the only adoption-status change this session enables, and it is drafted below.

## Obsoleted by this session

- `AppContext.tsx` + its provider, the `useAppState`/`useAppActions` hooks, `setBaseSiteForCode`/`setLanguageForCode`/`getConfiguration`/`reBootstrap`, the `Promise.all` bootstrap, the catch-all-to-maintenance, the duplicate maintenance `setInterval` — **deleted**.
- `AppVersionConfigInit.tsx` (its soft/hard classification was already Gate 2's; its UI is now HardUpdateScreen + SoftUpdateModal) — **deleted**.
- `RequireBaseSite.tsx` (replaced by the portal-mounts-only-in-`ready` invariant) — **deleted**.
- `i18n/i18n.ts` (`initI18n`), `i18n/loader.ts` (`loadTranslations`, `VERSION_KEY`, commented scaffolding), `i18n/loadNamespaces.ts` (`loadAllNamespaces`) — the burst chain, **deleted**. Gate 4 owns i18n init/registration.
- The `_bootstrap` flag + its type augmentation (barrier is long gone) — **deleted**.
- The `[BOOT-START]`/`[MOUNT-EFFECT]`/`[OVERLAY]` diagnostics — **deleted**.

## Conventions check

- **Part 4 (cleanliness):** confirmed. Net-subtractive: 6 files + a whole render-gate competing tree deleted, all named diagnostics + the fetchNamespace debug logs removed, no commented-out code, no TODOs, tsc/lint/tests green.
- **Part 4a (simplicity):** structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** flagged in "For Mastermind."
- **Part 6 (translations):** confirmed — no new keys; the extracted update screens keep the existing Serbian-hardcoded strings (per the 0b lock); backend-seeded keys unaffected; Gate-4 `changeLanguage` now correctly activates the loaded language.
- **Spec Part 1 invariants + structural defense:** re-confirmed in "For Mastermind."

## Known gaps / TODOs

- No `TODO`/`FIXME` added.
- The store deep-link in HardUpdateScreen/SoftUpdateModal remains the placeholder `https://memento-tech.com` — the real App Store/Play Store link is the spec's separate pre-launch task (Part 3), out of scope here.
- A stray `src/i18n/.i18n.ts.swp` vim swap file exists (Igor's editor artifact for the now-deleted `i18n.ts`); left untouched to avoid disrupting an open editor — see Part 4b.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** `latestVersion` slot — required by the hard/soft-update screens, one field. `setLanguage(code)` action — the runtime language-switch primitive PortalConfigDialog needs; reuses `runFreshnessGate` rather than duplicating fetch/register logic. `useConfiguration(key)` selector — one-liner replacing the deleted `getConfiguration` action; named in the spec. `softUpdateDismissal.ts` helper — isolates AsyncStorage details so SoftUpdateModal stays a renderer (two callers: the show-probe and the dismiss action). HardUpdateScreen/SoftUpdateModal — direct extractions, no redesign.
  - **Considered and rejected:** a bootStore `locale` slot to replace `setLocale` (rejected — 0a proved it dead, react-i18next handles re-renders); a `bootStore.selectors.ts` file (rejected — one selector, co-located with the store per existing style); subscribing to the whole `config` map in the 2 regex callers vs 6 `useConfiguration` calls (kept the spec's per-key selector — simpler at the call site, matches the brief).
  - **Simplified or removed:** the entire AppContext status machine + duplicate maintenance poll; the 19-namespace-aware `initI18n`/`loader`/`loadNamespaces` chain; the `_bootstrap` flag plumbing; the dead `ONE_MIN` + redundant `if-if` in the dismissal logic; 3 debug logs in fetchNamespace; 3 stale comments.
- **Three invariants — re-confirmed:**
  1. **One effect, empty deps:** `app/_layout.tsx:39` is the single boot effect (`[]` deps → `start()`). The other effect (line 47) is the Android nav-bar listener — touches no machine state. No boot-path effect lists `status`/`selectedBaseSite`/codes.
  2. **Machine writes status, view reads it:** grep-confirmed — the only `status` writes are bootStore's two `set({ status })` + the 4 `toX` helpers; zero status writes anywhere else in src/ or app/. The new components and re-pointed consumers only *read* via selectors.
  3. **Re-entry destroys nothing:** `start()` sets only `status:'booting'`, clears no slot. The Invariant-3 tests (incl. the real-Gate-4 re-entry test) pass. Note: `config` is now re-fetched each Gate-1 pass (a fresh fetch, not a clear) — non-destructive; the Invariant-3 tests were updated to assert the resolved slots (base site, language, softUpdate) survive while config is re-populated to the same value.
- **Structural defense — re-confirmed:** `bootStore.ts` and its leaf modules (`bootFreshness`, `bootGate`, `checksumStorage`, `baseSitesService`, `versionsService`, `fetchNamespace`, `maintenanceService`, `appVersionConfigService`, `configurationService`) and the new `softUpdateDismissal.ts` (AsyncStorage only) import **no** authStore and **no** chat store. Grep clean (only the doc-comment mentions in bootStore). `api.ts`'s lazy `getState()` edge to bootStore is unchanged.
- **Boot-loop regression test — manual steps for Igor (spec Part 8 / brief 6c), run on a real device after commit:**
  1. Fresh install / cleared storage → intro picker → pick a base site → header + product list render with no restart.
  2. Cold-restart with a stored site → reaches `ready` with zero translation/catalog fetches (checksum payoff).
  3. Cold-restart with maintenance ON → maintenance screen.
  4. Toggle maintenance OFF → returns to `ready` within ~5s WITHOUT clearing the stored base site and WITHOUT extra boot passes. **(the named boot-loop regression test)**
  5. Language switch via PortalConfigDialog → new language registers AND activates (changeLanguage), components re-render in the new strings.
  6. Base-site switch via PortalConfigDialog/UserMenu → `pickBaseSite` runs, brief `updating` splash, new site's content renders.
- **Adjacent observations (Part 4b):**
  1. `src/i18n/fetchNamespace.ts` had 3 debug `console.log` (full payload/error on every fetch) — **fixed this session** (see Cleanup); flagging for the record since it was outside the brief's explicit target list. Severity was low/medium (noise, no user impact).
  2. `src/i18n/.i18n.ts.swp` — stray vim swap file (orphaned now that `i18n.ts` is deleted). Severity: low (editor artifact, typically gitignored). Not fixed — it's Igor's open-editor state; deleting it could trigger a vim recovery prompt. Recommend Igor closes the buffer / removes it.
  3. `app/(portal)/(public)/product/[...productData].tsx:128` retains a `console.error('Failed to load product', error)` — pre-existing caught-error log, not a debug diagnostic; left as-is (out of scope, arguably fits an error-logging strategy). Severity: low.

### decisions.md entry draft (feature-close)

> **2026-05-29 — `oglasino-expo` expo-boot-redesign complete (Φ3 boot-loading resolved).** The gate state machine (`bootStore`) replaces `AppContext.bootstrap` as the sole boot authority and sole source of truth for `selectedBaseSite`/`language`/`config`/`status`. Version-checksum freshness (Gate 4, `/versions`) replaces the 19–20 namespace cold-start burst; a returning user with current content fetches zero namespaces/catalog. Per-platform floor/ceiling version model drives Gate 2 (force/soft/neither). Step 7b removed the entire old path: `AppContext`, `AppVersionConfigInit` (UI now `HardUpdateScreen` + `SoftUpdateModal`, driven by `bootStore.latestVersion`/`softUpdate`), `RequireBaseSite` (subsumed by the portal-mounts-only-in-`ready` invariant), the `initI18n`/`loadTranslations`/`loadAllNamespaces` chain (Gate 4 owns i18n), the `_bootstrap` flag + type augmentation, and all boot diagnostics. All 26 AppContext consumers re-pointed to bootStore selectors. Three invariants + structural defense hold. **Spec amendments owed (5):** (1) Part 3 "dev seed" → "safe-default-all-envs seed" (B2); (2) Part 5 `translations_<NS>` → `translations_<NS>_<lang>` payload key; (3) Part 4 namespace count is 20 (mobile) vs 22 (backend) — ADMIN_PAGES + BACKEND_TRANSLATIONS deliberately omitted; (4) Part 8 step-6 test wording reconciled with the "register the full in-memory map" logic; (5) **NEW — Gate 4 register step calls `i18n.changeLanguage(activeLang)`** after registering bundles, so the active language actually switches on re-entry (required for the runtime language switcher; idempotent on first boot). Forward pointer: `future/version-checksums-per-language.md` (per-NS-per-language checksums). Closes the issues.md `[BOOT]` interceptor obligation and the "always-mounted Stack exposes pre-bootstrap API calls" entry (now governed by bootStore slot population + the gates, not a queue/barrier).

### state.md edit drafts

- **Φ3 / Expo performance foundation:** flip boot-loading status → resolved (cold-start completes cleanly; the named boot-loop regression test is wired and documented for Igor's device run). Once Igor's manual regression passes, the feature flips `shipped`.
- **Expo backlog `Version checksums` row → mobile-stable** (mobile now consumes `/versions` via Gate 4; freshness is checksum-driven).
- **Risk Watch:** close the cold-start-hang row (resolved by the redesign). Open/track separately a "ceiling-hook live smoke" item — verify the EAS post-build ceiling write once Igor's first preview build runs (backend B2 concern; not exercisable from mobile until a real build).

### issues.md edit drafts

- `[BOOT]` interceptor log obligation (2026-05-28) → **closed** (the interceptor log was removed in boot-redesign-9; all remaining boot diagnostics removed this session).
- "Always-mounted `<Stack>` exposes pre-bootstrap API calls" → **update/close**: the apiStore barrier no longer exists; pre-`ready` requests are governed by bootStore slot population (the interceptor reads `selectedBaseSite`/`language` from bootStore and omits the headers until Gate 3 resolves them) + the sequential gates. The redesign's invariants govern this case.
- **New (low):** `app/(portal)/(public)/product/[...productData].tsx:128` keeps a `console.error` for a caught product-load failure — confirm whether it fits the logging strategy or should route through `serviceLog`. (Optional.)
- **Carry-over (low):** the `.env.*` paths-filter nit parked in 7a — route here if still relevant.

- Nothing else flagged.
