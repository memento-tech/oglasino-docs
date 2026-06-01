# Expo Structural Foundation

**Status:** `in-progress`
**Owner:** Igor
**Scope:** mobile (`oglasino-expo`) only. Four sequenced Mastermind chats that establish the structural floor before feature-adoption chats A–I open.

This document is the canonical output of the structural-audit Mastermind chat (2026-05-24) that ran a read-only structural audit of `oglasino-expo` and produced 26 findings across seven categories. It is a planning artifact, not a feature spec in the usual sense — no single engineer implements it. Each Φ chat below opens its own Mastermind chat per `meta/conventions.md` Part 10, with its own intake, audit (where needed), and briefs.

The chat that produced this document closes when this file is committed.

## 1. Why this document exists

The structural audit rated `oglasino-expo` at **5.5/10**. The app functions but is structurally wrong for the platform in three dominant areas:

1. **Lifecycle gaps.** The auth listener (`initAuthListener`) is defined but never called — the app relies on stale AsyncStorage persistence. `logoutFirebase` doesn't call `auth.signOut()`, so Firebase sessions persist after logout. There is no 401/403 response interceptor, no foreground auth re-validation, and no network state detection. Banned or deleted users continue using the app with stale credentials.

2. **Performance shape.** Zero `React.memo` across the entire codebase. All FlatList `renderItem` props are inline arrow functions. `expo-image` is installed but never imported — all 14+ image sites use RN `Image` with no disk caching. All Zustand subscriptions are whole-store (no selectors, no `useShallow`). The AppContext value is rebuilt on every render without memoization.

3. **RN convention violations.** Every layout uses bare `<Slot />` — no native `Stack` or `Tabs` navigators. The app has no native transitions, no swipe-back gesture, no tab state preservation. The message list is not inverted. There are `<div>` elements in RN code.

These issues cross-cut the existing A–I feature-adoption queue. A foundation pass is needed before feature chats open because: feature chats would build on broken lifecycle foundations (requiring rework), performance fixes need holistic re-render analysis (not per-feature), and navigation rearchitecture affects every screen's transition and header rendering.

## 2. Audit source

The full audit lives at [`sessions/2026-05-24-oglasino-expo-structural-audit-1.md`](../sessions/2026-05-24-oglasino-expo-structural-audit-1.md) and [`sessions/audit-expo-structural.md`](../sessions/audit-expo-structural.md). Future chats in this program read the audit directly. Do not re-derive findings from this spec — go to the audit.

## 3. The four foundation chats

### Φ1 — Auth lifecycle foundation

**Scope:** F1 (initAuthListener never called), F2 (logoutFirebase signOut gap), F3 (store cleanup on logout), F4 (no foreground auth re-validation), F5 (no 401/403 interceptor), F13 (no auth guards on secured routes), F22 (hydration race), Q2 (verify USER_BANNED response shape, cross-repo — small parallel backend audit chat).

**Why first:** every other chat depends on auth being correct. Lifecycle findings are foundational security and trust failures, not polish.

**Inputs:** the structural audit; [`features/user-deletion.md`](user-deletion.md) and [`features/user-deletion-auth-contract.md`](user-deletion-auth-contract.md); the answers to Q1, Q2, Q3 from the parallel backend audit chat.

**Expected outcome:** auth listener wired correctly with single-flight guard, logout fully tears down state, foreground revalidation works on extended-background resumes, banned/deleted users are forced out cleanly with a ban-notice dialog, secured routes guard at the layout level.

**Bucket-1 bugs that ride along:** B1 (closed by F2 fix).

**Effect on downstream chats:** chat E becomes much smaller — most of E was auth plumbing.

### Φ2 — Navigation foundation

**Scope:** F12 (replace all `<Slot />` with `Stack` and `Tabs`), F13 (auth guards integrate with navigators), F26 (`useWindowDimensions` instead of module-scope `Dimensions.get`), decision on native vs custom TopBar.

**Why second:** feature chats need to know the navigation shape they're building inside. Doing this after feature chats would require rework.

**Inputs:** the structural audit; the existing custom BottomBar and TopBar implementations for reference.

**Expected outcome:** bottom bar becomes a `<Tabs>` navigator with state preservation. Screen-to-screen navigation is `<Stack>` with native transitions and iOS swipe-back. Tab switches do not remount screens. Routes are auth-guarded at the layout level.

### Φ3 — Performance foundation

