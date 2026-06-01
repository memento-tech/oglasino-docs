# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** expo-boot-redesign step 7d, Phase 0 (read-only) — audit Issue 1 (i18n-timing for `selectedBaseSite` consumers reading outside the Stack gate's protection). Answer Questions A–D with file:line evidence, then STOP and report before any Phase 1 fix.

## Phase 0 outcome (one line)

**Issue 1 is NOT systemic / NOT structural-above-the-gate, and there are NO outside-Stack consumers that need guarding.** It is an *inside-Stack `updating`-sub-window* timing hazard confined to the portal header chrome, which Igor already fixed — but his patch is in **`app/(portal)/_layout.tsx`**, not `CategoryNavigation.tsx` as the brief assumes. Details A–D below. **Stopped after Phase 0 per the brief's STOP gate; Phase 1 not started.**

---

## A — Is `app/(portal)/_layout.tsx` inside or outside the gated Stack?

**Inside (it is a child route of the gated root `<Stack>`); it unmounts together with that Stack.**

The gated region in `app/_layout.tsx` (verbatim, `:93-97`):

```tsx
{(bootStatus === 'ready' || bootStatus === 'updating') && (
  <View className="flex-1">
    <Stack screenOptions={{ headerShown: false }} />
  </View>
)}
```

`app/(portal)/_layout.tsx` defines the layout for the `(portal)` route segment, which is a **screen of this root `<Stack>`**. This is plain React conditional rendering, not an expo-router subtlety: when `bootStatus ∉ {ready, updating}` the `<Stack>` element is absent from the React tree, so none of its route screens — including `(portal)/_layout.tsx` and its header chrome — can exist. The whole `(portal)` tree mounts/unmounts with the root Stack.

This is exactly the 7c design intent (summary `boot-redesign-12`, spec Part 8 step 7: "portal tree mounts ONLY in ready/updating"). Before 7c the Stack was always-mounted and the portal tree ran under the overlay (summary `boot-redesign-11` §0a/0e); 7c gated it.

**Conclusion:** the chrome cannot mount before the root Stack mounts → Issue 1 does NOT hit the brief's "structural — layout mounts above the gate → STOP" branch (1.2 case 3). The remaining hazard is *within* the Stack-mounted window — specifically the `updating` sub-state.

## B — Which exact symptom, and the mechanism

**Symptom (code-supported):** raw translation keys in the header chrome — most visibly `CategoryNavigation`'s category labels `t(cat.labelKey)` (`CategoryNavigation.tsx:58`), the "suggest category" button `tHeader('suggest.category.button.label')` (`:65`), and `ConsumerProtectionBanner`'s `tCommon('caution.diff.base.site.description')` (`ConsumerProtectionBanner.tsx:35`).

**The ordering hazard (the real mechanism):**

1. `pickBaseSite` sets the base site **before** any translation work: `set({ selectedBaseSite: fullDto, language: lang })` at `bootStore.ts:280`, then `await runFreshnessGate()` at `:285`.
2. `runFreshnessGate` enters `updating` **up front** — `get().toUpdating()` at `:343` — then does all network work (`/versions`, catalog + per-namespace refetch, `:347-459`).
3. i18n registration + `i18n.changeLanguage(activeLang)` happen only at **step 6, `:467-496`**, immediately before `set({ status: 'ready' })` at `:501`.

So for the entire `updating` window: `selectedBaseSite` is set, but i18n is **not yet initialized** (fresh install) / **not yet switched** (language change). The old `(portal)/_layout.tsx` gate keyed the chrome on `selectedBaseSite` (truthy during `updating`) — so the chrome mounted into that window and `t(...)` returned raw keys. This contradicts the bootStore's own documented invariant ("register the active language into i18n BEFORE 'ready' so the portal never paints raw keys", `bootStore.ts:312-314`, audit §6.2) — that invariant assumes the chrome renders only at `ready`.

**Occlusion caveat (important, stated honestly):** the `updating` splash overlay in `app/_layout.tsx:114-119` is an opaque full-screen `<View className="absolute inset-0 ... bg-background">` painted after the Stack, so it nominally **occludes** the `updating` window itself. The chrome lives under it (inside `SafeAreaView`). The visible artifact is therefore best explained as the **`updating → ready` boundary frame**: at `:501` the status flips to `ready` synchronously (overlay unmounts, `:114`), while the chrome's already-mounted `useTranslation` hooks (mounted during `updating` when i18n was uninitialized) only repaint resolved strings when react-i18next's `initialized`/`languageChanged` event propagates — which can land a frame later, revealing raw keys as the overlay lifts. This is most pronounced on **cold-boot-after-pick** (i18n uninitialized → initialized, a bigger visual jump than old-lang → new-lang). I could not pin the exact perceived frame from code alone; recommend Igor confirm whether he saw it on the fresh-install pick, a cold-restart, or a runtime language switch. **The fix is correct regardless** (see D): mounting the chrome only at `ready` makes its first render read already-initialized i18n synchronously, removing the dependence on event-propagation timing entirely.

