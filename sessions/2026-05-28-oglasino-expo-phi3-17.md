# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Produce a concrete target-architecture proposal for refactoring the app's boot/HTTP layer so that the class of bugs seen across Briefs 12–16 cannot recur. Read-only design session.

## Implemented

- Read-only design audit. No code changes. No instrumentation.
- All ten file-groups in the brief read end-to-end.
- Brief 12–16 session summaries re-read for prior context.
- Φ2 entries in `decisions.md` (2026-05-25 and 2026-05-27) re-read verbatim.
- D1–D7 answered explicitly with concrete recommendations.
- One recommended target architecture stated as a decision; rejected alternatives noted.
- Implementation order (D6) sketched as the skeleton for the next brief.

## Files touched

None. Read-only.

## Tests

Not run. No code changes.

## Cleanup performed

None needed. No code changes.

## Config-file impact

- conventions.md: no change
- decisions.md: no change required from this session. If the recommended target lands, the close-out chat for the refactor will draft a `decisions.md` entry. Pre-staged title for that entry, for Mastermind to consider: "Boot/HTTP layer refactor: codes-derived barrier, per-screen base-site render gate, api↔authStore cycle broken."
- state.md: no change. The Φ3-close-out / boot-refactor work is already implicit in the "Expo structural foundation" trajectory; no new backlog row needed.
- issues.md: no change. Briefs 12–16 are tracked in their session summaries; the open `[DBG]` log in `api.ts:36-45` is a known carry-over to be removed by the implementation brief, not a fresh entry.

## Obsoleted by this session

Nothing in code. Read-only design.

If the recommended target lands, the following pieces of code become dead and will be deleted in the implementation brief:
- The separate `bootstrapResolve` + `bootstrapPromise` couple in `apiStore.ts:4-7`. Replaced by a codes-derived `waitForBootstrap()` plus a waiter list.
- The `useAuthStore` import and direct `setRestored` / `setAccountBanned` calls in `api.ts:3,64,88`. Moved to a separate `authInterceptors.ts` registered from `AppInit`.
- The remaining `[DBG]` console.log in `api.ts:36-45` (still present after Brief 16; Brief 14's diagnostic removal missed it because it was a different DBG site added during Brief 16's audit).

## Conventions check

- Part 4 (cleanliness): N/A — no code changes.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): two observations flagged in "For Mastermind."
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults) — the proposed barrier sits at the shared HTTP layer per the "routes are reusable" intent. Part 5 (session template) — followed below.

## Known gaps / TODOs

- The recommended target assumes ES-module live bindings and Metro's per-absolute-path module caching as documented. The one residual unknown is whether `globalThis`-pinned state survives `r-r` (full Metro reload) the same way it survives Fast Refresh — likely yes, since `r-r` triggers a full JS context reset and `globalThis` resets with it, which is the right behavior. Confirm with a single smoke step in the implementation brief.
- D6's step ordering is the natural dependency order; alternative orderings are possible. Implementation brief is free to bundle steps 1+2 (boot-state and cycle-break) into one session if the engineer prefers, but step 3 (render gate) should follow as its own session because it changes user-visible behavior on the intro path.

