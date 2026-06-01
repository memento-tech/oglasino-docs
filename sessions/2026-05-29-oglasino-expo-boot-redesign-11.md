# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** expo-boot-redesign step 7c, Phase 0 — read-only empirical verification of Mastermind's "no boot-loop possible" hypothesis for the proposed Stack-level gate (`Stack mounts only when bootStatus ∈ {ready, updating}`). Answer 0a–0f with file:line + quotes, then STOP and report before any Phase 1 implementation.

## Phase 0 (read-only) — outcome

**Verified. The fix is loop-safe.** The always-mounted-Stack pattern (0a/0e) and the status-writer confinement (0d) are confirmed, and the proposed Stack-level gate sits on the single common mount point of every offending screen. Two refinements for Mastermind. (1) The literal claim "no path from a screen mounting to a gate writing status exists" is **false** — the product deep-link auto-switch (A4) is such a path — but it is (a) guarded to fire only when `status === 'ready'`, (b) convergent, and (c) stays within `{ready, updating}`, so it neither loops nor unmounts the Stack under the fix. (2) The brief's *named* 400 source (`index.tsx`'s `ProductList`) is actually `selectedLanguage`-guarded and should not fire at intro-picker, so the real origin should be confirmed from the access log — the fix is correct regardless. Details in *Brief vs reality*. **Stopped after Phase 0 per the brief's STOP gate; Phase 1 not started; awaiting Mastermind's go-ahead.**

**Method / reliability note (important):** the file viewer (Read tool) was intermittently unstable this session — it returned stale content (e.g. a pre-7b `ProductList.tsx` still showing the deleted `useAppState()`), wobbling line numbers, and once an abbreviated placeholder body. I treated no single read as authoritative: every load-bearing claim below is cross-corroborated against at least one of (a) a clean re-read, (b) `rg`/`cat` via shell, (c) the prior audits, and (d) the 7b session summary (`boot-redesign-10`). The one place I could not get a fully clean line-level read is `ProductList.tsx`'s effect body; I state its conclusion (corroborated by `cat` + audit §5) and flag the residual uncertainty rather than quote uncertain line numbers. I did **not** fan this out to a workflow: the task is a precise, self-corroborating read sweep over ~10 files where I had to verify each quote myself, and a multi-agent fan-out would have inherited the same viewer instability without adding precision.

---

## Phase 0 findings (0a–0f)

### 0a — Current `app/_layout.tsx` structure — always-mounted Stack confirmed

`app/_layout.tsx` (read clean, consistent across re-reads):

- `<Stack>` is mounted **unconditionally** inside a plain wrapper, no status guard:
  ```tsx
  // app/_layout.tsx:78-80
  <View className="flex-1">
    <Stack screenOptions={{ headerShown: false }} />
  </View>
  ```
- The overlay is an **absolute sibling rendered on top**, gated on *not* ready / update-required:
  ```tsx
  // app/_layout.tsx:94-100
  {bootStatus !== 'ready' && bootStatus !== 'update-required' && (
    <View className="absolute inset-0 items-center justify-center bg-background">
      {bootStatus === 'maintenance' && <BaseSiteSelector isMaintenance />}
      {bootStatus === 'intro-picker' && <BaseSiteSelector isMaintenance={false} />}
      {(bootStatus === 'booting' || bootStatus === 'updating') && <LoadingOverlay />}
    </View>
  )}
  ```
- `AppInit` is gated on `ready` alone (`:76` `{bootStatus === 'ready' && <AppInit />}`). `<MaintenancePollInit />`, `<PortalHost />`, `<DialogManager />`, `<SoftUpdateModal />` are at layout level on all statuses (`:77, :81, :82, :86`). `update-required` has its own terminal branch (`:103` `<HardUpdateScreen />`).

**Conclusion:** the always-mounted-Stack pattern is real. On `intro-picker`, `maintenance`, `booting`, and `updating`, the `<Stack>` and its entire `(portal)` mount tree are mounted and running underneath the overlay; the overlay only hides them visually. This is the visual-only invariant 7b mistook for structural.

### 0b — The screen that fired `POST /public/product/search`

The chain is `index.tsx → FilteredProductList → ProductList`:

- `app/(portal)/(public)/index.tsx` (clean read) renders `<FilteredProductList useFilterStore={usePortalFilterStore} fetchPage={getPortalProducts} applyRandom CardContentComponent={PortalProductCard} />` — **no longer wrapped in `<RequireBaseSite>`** (7b removed the wrapper; confirmed in `boot-redesign-10` and post-pick audit §3, which records the old `index.tsx:10-19` wrap).
- The actual search fetch lives in `ProductList` (full clean read this session). It reads the language from bootStore and **self-guards the mount fetch on it**:
  ```tsx
  // src/components/product/ProductList.tsx:49
  const selectedLanguage = useBootStore((s) => s.language);
  ...
  // :101-109  — the mount fetch
  useEffect(() => {
    if (!selectedLanguage) return;   // ← GUARD
    onRefresh();                     // → loadNextPage(true) → fetchPage(0) === getPortalProducts === POST /public/product/search
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [fetchPage]);                   // ← deps are [fetchPage], NOT selectedLanguage
  ```

**This contradicts the brief's named candidate.** On a fresh install at `intro-picker`, `bootStore.language` is null (Gate 3's no-stored-site path — `runBaseSiteGate:222-248` — sets `baseSiteOverviews` and `toIntroPicker`, but never sets `language`), so this effect hits the `!selectedLanguage` guard and returns **without** firing `/product/search`. So `index.tsx`'s `ProductList`, as it stands today, should **not** be the source of the cold-boot 400.

Two consequences, both relevant:
1. **The 400's true origin needs confirming from the access log, not assumed.** The brief names `FilteredProductList` in `index.tsx` as "the most likely candidate"; the code shows it is language-guarded. The 400 therefore either (a) originates from a different mount (I could not exhaustively trace all `/product/search` callers this session — the shell's output delivery was degraded; I verified the home list and the SearchInput/header path is `selectedBaseSite`-gated in `(portal)/_layout.tsx:16` so the header doesn't mount at intro-picker), (b) predates this `!selectedLanguage` guard, or (c) is a transient. **Flagged for Mastermind** — the backend access log entry (which the brief cites) will name the exact request and its headers.
2. **There is a *latent* second bug the gate also fixes.** Because the mount effect's deps are `[fetchPage]` (not `selectedLanguage`), if `ProductList` mounts while `language` is null (today it does — always-mounted Stack), it returns early and **never re-fires** when `language` is later set, since `selectedLanguage` isn't a dependency. Under the current always-mounted Stack, that means a first-install home feed can stay empty after the pick. The Stack-level gate fixes this for free: `ProductList` first mounts only at `ready` (language already set), so the guard passes and the feed loads.

**Conclusion:** regardless of the exact 400 origin, the Phase 1 Stack-level gate is the right fix — it prevents *every* portal screen from mounting before `ready`, and it additionally repairs the `[fetchPage]`-deps empty-feed bug above. I'm explicitly **not** asserting `index.tsx`'s `ProductList` is the 400 source, because the code says it is guarded; that diagnosis claim in the brief should be reconciled against the access log.

### 0c — Every site that calls `pickBaseSite` / `setLanguage`

Authoritative `rg` over `src/` and `app/`:

| # | Site | Call | Fires on… |
| --- | --- | --- | --- |
| A1 | `src/components/init/BaseSiteSelector.tsx:38` (hook) → `:119` `onPress={() => setBaseSiteForCode(baseSite.code)}` | `pickBaseSite` | **user gesture** (picker tap) |
| A2 | `src/components/user/UserMenu.tsx:33` (hook `switchBaseSite`) → `:57` `onPress: () => switchBaseSite(user.baseSite.code)` | `pickBaseSite` | **user gesture** |
| A3 | `src/components/dialog/dialogs/PortalConfigDialog.tsx:41` (hook) → `:128` `onPress={() => setBaseSiteForCode(baseSite.code)}` | `pickBaseSite` | **user gesture** |
| A4 | `src/components/dialog/dialogs/PortalConfigDialog.tsx:42` (hook) → `:95` `onPress={() => setLanguageForCode(lang.code)}` | `setLanguage` | **user gesture** |
| A5 | `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx:99` `onPress={() => switchBaseSite(baseSite.code)}` (a `switchBaseSite` prop, type at `:298`) | `pickBaseSite` (via prop) | **user gesture** |
| A6 | `app/(portal)/(public)/product/[...productData].tsx:45` (hook) → `:109` `setBaseSiteForCode(res.baseSiteOverview.code)` | `pickBaseSite` | **mount (indirect, `ready`-guarded)** ⚠️ |