**Scope:** F7 (Zustand selectors + useShallow across all 22+ subscription sites), F8 (React.memo on ProductCard, ExtraProductCard, ProductReview, Message, chat-list items, plus stable useCallback for renderItem), F9 (expo-image adoption — replace all 14+ RN Image imports), F10 (AppContext value memoization, useCallback on actions), F20 (CategorySelector and CitySelector to FlatList/SectionList), F25 (scrollEventThrottle on ProductList, consider Reanimated for scroll-driven UI), F23 (split useChatStore into focused stores).

**Why third:** needs navigation in place (Tabs persistence changes re-render math) but doesn't depend on full lifecycle work beyond Φ1.

**Inputs:** the structural audit; the existing useChatStore for splitting; the existing 14+ Image import sites for expo-image migration.

**Expected outcome:** product feed scrolls smoothly on mid-range Android. List items only re-render when their data changes. Images cache to disk. Store updates only re-render components consuming the changed slice.

**Note on P3:** `expo-image` adoption was originally deferred to chat H. It moves into Φ3 here because images affect the product-feed re-render math.

### Φ4 — Service layer + error contract foundation

**Scope:** F18 (implement Part 7 error contract across all 17 services), F24 (reportService error-field inversion bug), F6 (NetInfo install + offline vs maintenance distinction).

**Why fourth:** this is the seam between mobile and backend. Doing it before chat A makes A possible — A is essentially "wire backend validation errors into the UI," which requires the service layer to surface those errors first.

**Inputs:** the structural audit; `meta/conventions.md` Part 7 (error contract); web's service layer as the reference shape.

**Expected outcome:** every service surfaces structured backend errors. UI can map field-level errors to UI state. Offline state shows offline UI, not maintenance UI.

## 4. Revised queue order

After Φ1–Φ4, chats A–I open in the order below. Most are smaller than originally scoped because foundation work absorbed cross-cutting concerns. Final polish closes with Ω and Ψ.

| # | Chat | Scope change from foundation |
|---|---|---|
| Φ1 | Auth lifecycle foundation | New — see section 3 |
| Φ2 | Navigation foundation | New — see section 3 |
| Φ3 | Performance foundation | New — see section 3 |
| Φ4 | Service layer + error contract foundation | New — see section 3 |
| A | Mobile validation rebuild | Slightly smaller — Φ4 delivers the Part 7 service layer A depends on |
| B | Mobile messaging adoption | Unchanged — messaging-specific scope |
| C | Mobile user-settings cleanup + consent-field removal | Unchanged |
| D | Mobile filtering & search polish | Unchanged |
| E | Mobile user-deletion adoption | Massively smaller — Φ1 delivers auth listener, logout fix, 401/403 interceptor, auth guards, hydration race fix. E retains only the deletion UI surface (danger zone, deletion-in-flight flag) |
| H | Mobile image-pipeline polish | Somewhat smaller — Φ3 delivers expo-image adoption (P3). H retains image upload UX, stale DTO cleanup, hardcoded strings |
| G | Mobile consent UI v1 | Unchanged |
| F | Mobile analytics (GA v1) | Unchanged |
| I | Mobile backend-calls optimization | Unchanged |
| Ω | Final structural sweep + dead-code removal | New — see section 5 |
| Ψ | Runtime verification pass | New — see section 5 |

## 5. The new Ω and Ψ chats

### Ω — Final structural sweep + dead-code removal

Scope drawn from the audit's "Out-of-scope observations" section. This chat runs after all feature chats close and sweeps the codebase for remaining structural debt:

- `react-dom` in production dependencies (~130KB dead weight if no Expo web)
- `dotenv` in production dependencies (Node.js module, doesn't run on RN)
- Dead service functions: 12 exported functions with zero consumers (`getUserDetails`, `updateUserAuth`, `updateUser`, `getUserFollows`, `getUserForId`, `getPortalProductDetails`, `getDashboardProductDetails`, `getPortalAutocompleteSuggestions`, `getDashboardAutocompleteSuggestions`, `getDashboardProducts`, `markAsSeen`, `healthCheck`)
- `healthCheckService.ts` — entirely dead file
- `loginWithFacebookFirebase` — dead stub (entire body commented out)
- `cookie/` directory — 3 web-only dead files (`GlobalCookie.ts`, `ConsentData.ts`, `UserPreference.ts`)
- `UserMenu.tsx` broken paths (`/dashboard/balance`, `/dashboard/account-verification` — should be `/owner/dashboard/...`)
- `setActiveChatId(undefined)` type mismatch — signature expects `string | null`, receives `undefined`
- Chat list FlatList has no pagination wired — `loadMoreChats()` never called, >15 chats invisible
- `KeyboardAvoidingView` behavior inconsistency (`'padding'` in messages, `'height'` in dialogs)
- Store directory rename — `src/lib/store/` vs `src/lib/stores/` (dual naming)
- `var` → `const`/`let` in `useChatStore.ts:454` and `user.tsx:178`
- `getUniqueID` imported from `react-native-markdown-display` — replace with `crypto.randomUUID()` or uuid
- Three `.tsx` service files contain no JSX (`configurationService.tsx`, `appVersionConfigService.tsx`, `maintenanceService.tsx`)
- Input-heavy forms without `KeyboardAvoidingView` (user settings, product edit)

### Ψ — Runtime verification pass

Not a code chat. A QA-style chat with a brief to run the app on real devices (Pixel 4a or equivalent mid-range Android plus iPhone) and verify that structural fixes produced intended runtime behavior. Findings the audit could not verify without runtime:

- **F6 — Maintenance-vs-offline.** Verify that airplane mode shows an offline indicator (not the maintenance screen) after Φ4's NetInfo integration.
- **F7 — Re-render storms.** Run a React DevTools profiler trace on the product feed after Φ3. Quantify re-renders per scroll frame, per filter change, per store update. Confirm list items skip re-render when their data hasn't changed.
- **F22 — Hydration race.** Measure cold-start timing after Φ1. Confirm init components wait for `_hasHydrated` before reading `user`. Confirm no double-initialization of chat/notification/favorites listeners.
- **F12 — Navigation transitions.** Verify native slide transitions between screens, iOS swipe-back gesture, tab state preservation across tab switches.
- **F9 — Image disk caching.** Scroll through product feed, kill app, reopen, scroll back. Confirm images load from disk cache (no network re-fetch).
- **General performance.** JS thread FPS on mid-range Android during product feed scroll. Memory profile over a 20-minute session.

## 6. Risk watch additions for state.md

The following entries appear in `state.md`'s Risk Watch section (applied by this session):

- **Mobile lifecycle defects are foundational security/trust issues.** Auth listener never invoked, `logoutFirebase` doesn't call `auth.signOut()`, no 401/403 interceptor. Closed when Φ1 ships. Reference: structural audit F1, F2, F5.
- **Mobile is structurally a web SPA in a native shell.** All layouts use bare `<Slot />`. Closed when Φ2 ships. Reference: structural audit F12.
- **Product feed performance is wrong for mid-range Android.** Zero React.memo, whole-store subscriptions, expo-image unused. Closed when Φ3 ships. Reference: structural audit F7, F8, F9, F10, F25.
- **Mobile service layer silently swallows backend validation errors.** No Part 7 error contract. Closed when Φ4 ships. Reference: structural audit F18.
- **Cross-repo questions queued from structural audit.** Q1, Q2, Q3. Q2 blocks Φ1. Close when all three answered.

## 7. What this program does NOT do

- No feature adoption. Feature adoption is the A–I queue.
- No new features. Foundation only.
- No backend or web changes except the parallel backend audit chat answering Q1, Q2, Q3.

## 8. Definition of done for the program

- All four Φ chats closed with their scopes shipped.
- All four Risk Watch entries from section 6 closed.
- A–I queue opens against a structurally correct foundation.

## 9. Sequencing summary

| # | Chat | Status |
|---|---|---|
| Φ1 | Auth lifecycle foundation | `shipped` (code) / `verifying` (manual smoke pending on real devices per spec §10) — spec at [`expo-auth-lifecycle.md`](expo-auth-lifecycle.md) |
| Φ2 | Navigation foundation | shipped (code) / verifying (smoke pending) — spec at [expo-navigation-foundation.md](expo-navigation-foundation.md) |
| Φ3 | Performance foundation | `shipped` — cold-start boot-loading hang resolved by the `expo-boot-redesign` feature (see [decisions.md](../decisions.md) 2026-05-29); real-device regression passed. Spec at [expo-performance-foundation.md](expo-performance-foundation.md) |
| Φ4 | Service layer + error contract foundation | `planned` |
| A | Mobile validation rebuild | `planned` — blocked on Φ4 |
| B | Mobile messaging adoption | `planned` |
| C | Mobile user-settings cleanup | `planned` |
| D | Mobile filtering & search polish | `planned` |
| E | Mobile user-deletion adoption | `planned` — massively smaller after Φ1 |
| H | Mobile image-pipeline polish | `planned` — somewhat smaller after Φ3 |
| G | Mobile consent UI v1 | `planned` |
| F | Mobile analytics (GA v1) | `planned` — depends on G or ATT-only decision |
| I | Mobile backend-calls optimization | `planned` |
| Ω | Final structural sweep + dead-code removal | `planned` |
| Ψ | Runtime verification pass | `planned` — last |