---

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):**
  - The proposal adds one new component (`<RequireBaseSite>`, ~5 lines) and one new init file (`authInterceptors.ts`, ~25 lines). Both earn their place: the component is the single render gate enforcing Constraint 1 across multiple screens (justification per Part 4a — solves a concrete problem visible today across at least the home, catalog, product, and user surfaces); the init file is what allows `api.ts` to stop importing `useAuthStore`, closing the longest-standing require cycle.
  - The proposal adds a `globalThis` pin on the apiStore singleton (~4 lines). Justified by the Brief 16 finding that Fast Refresh can re-evaluate `apiStore.ts` independently of `api.ts`, leaving the interceptor closure bound to a stale state object. The `globalThis` pin is the standard React-Native / Metro pattern for surviving Fast Refresh (mirrors zustand and TanStack Query's well-known patterns).

- **Considered and rejected:**
  - **(a) Move boot-critical state into AppContext / a dedicated zustand store and have the axios interceptor read via `getState()`.** Rejected — introduces a fresh dependency from `api.ts` into a React-tree-coupled store, *adding* a cycle (axios interceptor importing a zustand store that may transitively re-export React types) where we are trying to *remove* one. The plain module + `globalThis` pin is mechanically simpler and structurally lighter.
  - **(b) A class-instance singleton (`new BootState()`) exported from `apiStore.ts`.** Rejected — exposes more API surface than needed (`bootState.getBaseSite()` instead of `getCurrentBaseSiteCode()`) and creates a JS-class object whose identity is tied to module identity. The function-export shape is closer to what's there today and matches surrounding style (Part 4a "match the surrounding code's style").
  - **(c) Per-screen guard on `ProductList` only, no `<RequireBaseSite>` boundary.** Rejected — the catalog, product detail, and user detail screens all fire backend calls on mount (verified via grep: `app/(portal)/(public)/product/[...productData].tsx:90` and `app/(portal)/(public)/user/[...userData].tsx:27`). A boundary component covers all of them with one wrap each. One wrapper component beats N inline guards.
  - **(d) Break the chat-store triangle in this refactor.** Rejected — the triangle is feature-coupled (cross-store actions in `useActiveChatStore.sendMessage`, `useChatNavStore.setTempReceiver`, `useActiveChatStore.getActiveChat` are the feature's actual workflow). Pulling them into a `chatActions.ts` coordinator is a 200–400-line refactor that belongs in chat-A or chat-B (per the Expo backlog) where feature-level test coverage exists. Tracked as a known gap; deliberately deferred. The chat-triangle cycle is benign at evaluation order (all cross-calls go via `useXStore.getState()` — runtime, not top-level), so leaving it does not regress safety.
  - **(e) Conditionally render `<Tabs>` or the inner `<Stack>` to physically prevent the feed from mounting.** Rejected — Φ2 decisions.md 2026-05-27 explicitly forbids conditionally rendering *any* navigator (Stack or Tabs). The constraint exists because conditional navigators trigger expo-router's navigation-container reinit and an observed infinite loop. Gate inside the screen body, not the navigator.
  - **(f) Add a 15-second timeout on the barrier.** Already rejected in Brief 14 per Igor's hard rule. Re-confirming the rejection: a timeout converts a hanging request into a wrong-headers request (or a synthetic error), which is worse than the hang because it masks the real cause. The right answer is "the barrier never blocks when codes-set" plus "codes are set on every path that should fire requests."
  - **(g) Replace the `_bootstrap` config flag with URL-pattern matching.** Already rejected in Brief 14. Keeping the explicit flag.

- **Simplified or removed (in the proposal):**
  - Deletes the `bootstrapResolve` + `bootstrapPromise` couple in favor of a single `waiters: (() => void)[]` array plus a fresh codes-present check inside `waitForBootstrap()`. Net: ~10 lines simpler in `apiStore.ts`.
  - Deletes `api.ts:3` `useAuthStore` import and the two direct store mutations in interceptors (lines 64 and 88) by registering them via the new `authInterceptors.ts` init file. Net: `api.ts` becomes a pure HTTP-plumbing module with no React-tree coupling.

### Part 4b adjacent observations

1. **`api.ts:36-45` still carries a `console.log('[DBG]', ...)` block.** Brief 14's cleanup removed three diagnostic sites but did not touch this one (a fresh DBG added during Brief 16's audit attempt). Per the brief: "The existing `[DBG]` logs may be read, not extended." File: `src/lib/config/api.ts:36-45`. Severity: low (cosmetic). Will be removed as part of the implementation brief's cleanup pass; no separate action needed now.
2. **`AppVersionConfigInit.tsx:29` still carries the misspelled TODO `// TODO: Translations PLUS after meintanence page do better check`.** Flagged in Brief 16; re-flagging because it sits in the audited file scope here. Severity: low. No matching `issues.md` entry. Out of scope for this design.

---

## Current-state topology (boot/HTTP/cycles/render)

### Module evaluation order (cold start)

1. Metro entry `expo-router/entry` resolves `app/_layout.tsx` (`package.json:3`).
2. Transitive imports eagerly evaluate `api.ts` → `authStore.ts` → `authService.ts` → `api.ts` (cycle, benign at eval); `apiStore.ts` (leaf, no imports); the bootstrap services; the chat-store triangle (`useActiveChatStore` ↔ `useChatNavStore` ↔ `useChatListStore` plus all three back-reference `authStore`, and `authStore.ts` imports all three — also a cycle, benign at eval because all cross-calls go via `useXStore.getState()`).
3. `api.ts:148` instantiates `BACKEND_API = createApiInstance(BACKEND_API_URL)` — request and response interceptors are both attached **before** `BACKEND_API` is observable to any consumer (`createApiInstance` is pure-return).
4. `apiStore.ts` evaluates once. Module-scope `let` bindings: `currentBaseSiteCode = null`, `currentLangCode = null`, `bootstrapResolve = (resolver captured)`, `bootstrapPromise = new Promise(...)`. Singleton via Metro's absolute-path cache; Babel alias `@` → `./src` (`babel.config.js`) and the relative `./apiStore` import from `api.ts` both resolve to the same absolute file.

### Boot/render topology (post-Φ2, current)

- `RootLayout` (`app/_layout.tsx:25-85`) holds local `status` (`useState<AppStateValue['status']>()` — initialized to `undefined`). `setStatus` is passed as `onStatusChanged` to `AppContextProvider`.
- Provider tree: `GestureHandlerRootView` → `AppContextProvider` → `AppVersionConfigInit` → `ToastProvider` → `ThemeProvider` → `SafeAreaView`.
- Inside `SafeAreaView`:
  - `{status === 'ready' && <AppInit />}` (`_layout.tsx:62`) — auth listener, chat init, etc., gated on terminal-ready.
  - `<Stack screenOptions={{ headerShown: false }} />` (`_layout.tsx:64`) — **always mounted**, auto-discovers `(portal)`, `owner`, `+not-found`, `__smoke__`.
  - `<PortalHost />` + `<DialogManager />`.
  - `{status !== 'ready' && (<absolute overlay>)}` (`_layout.tsx:69-77`) — visual cover for loading / maintenance / select-base-site; **does NOT prevent the screens underneath from mounting**.
- `(portal)/_layout.tsx:11-36` — renders `{selectedBaseSite && (<chrome>)}` (chrome guard from Φ2) then `<Tabs>`. Tabs is always mounted; only the chrome is conditional.
- `(portal)/(public)/_layout.tsx:5-13` — renders `<Stack screenOptions={{ headerShown: false }} />`. Side-effect: `setPortalScope('portal')`.
- `(portal)/(public)/index.tsx:7-18` — mounts `<FilteredProductList .../>`. No render gate; fires regardless of `selectedBaseSite`.
- `ProductList.tsx:101-103` — `useEffect(() => { onRefresh(); }, [fetchPage])` — unconditional mount-phase fetch.

### Boot-critical state (`apiStore.ts:1-31`)

- Four module-level `let` bindings: `currentBaseSiteCode`, `currentLangCode`, `bootstrapResolve`, `bootstrapPromise`.
- Coupling: only `setCurrentBaseSiteCode` (line 17-23) resolves the promise. `setCurrentLangCode` does not.
- Invariant the interceptor relies on (`api.ts:46`): `!getCurrentBaseSiteCode() && !config._bootstrap` ⇒ `await waitForBootstrap()`.
- The coupling is *by convention*: the file's author maintained "resolver fires iff code is set." A future edit could break this without lint/tsc catching it — there is no structural derivation.
- Fast-Refresh fragility (Brief 16 Mechanism A): if `apiStore.ts` re-evaluates piecemeal, the OLD interceptor closure can read OLD `getCurrentBaseSiteCode` / OLD `waitForBootstrap` while `AppContext.tsx`'s live bindings point at the NEW module's setters. The barrier becomes desynced from codes for the duration of that mismatch.

### Require cycles (verified by grep)

- **Cycle A — auth-HTTP triangle**: `api.ts:3` imports `useAuthStore` from `authStore.ts:13` which imports from `authService.ts:15` which imports `BACKEND_API` from `api.ts`. Benign at module-evaluation because no consumer reads `BACKEND_API` at top level — all reads are inside async function bodies executed after evaluation. Brief 16 confirmed this. Metro emits a "Require cycle" warning at boot.
- **Cycle B — chat triangle (Φ3-introduced)**: `useActiveChatStore.ts:21,30,31,32` ↔ `useChatNavStore.ts:6,7,8` ↔ `useChatListStore.ts:17` (all three back-reference `useAuthStore`; the active/nav stores cross-call each other directly). Plus `authStore.ts:21,22,23` imports all three chat stores to clear them on `logout()`. Benign at evaluation for the same reason (all cross-calls via `getState()`). Metro emits two "Require cycle" warnings.

### Subordinate dependencies of `BACKEND_API`

Twenty consumer files (verified by `grep "from .*config/api"`): six bootstrap-or-init services, the auth service, fourteen feature services (catalog, products search, reports, etc.), the i18n namespace fetcher, the push-token service, and the view-tokens store. Among these, **only `authService.ts` participates in Cycle A** (because `authStore.ts` is `api.ts`'s back-edge target, and `authStore.ts` imports `authService.ts`). The other nineteen are leaf consumers — they import `BACKEND_API` and call methods on it, with no back-import to `api.ts`'s dependency closure.

---

## D1 — Boot-state hardening

**Recommendation: structurally couple the barrier to code-presence by deriving `waitForBootstrap()` from a fresh codes-present check on every call, pin the singleton on `globalThis` to survive Fast Refresh, and atomicize the code-set via a single `setCodes(baseSite, lang)` API.**

Target shape (sketch — concrete details belong to the implementation brief):

```ts
// src/lib/config/apiStore.ts — target
type ApiStoreState = {
  baseSite: string | null;
  lang: string | null;
  waiters: (() => void)[];
};

const KEY = '__OGLASINO_API_STORE__';
const g = globalThis as { [KEY]?: ApiStoreState };
const state: ApiStoreState = (g[KEY] ??= { baseSite: null, lang: null, waiters: [] });

export const getCurrentBaseSiteCode = () => state.baseSite;
export const getCurrentLangCode = () => state.lang;

export function setCodes(baseSite: string, lang: string) {
  state.baseSite = baseSite;
  state.lang = lang;
  const toFire = state.waiters;
  state.waiters = [];
  toFire.forEach((r) => r());
}

export function setLangOnly(lang: string) {
  // language-switch path; codes are already set
  state.lang = lang;
}

export function waitForBootstrap(): Promise<void> {
  if (state.baseSite !== null) return Promise.resolve();
  return new Promise<void>((r) => state.waiters.push(r));
}
```

Properties that hold by construction:

- **Barrier ⇔ codes.** `waitForBootstrap()` reads `state.baseSite` on every call; it is impossible for the barrier to "resolve" while codes are null, and impossible for the barrier to "hold" while codes are set. There is no separate `resolver` field whose lifecycle could desync.
- **Atomic pair-set.** `setCodes` writes both fields and then flushes waiters in a single synchronous block. Any waiter that reads via `getCurrentBaseSiteCode()` / `getCurrentLangCode()` after being awoken sees both non-null.
- **Survives Fast Refresh.** If `apiStore.ts` is re-evaluated under Fast Refresh, the `g[KEY] ??=` assignment is a no-op (the key already exists), so `state` continues to point at the same object. The OLD interceptor closure and the NEW `AppContext.tsx` setter calls both manipulate the same single state object. (Note: `r-r` full reload is a separate JS context reset and correctly resets `globalThis`.)
- **Survives Cycle A.** `apiStore.ts` is still a leaf module — zero imports. The cycle goes through `api.ts ↔ authStore ↔ authService`, not through `apiStore`. The hardening here adds no edges.
- **No new exports beyond `setCodes` / `setLangOnly`.** Eliminates the convention coupling. `AppContext.tsx:127-128` becomes `setCodes(storedBaseSite.code, language.code)`; the base-site action's `lines 207-208` becomes the same; `setLanguageForCode`'s `line 238` becomes `setLangOnly(lang.code)`.

Alternatives evaluated:

- **Move into a zustand store.** Adds an import edge from `api.ts` into a React-tree-coupled module; gains nothing over the function-export shape and risks introducing a fresh cycle.
- **Class-instance singleton.** Cosmetic difference; function exports match surrounding code style and are mechanically identical.
- **Recreate barrier deterministically.** This IS the proposal — the freshness check replaces the recreate dance.

Complexity cost: ~+12 lines, −10 lines (deletion of old). Net wash.

---

## D2 — Render gating

**Recommendation: introduce a `<RequireBaseSite>` boundary component that wraps base-site-dependent screen content. The boundary gates inside the screen body (not the navigator), so Φ2's no-conditional-navigator constraint is fully preserved.**

Target shape:

```tsx
// src/components/context/RequireBaseSite.tsx
import { useAppState } from '@/components/context/AppContext';
import { ReactNode } from 'react';

export function RequireBaseSite({
  children,
  fallback = null,
}: {
  children: ReactNode;
  fallback?: ReactNode;
}) {
  const { selectedBaseSite } = useAppState();
  if (!selectedBaseSite) return <>{fallback}</>;
  return <>{children}</>;
}
```

Usage at screen entry points:

```tsx
// app/(portal)/(public)/index.tsx — target
export default function HomeScreen() {
  return (
    <RequireBaseSite>
      <View style={{ flex: 1 }}>
        <FilteredProductList .../>
      </View>
    </RequireBaseSite>
  );
}
```

Surfaces that need it (verified by grep for `BACKEND_API` mount-phase fetches inside the `(portal)/(public)/` tree):

- `app/(portal)/(public)/index.tsx` — home feed (`FilteredProductList`).
- `app/(portal)/(public)/catalog/[...categories].tsx` — catalog feed.
- `app/(portal)/(public)/product/[...productData].tsx` — product detail (`getProductDetails`, `getUserForId`).
- `app/(portal)/(public)/user/[...userData].tsx` — user detail (`getUserForId`).

Surfaces that do NOT need it: `about.tsx`, `blog/free-zone.tsx`, `pricing.tsx`, `privacy.tsx`, `terms.tsx` — static-content screens that do not call `BACKEND_API` on mount. (If a future content screen adds a backend call, wrap it.)

Φ2 compatibility argument (per the brief's tension):

The Φ2 constraint forbids conditionally rendering **a navigator** (`<Stack>` or `<Tabs>`). The proposal does not conditionally render any navigator: the root `<Stack>`, the portal `<Tabs>`, and the public `<Stack>` all stay always-mounted. Expo-router's auto-discovery routes are unchanged. When `selectedBaseSite` is undefined, the active screen's `<RequireBaseSite>` short-circuits to `fallback` (null or a placeholder); the screen component itself is still mounted as a navigator child, but its tree below the boundary is not. Mount-phase `useEffect`s inside `<FilteredProductList>` / `<ProductList>` / detail screens do not fire because those components do not mount.

This is **structurally identical to the pre-Φ2 gate** in semantics (children gated on bootstrap state) without re-introducing the Φ2 failure mode (conditional navigator). The brief asks: "can the feed be prevented from mounting in select-base-site state WITHOUT conditionally rendering a navigator?" The answer is **yes**, and the mechanism is per-screen content gating via the boundary.

Layout-level alternative rejected: the inner `<Stack>` in `(portal)/(public)/_layout.tsx` IS a navigator; conditionally rendering it (or its children's outer wrapper that React would see as mount/unmount) risks the same expo-router reinit pattern Φ2 forbids. Keeping the navigator untouched and gating one level deeper (inside each screen) is the only Φ2-safe placement.

Optional refinement: a `fallback={<LoadingOverlay />}` per-screen if the visual jump from `null` to full screen matters. The home screen will be visually covered by the root-level overlay during `select-base-site` regardless, so `fallback={null}` is sufficient there.

---

## D3 — Require cycles

### Cycle A — `api.ts ↔ authStore ↔ authService`

**Why it exists.** `api.ts` directly calls `useAuthStore.getState().setRestored(true)` (line 64) and `setAccountBanned(true)` (line 88) from inside response-interceptor branches. `authStore.ts` composes `login` / `register` / `logout` from `authService.ts` exports. `authService.ts` uses `BACKEND_API` from `api.ts`.

**Recommended break.** Invert the dependency at the `api.ts → authStore` edge by extracting the auth-related interceptor effects into a separate file. `api.ts` exposes a registration hook; `authInterceptors.ts` (new) imports `useAuthStore` and registers callbacks against the hook at app startup.

```ts
// src/lib/config/api.ts — target (no useAuthStore import)
type AuthInterceptorHooks = {
  onAccountRestored?: () => void;
  onAccountBanned?: () => void;
};
let authHooks: AuthInterceptorHooks = {};
export function configureAuthInterceptorHooks(hooks: AuthInterceptorHooks) {
  authHooks = hooks;
}

// in response interceptor:
if (response.headers?.['x-account-restored'] === 'true') {
  authHooks.onAccountRestored?.();
}
// ...
if (isErrorWithCode(error, 'USER_BANNED') || isErrorWithCode(error, 'EMAIL_BANNED')) {
  auth.signOut();
  authHooks.onAccountBanned?.();
  return new Promise(() => {});
}
```

```ts
// src/lib/init/authInterceptors.ts — new
import { configureAuthInterceptorHooks } from '@/lib/config/api';
import { useAuthStore } from '@/lib/store/authStore';

let registered = false;
export function registerAuthInterceptors() {
  if (registered) return;
  registered = true;
  configureAuthInterceptorHooks({
    onAccountRestored: () => useAuthStore.getState().setRestored(true),
    onAccountBanned: () => useAuthStore.getState().setAccountBanned(true),
  });
}
```

**Where to call `registerAuthInterceptors()`.** Two options:

- **At module-evaluation time from a new top-level import in `app/_layout.tsx`.** Earliest possible — the hooks are wired before any request fires. Cost: one new module that runs side-effects at import time, which is mildly atypical for this codebase but defensible because it's a single-purpose init.
- **From `AppInit` (the `status === 'ready'` gate).** Symmetric with other init wiring. Downside: pre-`ready` requests that fail with 403 USER_BANNED would not trigger the dialog, because the hooks are unregistered. Pre-`ready` requests are gated by D1's barrier (no codes ⇒ no requests fire), so in practice this window has zero relevant traffic. But the bootstrap calls themselves *can* fire and *could* return 403 if the user's token is banned mid-bootstrap (vanishingly unlikely on the cold-start path, since they'd have to sign in first). I recommend the first option — wire at module evaluation, simplest and safest.

Result: `api.ts` no longer imports `authStore.ts`. Cycle A is broken structurally. The 401 token-refresh path inside `api.ts` still calls `auth.signOut()` and `auth.currentUser` from the Firebase SDK; no `useAuthStore` dependency remains.

Complexity cost: ~+25 lines (new file), −5 lines (removed imports + calls). Net +20 lines, one fewer cycle.

### Cycle B — chat triangle

**Why it exists.** Cross-store coordination inside actions:
- `useActiveChatStore.sendMessage` reads `useChatNavStore.getState().tempReceiver` + `tempProductReason`, reads `useChatListStore.getState().chats`, calls `useChatNavStore.getState().clearChatNav()`.
- `useActiveChatStore.getActiveChat` reads `useChatListStore.getState().chats`, writes `useChatListStore.setState`, reads `useChatNavStore.getState().tempReceiver`.
- `useChatNavStore.setTempReceiver` reads `useChatListStore.getState().chats`, calls `useActiveChatStore.getState().setActiveChatId`.
- `authStore.logout` calls `clearChatListStore`, `clearActiveChat`, `clearChatNav` on all three.

The cross-calls are the actual feature workflow. Each store would shrink to a state-only slice if the cross-call action bodies moved to a coordinator (`src/lib/chat/chatActions.ts`).

**Recommendation: do not break Cycle B in this refactor.** Reasoning:

- The break is a ~200–400-line refactor that touches feature behavior surface (every component that calls `useActiveChatStore.sendMessage`, etc., would import from the new coordinator). Feature behavior is explicitly out of scope per the brief.
- The brief's primary failure mode (boot-time 400) is driven by Cycle A's coupling between `apiStore` and the interceptor + the convention-coupled barrier, NOT by Cycle B.
- Cycle B is benign at module evaluation order (all cross-calls go through `useXStore.getState()`).
- The natural place to break Cycle B is in the chat-A or chat-B mobile chat (per the Expo backlog), where feature-level test coverage will exist.

**Track Cycle B as a known structural debt to address in the chat adoption chat.** This is the brief's "flag any that can't be broken without feature-level work."

---

## D4 — Barrier's fate

**Recommendation: keep the barrier, simplified per D1, as pure defense-in-depth.**

Once D2's `<RequireBaseSite>` gate prevents the feed and other base-site-dependent screens from mounting in `select-base-site` / `loading` / `maintenance` states, the barrier should virtually never block in normal use. Its role becomes:

- **Catch deep-link cold-start cases.** A push-notification tap that cold-starts the app onto `/product/[productId]` would bypass the home-screen gate. Brief 13 named this. The barrier holds the request until bootstrap finishes, then it succeeds with valid headers.
- **Catch future screens that forget the `<RequireBaseSite>` wrap.** A new screen author who calls `BACKEND_API.get(...)` on mount without wrapping in the boundary is structurally caught by the barrier — the request waits rather than failing 400.
- **Catch race-window requests during base-site / language switch.** During the brief transition through `loading`, codes are still set from the previous ready state (per `AppContext.tsx:207-208` for base-site switch, where codes are updated *before* the status-flip), so the barrier is a no-op. No regression.

Because D1 derives the barrier from code-presence, "kept" is structurally cheaper than "removed" — there is no separate promise lifecycle to maintain. The barrier *is* `waitForBootstrap()` reading the codes. Removing it would require deleting the await in `api.ts:46-48` and trusting the per-screen gate exhaustively, which is a less safe trajectory.

Optional refinement worth considering in the implementation brief (not required): in dev-only mode, log a warning when the barrier actually blocks. This surfaces "missed render gates" early, before they reach production.

---

## D5 — Component guard on ProductList

**Recommendation: yes, add the component guard as defense-in-depth on `ProductList.tsx:101-103`'s fetch effect.**

Concrete shape:

```ts
useEffect(() => {
  if (!selectedLanguage) return;
  onRefresh();
}, [fetchPage, selectedLanguage]);
```

With D2 in place, `ProductList` does not mount while `selectedBaseSite` is undefined. By the time the component mounts, both `selectedBaseSite` and `selectedLanguage` are guaranteed non-null (`AppContext.tsx:127-131` sets codes and stores both before transitioning to `'ready'`; `selectedLanguage` is set on the state at line 137). So this guard *should* be a no-op in normal operation.

It earns its place because:

- It's two lines.
- It survives a future refactor that removes or changes the `<RequireBaseSite>` boundary.
- It's the right shape regardless: the fetch effect's pre-condition is "we have a language to fetch with."
- Brief 16's Mechanism A (Fast Refresh state retention) — if it ever triggers in dev — would skip the barrier; the component guard would still catch a null `selectedLanguage`.

The `[fetchPage, selectedLanguage]` dependency array also handles language switches correctly (re-fetch on language change), which is current behavior maintained.

---

## D6 — Migration shape

The implementation brief should land in this order. Each step is independently testable; an engineer can stop after any step and leave the app in a working state.

### Step 1 — Boot-state hardening (D1)

- Refactor `src/lib/config/apiStore.ts` to the target shape: `globalThis`-pinned singleton, codes-derived `waitForBootstrap`, atomic `setCodes` + `setLangOnly` APIs. Delete `bootstrapResolve` / `bootstrapPromise`.
- Update `AppContext.tsx:127-128` to `setCodes(storedBaseSite.code, language.code)`.
- Update `AppContext.tsx:207-208` to `setCodes(site.code, lang.code)`.
- Update `AppContext.tsx:238` to `setLangOnly(lang.code)`.
- Update `api.ts:46` barrier condition — semantically unchanged (`!getCurrentBaseSiteCode()`), but verify the new `waitForBootstrap` shape is awaitable.

**Verifiable in isolation.** Existing tests pass. Manual smoke on cold start (stored base site path) — feed loads. Smoke on `select-base-site` path — feed is no longer rendered yet (covered in Step 3), but `[DBG]` shows the barrier is held correctly.

### Step 2 — Break Cycle A (D3a)

- New `src/lib/init/authInterceptors.ts` with `registerAuthInterceptors()`.
- Refactor `api.ts` — remove `useAuthStore` import, add `configureAuthInterceptorHooks` export, replace direct `useAuthStore.getState()` calls with `authHooks` callbacks.
- Add `import '@/lib/init/authInterceptors';` plus `registerAuthInterceptors()` call from a deterministic-early call site (recommend a tiny `src/lib/init/registerInterceptors.ts` imported by `app/_layout.tsx` at top, or a top-level side-effect import).

**Verifiable in isolation.** Existing tests pass; Metro warning count drops by one (only Cycle B remains). Manual smoke: account-banned flow still triggers dialog; account-restored header still flips `restored` state.

### Step 3 — Per-screen render gate (D2)

- New `src/components/context/RequireBaseSite.tsx`.
- Wrap content of: `(portal)/(public)/index.tsx`, `(portal)/(public)/catalog/[...categories].tsx`, `(portal)/(public)/product/[...productData].tsx`, `(portal)/(public)/user/[...userData].tsx`.
- For each wrap, the screen's mount-phase fetches are now downstream of `<RequireBaseSite>`.

**Verifiable in isolation.** Cold start with no stored base site: barrier holds, intro overlay shows, no `/public/product/search` fires. After base-site selection: feed mounts and loads.

### Step 4 — Component guard (D5)

- `ProductList.tsx:101-103` — add `if (!selectedLanguage) return;` and `selectedLanguage` to deps.
- Confirm no regression on base-site / language switch.

**Verifiable in isolation.** Existing language-switch path still re-fetches.

### Step 5 — Cleanup pass

- Remove `[DBG]` log block in `api.ts:36-45` (Brief 14 cleanup miss).
- Remove the `TODO: Translations PLUS after meintanence page do better check` in `AppVersionConfigInit.tsx:29` (low-priority; engineer's call whether to bundle here or defer).

### Step 6 (deferred, separate chat) — Break Cycle B

Belongs to the chat-A or chat-B mobile chat. Out of scope for the boot refactor.

### Bundling guidance for the engineer

- Steps 1 + 2 can land in one engineer session if the engineer prefers — both are small, both are HTTP-layer plumbing, and their tests are similar. Their combined diff size is ~80 lines net.
- Step 3 should be its own session because it changes user-visible behavior on the intro path and warrants a focused smoke.
- Step 4 + Step 5 can be folded into Step 3 if the engineer wants to close out the cleanup in one go.

---

## D7 — Risk + verification

### Risks

- **`globalThis` pinning may behave unexpectedly under r-r (full Metro reload).** `r-r` resets the JS context; the new context creates a fresh `globalThis`. Codes reset to null — correct behavior, matches a true cold start. Verify with the smoke checklist below.
- **Fast Refresh edge case: editing `api.ts` re-evaluates the file but the previously-attached interceptor closure on `BACKEND_API` is gone.** Metro creates a new `BACKEND_API` instance; the new instance has new interceptors that read the same `globalThis`-pinned codes. Expected: no regression. The risk is theoretical and applies regardless of the design — current code has the same property.
- **Per-screen gate coverage drift.** A new screen author misses the `<RequireBaseSite>` wrap. The barrier (D4) catches it as a held request. In dev, the optional dev-mode warning surfaces the miss earlier. Long-term mitigation: convention in the brief and a `meta/conventions.md` line under Part 8 if Mastermind decides to formalize.
- **Maintenance-state polling interaction.** The 5s polling effect in `AppContext.tsx:162-184` calls `checkIfMaintenance()` continuously. These are bootstrap-flagged requests (line 6 of `maintenanceService.tsx` has `_bootstrap: true`), so they bypass the barrier. Codes are set on the path where polling detects "maintenance cleared" (it calls `bootstrap()` which calls `setCodes` on success). No new risk.
- **Φ2 tab-state preservation.** Untouched. Root `<Stack>` and portal `<Tabs>` remain always-mounted. The `<RequireBaseSite>` boundary lives one render level below the navigator tree.
- **Cycle A break: race between interceptor firing and `registerAuthInterceptors()`.** If a 403 USER_BANNED somehow fires before registration completes, `authHooks.onAccountBanned?.()` is a no-op. The user does not see the banned dialog. Mitigation: register at module-import time (top of `app/_layout.tsx`), which executes before any request can fire because `BACKEND_API` is itself constructed at module-import time and no request fires until React mounts. The interceptor only triggers from in-flight requests, which need React to be alive. Window is effectively zero on cold start.

### Smoke verification (must pass post-refactor)

1. **Cold start with stored base site.** App opens, bootstrap reads stored site, codes set via `setCodes`, status goes `loading` → `ready`, feed mounts, `POST /public/product/search` fires with valid `X-Base-Site` and `X-Lang` headers, returns 200.
2. **Cold start with no stored base site (intro / select-base-site path).** App opens, bootstrap reaches `select-base-site` status, intro overlay renders. `POST /public/product/search` does NOT fire (`<RequireBaseSite>` short-circuits the home screen). User taps a country → `setBaseSiteForCode` → `setCodes` → status `ready` → feed mounts → request fires with valid headers, 200.
3. **First-run user enters maintenance.** Bootstrap throws or maintenance flag → status `maintenance` → overlay shows → no feed mount → no requests fire (no codes ⇒ barrier holds anything that slips through). Polling detects maintenance cleared → bootstrap re-runs → codes set → feed loads.
4. **Returning user, normal cold start.** Same as case 1 (stored base site path).
5. **Base-site switch.** User in `ready` state taps another country in the country switcher. `setBaseSiteForCode` updates codes synchronously, status briefly `loading` then `ready`, feed re-renders with new headers. Codes never null during the transition. Barrier never blocks.
6. **Language switch.** `setLanguageForCode` calls `setLangOnly`, status briefly `loading` then `ready`. `selectedBaseSite` stays defined; feed stays mounted. `ProductList`'s language-effect (`ProductList.tsx:119-133`) triggers `onRefreshCurrent`. Headers carry new lang.
7. **Maintenance detected mid-session.** Polling flips status to `maintenance`. Overlay covers. No new requests fire. (`<RequireBaseSite>` keeps the feed mounted because `selectedBaseSite` is still defined — that's a trade-off; the feed component is alive but no longer fetches because the maintenance overlay is on top. If this matters, the boundary can be extended to gate on status too. Recommend not extending in v1 — overlay covers UX and the only cost is one already-fetched feed in memory.)
8. **Φ2 tab-state preservation.** Navigate portal → owner → portal. Portal feed scroll position preserved. Confirmed in Φ2 manual smoke; this refactor does not regress.
9. **Account-banned mid-session.** 403 USER_BANNED returns; `authHooks.onAccountBanned()` fires; banned dialog opens via the existing `AccountStateDialogsInit` subscription. (Cycle-A break smoke.)
10. **Token-refresh 401.** Single-flight 401 refresh path in `api.ts:93-131` still works. (Cycle-A break does not touch this path.)
11. **Fast Refresh.** Edit `apiStore.ts`, edit `api.ts`, edit `ProductList.tsx` — app continues, no 400s, no held-forever requests, codes stay set.
12. **`r-r` full reload.** Full reload resets state to cold start. Behavior follows cases 1, 2, or 3 depending on stored data.

---

## Recommended target architecture — single decision

**Adopt all of D1, D2, D3a, D4, D5. Defer D3b (chat triangle) to a chat-feature chat.**

In one sentence: the boot/HTTP layer becomes a `globalThis`-pinned codes singleton whose barrier is structurally derived from code-presence; base-site-dependent screens gate their content via a `<RequireBaseSite>` boundary so the feed cannot mount on intro; the `api.ts ↔ authStore` cycle is broken by extracting interceptor-side auth effects into a registration hook; the component-level `ProductList` guard stays as defense-in-depth; the HTTP-layer barrier stays as a safety net for deep links and missed gates.

This satisfies the brief's "structural, not conventional" requirement (the barrier and the codes share one storage cell; nothing else can desync them), Constraint 1 (the feed and other base-site-dependent surfaces do not mount during `select-base-site`), Constraint 2 (returning-user path is unchanged — `setCodes` happens before status flips to `ready`, so the gate passes immediately), Φ2's no-conditional-navigator rule (all navigators stay always-mounted; only screen content is gated), and Φ2's tab-state-preservation invariant (the navigator tree is untouched).

Rejected target alternatives (briefly):

- **Roll back to pre-Φ2 `<Slot />` at root.** Reintroduces F12, regresses Φ2 (tab-state loss on portal ↔ owner).
- **HTTP-layer barrier only, no render gate.** Brief 16 showed the barrier alone is fragile; product feed mounting during intro is structurally wrong regardless of whether the barrier holds the request.
- **Render gate only, no HTTP barrier.** Removes defense-in-depth; a single missed wrap causes a 400. Cost of keeping the barrier is ~12 lines.
- **Big-bang refactor.** The work decomposes cleanly into five independently verifiable steps; landing them incrementally lowers risk and is more reviewable.

---

## Closure gate

This session's recommended config-file edits: none required.

- `conventions.md`: no change required. If Mastermind decides to formalize the "wrap base-site-dependent screens in `<RequireBaseSite>`" convention into Part 8, that draft will come from Mastermind, not from this design brief.
- `decisions.md`: no change required now. The implementation brief's close-out will draft a `decisions.md` entry. A pre-staged title is suggested above under "Config-file impact" for Mastermind's convenience; not a draft.
- `state.md`: no change. The work is implicit in the existing Expo structural foundation trajectory.
- `issues.md`: no change. The two adjacent observations flagged in Part 4b will be cleaned up by the implementation brief.

The "no implicit config-file dependency" check passes.