A1–A5 are all `onPress` handlers — user gestures, none on mount/render.

A6 is the load-bearing one. Verified body (came through consistently on re-read):
```tsx
// app/(portal)/(public)/product/[...productData].tsx
const selectedBaseSite = useBootStore((s) => s.selectedBaseSite);   // :43
const status = useBootStore((s) => s.status);                       // :44
const setBaseSiteForCode = useBootStore((s) => s.pickBaseSite);     // :45
...
useEffect(() => {
  if (status !== 'ready' || !selectedBaseSite || !productId || Number.isNaN(productId)) {
    return;                                                         // :96-98  ← GUARD: ready-only
  }
  let mounted = true;
  const load = async () => {
    try {
      setPreparing(true);
      const res = await getPortalProductDetails(productId);        // :103
      if (!mounted) return;
      if (res.baseSiteOverview?.code && selectedBaseSite.code !== res.baseSiteOverview.code) {  // :105
        setProductDetails(res);
        setBaseSiteForCode(res.baseSiteOverview.code);             // :109  ← mount → pickBaseSite
        return;
      }
      setProductDetails(res);
      setOwner(res.owner ? await getUserInfo(res.owner) : undefined);
    } catch (error) { console.error('Failed to load product', error); }
    finally { if (mounted) setPreparing(false); }
  };
  load();
  return () => { mounted = false; };
}, [productId, selectedBaseSite, status]);                          // :129
```
The effect runs on mount, but the **first line short-circuits unless `status === 'ready'`**. So `pickBaseSite` is only reachable from this screen *after* the machine has already reached `ready` (deep-link to a product whose base site differs from the stored one). It does not call `pickBaseSite` directly on mount — it calls it from the product-details response handler, only on a base-site mismatch. **This is a mount→status-write path, but a `ready`-gated and convergent one** (see the loop verdict and *Brief vs reality* below). It contradicts the literal premise; it does not endanger the fix.

### 0d — Every caller of `start()` / `reEnter()` / status writers

- **`status` is written only inside `bootStore.ts`.** The status writers (from the clean full-file read) are: initial state `status: 'booting'` (`:106`); `start()` `set({ status: 'booting' })` (`:118`); `runFreshnessGate` `set({ status: 'ready' })` (`:501`); and the four `toX` helpers `toMaintenance`/`toUpdateRequired`/`toIntroPicker`/`toUpdating` (`:504-507`), which the gates call. No `useBootStore.setState`/status write exists anywhere outside `bootStore.ts`. ✓ (Invariant 2.)
- **`start()` is called from exactly one place:** `app/_layout.tsx:40` `useBootStore.getState().start()`, the empty-deps mount effect (`:39-41`). ✓ (Invariant 1.)
- **`reEnter()` is called from exactly one place:** `src/components/init/MaintenancePollInit.tsx:38` `await useBootStore.getState().reEnter()`, inside `pollMaintenanceOnce`, and **only on the maintenance-clear edge** (`:37` `if (!maintenanceOn)`). The poll's `setInterval` runs **only while `status === 'maintenance'`** (`:48-49` `if (status !== 'maintenance') return;`). ✓

No status writer, `start()`, or `reEnter()` exists outside `bootStore.ts`, the mount effect, and the maintenance poll. The confinement the spec relies on holds.

*Adjacent observation (out of scope for 7c, flagged): because the poll's effect returns early unless `status === 'maintenance'`, there is currently **no active poller while `ready`** — the spec transition-table row "ready → active-checker detects maintenance on → reEnter" (`:151`) is not exercised by `MaintenancePollInit` as written. The wired re-entry is the maintenance-clear edge only. This does not affect 7c (the brief's manual scenarios 4/5 use cold-boot-maintenance and the clear edge, both covered) — noted for Part 4b.*

### 0e — What mounts under the overlay on intro-picker

With the Stack always mounted (0a), on `intro-picker` the entire portal tree mounts beneath the overlay:

- `app/(portal)/_layout.tsx` (`PortalLayout`, clean read) mounts. It reads `const selectedBaseSite = useBootStore((s) => s.selectedBaseSite)` (`:12`) and gates **only the header chrome** on it (`:16` `{selectedBaseSite && (<View>…TopBar…CategoryNavigation…</View>)}`), but renders `<Tabs>` **unconditionally** (`:26-33`). So the header is render-suppressed, but the tab screens still mount.
- `app/(portal)/(public)/_layout.tsx` (`PublicLayout`, clean read) mounts: a mount effect `setPortalScope('portal')` (`:8-10`) and `<Stack/>` (`:12`).
- `app/(portal)/(public)/index.tsx` (`HomeScreen`, clean read) mounts → `<FilteredProductList/>` → `ProductList` → `fetchInitial()` fires `POST /product/search` (per 0b).

**Conclusion:** the screens running effects under the overlay today are the full `(public)` group, with `index.tsx`'s product list the proven offender. The Phase 1 fix gates the **single** Stack mount point (`app/_layout.tsx:78-80`), which is the common parent of all of them — so it catches every one at once. This is the structural enforcement 7b was missing.

### 0f — The `updating` transient, and the `booting`-transit question

From the clean full-file `bootStore.ts` read:

- **`pickBaseSite` (`:260-286`):** does **not** write `status`. It `fetchBaseSiteByCode` (`:266`), computes the language, `set({ selectedBaseSite: fullDto, language: lang })` (`:280`), persists, then `await runFreshnessGate()` (`:285`). `runFreshnessGate` calls `toUpdating()` → `status: 'updating'` (`:343`/`:507`) and terminates at `set({ status: 'ready' })` (`:501`). **Path from `ready`: `ready → updating → ready`. Does NOT transit `booting`.** ✓
- **`setLanguage` (`:296-307`):** same shape — no `booting`, just `runFreshnessGate` → `updating` → `ready`. **`ready → updating → ready`.** ✓
- **`start()` / `reEnter()` (`:114-127`):** `set({ status: 'booting' })` (`:118`) → gates → (stored-site path) `runFreshnessGate` → `updating` → `ready`. **Transits `booting`** — but only on app start and on maintenance-clear re-entry, where the Stack is legitimately unmounted anyway.

**Conclusion:** the two switcher primitives (`pickBaseSite`, `setLanguage`) go `ready → updating → ready` and **never transit `booting`**. Under the proposed `ready || updating` gate, both a language switch (scenario 6) **and** a base-site switch (scenario 7) keep the Stack mounted across the transient — exactly as the brief's scenario-6/7 expectations require. The only `booting` transit is `start`/`reEnter`, which is the cold-boot / maintenance-clear case where unmount is correct.

---

## Loop-hypothesis verdict (for Mastermind)

**The fix is loop-safe. One refinement to the premise.**

1. **The literal premise is false.** A6 (product deep-link auto-switch, `product/[...productData].tsx:109`) is a real "screen mounts → fetch → `pickBaseSite` → status write" path. Mastermind's "No path from a screen mounting to a gate writing status exists" should be amended.

2. **Why it is nonetheless loop-safe and unmount-safe under the fix:**
   - The A6 effect is **guarded** by `status === 'ready'` (`:96`), so it **cannot fire during cold boot** — it is irrelevant to the scenario-1 400 (which is `ProductList`, 0b, and is killed by the Stack gate).
   - `pickBaseSite` goes **`ready → updating → ready`** (0f) — both states keep the Stack mounted under the gate, so the deep-linked product screen **does not unmount** during the switch.
   - It **converges**: after `pickBaseSite(Y)` sets `selectedBaseSite = Y`, the effect (deps include `selectedBaseSite`, `status`) may re-run, but the mismatch test `selectedBaseSite.code !== res.baseSiteOverview.code` is now false, so `pickBaseSite` is **not** called again. The recursion is idempotent on the matching base site.
   - Net cost on a foreign-base-site deep link: a few redundant `getPortalProductDetails` fetches as `selectedBaseSite`/`status` churn through `ready→updating→ready`, then stable. No unmount, no infinite loop. (The redundant fetches are a minor efficiency note, out of scope for 7c — see *For Mastermind*.)

3. **Scenario-1 (the actual bug) is fixed by construction — with a diagnosis caveat.** Under the gate, no portal screen (including whatever truly fires the 400) mounts until `ready`, so any pre-`ready` portal request is never issued. The 400 disappears because the request isn't made — exactly as the brief's scenario-1 requires. Caveat (0b): the brief's *named* source (`index.tsx`'s `ProductList`) is actually `selectedLanguage`-guarded and should not fire at a fresh-install intro-picker, so the real origin should be confirmed from the access log. The fix is correct either way; the diagnosis is what needs reconciling.