**Which of the brief's options:** raw keys, during the cold-boot transition after picking a base site (and/or the language-switch transient) — **not** the steady-state "after reaching ready" serious case (once propagation completes the chrome is correct).

## C — `selectedBaseSite` × `useTranslations` consumers, mapped by mount location

Exhaustive over `src/` + `app/` (22 files read `selectedBaseSite`, all now via `useBootStore`; the post-pick audit's AppContext list is superseded). `useTranslations` cross-checked per file.

### Inside the gated Stack (mount only in `ready`/`updating` — protected by the 7c gate)

| File | useTranslations? | Mount chain |
|---|---|---|
| `app/(portal)/_layout.tsx` (chrome wrapper) | indirectly (its children) | root Stack → `(portal)` |
| `src/components/navigation/CategoryNavigation.tsx` | **yes** (COMMON_SYSTEM, HEADER) | chrome (now `ready`-gated, see D) |
| `src/components/ConsumerProtectionBanner.tsx` | **yes** (COMMON) | chrome |
| `src/components/navigation/TopBar.tsx` | no | chrome |
| `src/components/SearchInput.tsx` | **yes** (COMMON_SYSTEM, HEADER, COMMON) | ← TopBar (chrome) |
| `src/components/user/UserMenu.tsx` | **yes** (BUTTONS) | ← BottomBar (tab bar) |
| `src/components/navigation/Footer.tsx` | **yes** (COMMON_SYSTEM, FOOTER, PAGING) | ← portal screens (about/privacy/free-zone/product/terms/pricing) + ProductList |
| `src/components/navigation/Filters.tsx` | **yes** (COMMON_SYSTEM) | portal filter UI |
| `src/components/product/ProductBreadcrumb.tsx` | **yes** (COMMON_SYSTEM) | product screen |
| `app/(portal)/(public)/catalog/[...categories].tsx` | **yes** | portal screen |
| `app/(portal)/(public)/product/[...productData].tsx` | **yes** (+ reads `status`) | portal screen |
| `app/owner/_layout.tsx` | **yes** (INTRO) | root Stack → `owner` |
| `app/owner/dashboard/products/[productId].tsx` | **yes** | owner screen |
| `src/components/filters/CurrencySelector.tsx` | no | ← PriceFilter ← Filters |

### Mounted in BOTH an inside-Stack chain and a dialog

| File | useTranslations? | Mount chains |
|---|---|---|
| `src/components/filters/RegionCityFilter.tsx` | **yes** (COMMON_SYSTEM) | `Filters` (inside-Stack) **and** `FiltersDialog` (dialog) |
| `src/components/filters/ProductOrder.tsx` | **yes** (COMMON_SYSTEM) | `Filters` (inside-Stack) **and** `FiltersDialog` (dialog) |
| `src/components/basic/CitySelector.tsx` | **yes** | owner screens (inside-Stack) **and** `BasicInfoProductDialog` (dialog) |
| `src/components/basic/CategorySelector.tsx` | **yes** | owner screens (inside-Stack) **and** `BasicInfoProductDialog` (dialog) |

### Outside the gated Stack — dialogs (rendered by `DialogManager` in a RN `<Modal overFullScreen>` at `app/_layout.tsx:99`)

| File | useTranslations? |
|---|---|
| `src/components/dialog/dialogs/PortalConfigDialog.tsx` (reads `selectedBaseSite`, `selectedLanguage`-equivalent, `baseSites`) | **yes** (DIALOG, COMMON, HEADER) |
| `src/components/dialog/dialogs/CurrencySelectionDialog.tsx` | **yes** (DIALOG, INPUT) |
| `src/components/dialog/dialogs/FiltersDialog.tsx` | **yes** (COMMON_SYSTEM, DIALOG, DASHBOARD_PAGES) |
| `src/components/dialog/dialogs/PreviewProductDialog.tsx` | **yes** (COMMON_SYSTEM, DIALOG) |
| `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx` | **yes** (DIALOG) |

### Outside the gated Stack — root-layout persistent (`app/_layout.tsx` direct children)

`AppInit`, `MaintenancePollInit`, `SoftUpdateModal`, `LoadingOverlay`, `BaseSiteSelector` (overlay picker), `HardUpdateScreen`. **None read `selectedBaseSite`.** Notably `BaseSiteSelector.tsx` — the only one of these that renders text pre-`ready` — reads `baseSiteOverviews`/`pickBaseSite` (`:37-38`), NOT `selectedBaseSite`, and uses its **own** `tIntro` closure backed by a direct `fetchNamespace(INTRO,'sr')` + `INTRO_FALLBACK` (`:21-69`), NOT the i18n singleton. It is immune to the registration-timing hazard by design.

**Verdict on the load-bearing split:** the only structurally-outside-Stack `selectedBaseSite` + `useTranslations` consumers are the **dialogs**. They mount only when `currentDialogId` is set (`DialogManager.tsx:63`), which happens only via user gesture from ready-state portal UI — so they **cannot open before `ready`**. The single transient where `status` leaves `ready` while a dialog is open is a language/base-site switch initiated from `PortalConfigDialog` (`setLanguage`/`pickBaseSite` → `updating`); during that window i18n still holds the **prior** language fully (no raw keys — the new bundles + `changeLanguage` land together at Gate 4 step 6 before `ready`), and the dialog repaints to the new language at `ready`. **No outside-Stack consumer renders raw keys; none needs a `bootStatus === 'ready'` guard.**

## D — Igor's actual patch (Brief vs reality)

The brief assumes Igor patched `CategoryNavigation.tsx`. **He did not.** `CategoryNavigation.tsx` has no `bootStatus` guard — it still gates only on `if (!selectedBaseSite) return null;` (`:32`); its only diff from baseline is the 7b AppContext→bootStore migration (`useAppContext()` → `useBootStore((s) => s.selectedBaseSite)`).

Igor's patch is in **`app/(portal)/_layout.tsx:16`** — he changed the chrome-block gate from `{selectedBaseSite && (...)}` (the value summary `boot-redesign-11` §0e recorded at 10:42, and no session after 11 touched this file) to:

```tsx
const bootStatus = useBootStore((s) => s.status);   // :12
...
{bootStatus === 'ready' && (                          // :16
  <View>
    <ConsumerProtectionBanner />
    <TopBar useFilterStore={usePortalFilterStore}
      getAutocompleteSuggestions={getPortalAutocompleteSuggestions} />
    <CategoryNavigation />
  </View>
)}
```

This is the correct fix shape and is at the **right altitude**: one gate at the layout level covers all three chrome components (CategoryNavigation, TopBar, ConsumerProtectionBanner) at once — matching the Part 4a rationale Mastermind used in the 2026-05-27 decision (one `selectedBaseSite` guard over three callers rather than three per-component guards). Because the chrome now mounts only at `ready`, its first render reads an already-initialized i18n with the active language set (Gate 4 step 6 ran before `:501`), so strings resolve synchronously on first paint — the boundary-frame hazard (B) is gone.

**Recommendation:** accept Igor's `(portal)/_layout.tsx` patch as the template/fix for Issue 1. The fix template the brief describes ("gate on `bootStatus === 'ready'` via `useBootStore((s) => s.status)`") is exactly what Igor applied — just at the layout, not the leaf. No other file needs the guard (C).

---

## Files touched

- None. Phase 0 is read-only.

## Tests

- None run (read-only). No code changed.

## Cleanup performed

- None needed — read-only audit; no edits, no imports, no logging introduced.

## Config-file impact

- **conventions.md / decisions.md / state.md / issues.md:** no change this session. Phase 0 produced findings only. Any config edits (e.g. the feature-close `decisions.md` entry, `state.md` flip to `shipped`, Expo-backlog version-checksums row) belong to the feature-close pass after Phase 1, and are Docs/QA's to apply.
- **Closure gate:** no unstated config-file dependency introduced by this read-only phase.

## Obsoleted by this session

- Nothing in code (read-only). For the record: `.agent/audit-post-pick-consumers.md` (09:15) is now stale on *which store* consumers read — it maps `AppContext` (`useAppState`/`useAppActions`), but 7b migrated every consumer to `useBootStore` and AppContext is torn out (zero live references; only a doc-comment in `bootStore.ts`). Its file list and per-file `useTranslations` facts remain useful; its store attribution does not. Flagged, not edited (not my file to amend).

## Conventions check

- **Part 4 (cleanliness):** N/A — no code changed.
- **Part 4a (simplicity):** structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** three flagged in "For Mastermind."
- **Part 6 (translations):** N/A — no keys touched.
- **Spec Part 1 invariants + structural defense:** unaffected (read-only); re-confirmed incidentally — `status` written only inside `bootStore.ts`; `bootStore` imports no authStore/chat-store.

## Known gaps / TODOs

- The exact perceived frame/scenario of the raw-key flash (cold-boot pick vs cold-restart vs language switch) is not pinned from code alone — recommend Igor confirm. Does not change the fix (B/D). No `TODO`/`FIXME` added.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** nothing — Phase 0 is read-only.
  - **Considered and rejected:** running the consumer enumeration as a multi-agent workflow (rejected — a precise, self-corroborating read sweep over ~25 files where every quote must be verified by hand; following the `boot-redesign-11` precedent, fan-out adds no precision here). Spinning an Explore agent for render-chain corroboration (rejected — direct `rg` over importers was sufficient and cheaper).
  - **Simplified or removed:** nothing — read-only.

- **Direct answers for the Phase-1 release decision:**
  1. **Question A:** `(portal)/_layout.tsx` is *inside* the gated Stack and unmounts with it. Issue 1 is **not** structural-above-the-gate → does not trigger the brief's STOP-case-3.
  2. **Question C verdict:** issue 1 is an **inside-Stack `updating`-sub-window** hazard, confined to the portal header chrome. There are **no outside-Stack consumers needing a guard** (the only outside-Stack consumers are dialogs, which can't open pre-`ready` and don't render raw keys during the language-switch transient). This is the brief's 1.2 **case 2** ("only inside-Stack consumers / one-off already fixed locally — document and don't touch other files").
  3. **Question D / Brief vs reality:** Igor's patch is in **`app/(portal)/_layout.tsx:16`** (`{bootStatus === 'ready' && ...}`), not `CategoryNavigation.tsx`. Same fix shape, right altitude (covers all three chrome components). Recommend accepting it as-is; `CategoryNavigation.tsx` needs no further change and reads cleanly.
  - **Net:** if Phase 1 is released, the only code work is **Issue 2** (`PortalConfigDialog` empty base-site list — `ensureBaseSiteOverviews` on bootStore + dialog mount-fetch + loading state + tests). Issue 1 needs **no new code** beyond accepting Igor's existing `(portal)/_layout.tsx` patch.

- **Brief vs reality (formal):**
  1. **Igor's patch location.** Brief says (Issue 1, Question D): Igor patched `CategoryNavigation` / "quote his patched `CategoryNavigation.tsx`." Code says: `CategoryNavigation.tsx` has no `bootStatus` guard; the guard is in `app/(portal)/_layout.tsx:16` (chrome block gated `bootStatus === 'ready'`). Why it matters: Phase 1's "verify Igor's patched CategoryNavigation reads cleanly" should instead verify `(portal)/_layout.tsx`. Resolution: treat `(portal)/_layout.tsx:16` as the Issue-1 fix; no per-leaf guard needed.
  2. **Occlusion vs the reported symptom.** Brief frames Issue 1 as a plain "renders with raw keys" during a window. Code says the `updating` window is occluded by the opaque splash overlay (`app/_layout.tsx:114-119`); the code-supported visible artifact is the `updating→ready` boundary frame (react-i18next event-propagation lag on chrome hooks mounted while i18n was uninitialized). Why it matters: only affects the *explanation* and the manual-verification wording, not the fix. Resolution: Igor confirm the scenario; assert manual scenario 3 ("no raw keys / no flash of missing chrome on every entry to `ready`") rather than a specific window.

- **Adjacent observations (Part 4b):**
  1. **AppContext torn out but `.agent/audit-post-pick-consumers.md` not updated** (low) — the audit's store attribution is stale (consumers are bootStore now, AppContext gone). Doc artifact only; flagged so a future reader trusts the code, not the audit's store column. Not fixed (not my file).
  2. **Inside-Stack non-chrome consumers also mount during `updating`** (low) — the whole portal tree mounts in `updating`; tab screens (e.g. ProductList) and their translated children technically share the same boundary-frame exposure as the chrome did. They are occluded during `updating` and carry their own loading/empty states, and product content is data (not translation keys), so the chrome was the acute/visible case. If a residual flash is ever observed on a screen body, the same `ready`-gate remedy applies there — out of scope for 7d. Not fixed.
  3. **Dialogs are structurally outside the gate** (low/informational) — they ride a RN `<Modal>` above the splash overlay; safe today only because they open exclusively from ready-state UI. If a future feature ever auto-opens a dialog during boot, it would need a `ready` guard. Noted for awareness; no action now.

- **Stopped per the brief's STOP gate.** Phase 0 reported (A–D above). Phase 1 not started. Awaiting Mastermind to release Phase 1.
- Nothing else flagged.