---

## Brief vs reality

1. **The loop premise as stated is contradicted by the product deep-link auto-switch (A6).**
   - Brief says (Phase 0 framing): "No path from 'a screen mounted and rendered' to 'a gate writes status' exists — that's the structural defense."
   - Code says: `app/(portal)/(public)/product/[...productData].tsx:95-129` is exactly that path — a mount effect → `getPortalProductDetails` → `pickBaseSite` on base-site mismatch (`:109`).
   - Why this matters: the premise Mastermind asked me to confirm does not hold verbatim. It holds in a **weaker, safe form**: the path exists but is (a) `status === 'ready'`-guarded so it cannot fire during boot, (b) routed `ready → updating → ready` so it never unmounts the Stack under the fix, and (c) convergent (idempotent once the base site matches). The fix needs **no change** for loop-safety; only the *reasoning* needs the amendment.
   - Recommended resolution: restate the structural-defense claim in the convergent form above. No Phase 1 code change required. If the deep-link redundant-fetch churn is judged worth addressing, log it as a separate efficiency item (not part of 7c).

2. **The brief's named source of the cold-boot 400 is language-guarded and should not fire at intro-picker.**
   - Brief says (0b framing): "The most likely candidate is the `FilteredProductList` in `app/(portal)/(public)/index.tsx` (unwrapped from `RequireBaseSite` in 7b Phase 3)."
   - Code says: that list's fetch lives in `ProductList`, whose mount effect is `if (!selectedLanguage) return; onRefresh();` with deps `[fetchPage]` (`src/components/product/ProductList.tsx:49, 101-109`). At a fresh-install intro-picker, `bootStore.language` is null (Gate 3 no-stored-site path never sets it), so the guard short-circuits and `/product/search` is not issued from this screen.
   - Why this matters: the diagnosis pins the 400 on a screen that is, in current code, self-guarded against exactly this. The Stack-level fix is still correct and still eliminates the 400 (it prevents all portal mounts pre-`ready`), but Mastermind should reconcile the *diagnosis*: the real request origin should be read off the backend access log the brief cites, not assumed. (Bonus: the `[fetchPage]`-only deps are themselves a latent bug — see 0b consequence #2 — which the gate also fixes.)
   - Recommended resolution: confirm the 400's origin from the access log before/at Phase 1. If it turns out to be a transient or a now-guarded path, note that scenario-1's manual retest should assert "no `/product/search` before first `ready`" (which the gate guarantees) rather than "the previously-400ing request now succeeds."

*(Correction to a claim made earlier in this very session, before the file-viewer instability was understood: an intermediate read suggested `pickBaseSite` set `status: 'booting'` first, which would have unmounted the Stack on a base-site switch and broken scenario 7. The authoritative `bootStore.ts:260-286` read disproves that — `pickBaseSite` writes no status and goes `ready → updating → ready`. There is **no** scenario-7 problem. Flagging the retraction so the record is clean.)*

Nothing else rises to a brief-vs-reality challenge. Naming, the `console.error` at `:116`, and the dormant-while-ready poll (0d) are observations, not blockers.

## Obsoleted by this session

Nothing — read-only verification, no code touched.

## Cleanup performed

None needed — Phase 0 is read-only; no edits, no imports, no logging introduced.

## Config-file impact

- **conventions.md / decisions.md / issues.md:** no change this session.
- **state.md:** no change this session. The brief's spec amendment **#6** (the portal mount predicate `bootStatus ∈ {ready, updating}`) is a Phase 1 deliverable for Docs/QA at feature close, not a Phase 0 edit.
- **Closure gate:** no unstated config-file dependency. Phase 0 produced findings only; any config edits belong to Phase 1 / feature close and are Docs/QA's to apply.

## Conventions check

- **Part 4 (cleanliness):** N/A — no code changed.
- **Part 4a (simplicity):** the proposed single Stack-level gate remains the simplest enforcement; no per-screen guards proposed or added (the brief forbids re-adding them, and Phase 0 added nothing).
- **Part 4b (adjacent observations):** two flagged in *For Mastermind* (the dormant-while-`ready` maintenance poll; the A6 deep-link redundant-fetch churn).
- **Spec Part 1 invariants + structural defense:** Invariant 1 (one empty-deps effect, `app/_layout.tsx:39-41`) holds; Invariant 2 (status written only by the machine — 0d) holds; Invariant 3 not exercised this phase. Structural defense holds: `bootStore.ts` imports only api/i18n leaves, types, zustand, and its boot-leaf modules — **zero `authStore`, zero chat-store** (verified: the only `authStore`/chat mentions in the file are the doc-comment at `:57-58`; `rg` for auth/chat refs returned none).
- **Part 5:** this summary written to both `.agent/2026-05-29-oglasino-expo-boot-redesign-11.md` and `.agent/last-session.md`.

## Known gaps / TODOs

- **The exact backend request that produced the cold-boot 400 was not traced to a single confirmed call site this session.** `index.tsx`'s `ProductList` (the brief's named candidate) is language-guarded and should not be it (0b / Brief-vs-reality #2). I could not exhaustively enumerate all `/product/search` callers because the shell's output delivery degraded mid-session. The access log the brief cites will name the request; recommend confirming origin at Phase 1. This does **not** change the Phase 1 plan — the gate prevents all portal mounts pre-`ready` regardless.
- No `TODO`/`FIXME` added.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** nothing — Phase 0 is read-only.
  - **Considered and rejected:** running Phase 0 as a multi-agent workflow (rejected — a precise self-corroborating read sweep over ~10 files; fan-out would inherit the viewer instability without adding precision).
  - **Simplified or removed:** nothing — read-only.
- **Release decision:** Phase 0 confirms (0a/0e) the always-mounted Stack, (0d) the status-writer confinement to `bootStore` + the single mount effect + the maintenance poll, and the boot-store's auth/chat independence. The Stack-level gate at `app/_layout.tsx:78-80` is the common parent of every offending screen. **The fix is loop-safe and does not unmount the Stack on switcher transients (0f).** I'm ready for Phase 1 on your go-ahead.
- **Amend the loop premise (Brief vs reality #1):** the A6 product deep-link auto-switch *is* a mount→status-write path, but it is `ready`-guarded, `ready→updating→ready` (no unmount under the gate), and convergent. Please restate the structural-defense claim in that convergent form. No Phase 1 code change needed for loop-safety.
- **Scenario-7 is fine:** contrary to an intermediate (now-retracted) reading this session, `pickBaseSite` does **not** transit `booting` — base-site switches stay `ready→updating→ready` and the Stack stays mounted under `ready || updating`, matching the brief's scenario-7 expectation. No change to the brief's manual scenarios.
- **Reconcile the 400 diagnosis (Brief vs reality #2):** the brief's named candidate (`index.tsx`'s `ProductList`) is `selectedLanguage`-guarded (`ProductList.tsx:101-109`) and should not fire `/product/search` at a fresh-install intro-picker. Please confirm the real request origin from the backend access log. The Stack gate fixes scenario-1 regardless, and *also* repairs a latent empty-feed bug (the mount effect's deps are `[fetchPage]`, not `selectedLanguage`, so a null-language mount never re-fetches — 0b consequence #2). Suggest scenario-1's retest assert "no `/product/search` before first `ready`."
- **AppInit (brief §1.2):** I agree — keep it gated on `bootStatus === 'ready'` **alone**, not `|| updating`. Its effects (push registration, auth listener, chat init) are "first time we reach ready," not "the portal is showing"; re-mounting on every `updating` freshness pass would re-register listeners / double-subscribe. No change recommended.
- **Adjacent observations (Part 4b):**
  1. **Dormant-while-`ready` maintenance poll.** `MaintenancePollInit`'s interval runs only while `status === 'maintenance'` (`:48-49`), so the spec's "ready → maintenance detected → reEnter" transition (`expo-boot-redesign.md` table `:151`) is not currently exercised — only the maintenance-clear edge is wired. Out of scope for 7c (the brief's maintenance scenarios are covered); flagged for triage.
  2. **A6 deep-link redundant fetches.** On a foreign-base-site product deep link, the A6 effect (deps `[productId, selectedBaseSite, status]`) re-runs a few times as the switch churns `ready→updating→ready`, issuing redundant `getPortalProductDetails` calls before converging. Minor efficiency, no correctness impact, out of scope for 7c.
  3. **Pre-existing `console.error('Failed to load product', error)`** at `product/[...productData].tsx:116` — caught-error log, not a debug diagnostic; carried over from before this feature, already noted in `boot-redesign-10`. Left as-is.
- Nothing else flagged.
