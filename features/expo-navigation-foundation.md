# Expo Navigation Foundation (Φ2)

**Status:** `shipped` (code) / `verifying` (manual smoke pending per spec §8)
**Owner:** Igor
**Repo:** `oglasino-expo` only
**Branch:** `dev`
**Depends on:** Φ1 shipped (`features/expo-auth-lifecycle.md` — auth listener, logout fix, 401/403 interceptor, three auth-guarded layouts, hydration race resolved)
**Audit source:** `sessions/audit-phi2-navigation.md` (2026-05-25)
**Program context:** second of four foundation chats per `features/expo-structural-foundation.md`

## 1. What this feature does

Φ2 replaces every bare `<Slot />` in `oglasino-expo` with native `<Stack>` and `<Tabs>` navigators, **underneath** the existing custom UI (`BottomBar.tsx`, `TopBar.tsx`, `DashboardSidebar.tsx`). The visible design does not change. The structural code underneath changes from React-only to native-navigator-backed.

What users gain: native stack transitions and iOS swipe-back gesture on every screen-to-screen navigation; Android hardware back button behavior; tab state preservation across tab switches (each tab remembers its own scroll position, fetched data, and current screen); proper deep-link composition through layout trees; native screen lifecycle management.

What does not change: the visual design of BottomBar, TopBar, DashboardSidebar, and every screen body. The auth-guard behavior shipped in Φ1 (three `(secured)`/`owner` layouts using `<Redirect>` based on `useAuthStore(s => s.user)` + `_hasHydrated`). The provider stack at the root layout.

## 2. Scope

In scope:

- F12 — replace `<Slot />` in all six layouts with native `<Stack>` and `<Tabs>` navigators per section 3
- F13 (integration with navigators) — verify Φ1's auth guards survive the navigator swap (verified from expo-router source in audit Section 5)
- F26 — replace module-scope and component-body `Dimensions.get('window')` with `useWindowDimensions()` in the four sites identified by the audit
- Wiring BottomBar as the `tabBar` prop of `<Tabs>` and changing its navigation calls from `router.push()` to `navigation.navigate()`
- Wiring DashboardSidebar's navigation from `router.push()` to `router.replace()` to prevent stack accumulation in the dashboard
- Setting `headerShown: false` globally on all `<Stack>` and `<Tabs>` navigators so the custom TopBar remains the visual chrome
- Restructuring `app/` directory as needed for nested-navigator tab structure (engineer chooses exact shape)
- `+not-found.tsx` configured to render the custom TopBar pattern (`headerShown: false`, no native title)

Out of scope:

- Messages chat-list-to-detail split into separate routes (deferred to chat B per Fork D)
- UserMenu broken paths (`/dashboard/balance` instead of `/owner/dashboard/balance`) — deferred to Ω
- KeyboardAvoidingView `behavior` inconsistencies across the codebase — deferred to Ω
- Visual redesign of BottomBar, TopBar, DashboardSidebar — explicit non-goal
- Backend changes — none
- Cross-repo work — none

## 3. The navigator architecture (the key decision)

**Pattern: nested navigators with `<Tabs>` at portal level and `<Stack>` per tab and at owner level.**

### 3.1 Root layout — `app/_layout.tsx`

Becomes `<Stack screenOptions={{ headerShown: false }}>` rendered always, regardless of `status`. The Stack is no longer conditionally mounted alongside the loading / maintenance / select-base-site branches — those states render as absolute-positioned full-screen overlays on top of the always-mounted Stack.

Why: expo-router's `<Stack>` registers a navigator with the navigation container. Conditionally mounting and unmounting it (e.g., gating it on `status === 'ready' || status === 'loading'`) causes the navigation container to reinitialize on every mount, which remounts the root layout, which resets `status` back to its initial value — producing an infinite bootstrap loop. The bare `<Slot />` used before Φ2 did not register a navigator, so the conditional-rendering pattern worked. Switching to `<Stack>` made it incompatible. See `decisions.md` 2026-05-27 "Expo-router `<Stack>` cannot be conditionally rendered."

Concretely:

- `<Stack>` always rendered as a child of the SafeAreaView.
- `<AppInit>` mounts only when `status === 'ready'`.
- Loading overlay, maintenance overlay, and base-site-selector overlay are absolute-positioned full-screen views rendered on top of the Stack when their respective status applies.
- `PortalHost` and `DialogManager` remain as siblings at the root.

Provider stack outside the Stack remains identical: `GestureHandlerRootView` → `AppContextProvider` → `AppVersionConfigInit` → `ToastProvider` → `ThemeProvider` → `SafeAreaView`.

The new `<Stack>` configures each top-level route group's screen with `<Stack.Screen options={{ headerShown: false }}>` (or relies on the screenOptions default).

### 3.2 Portal layout — `app/(portal)/_layout.tsx`

Becomes `<Tabs tabBar={(props) => <BottomBar {...props} />} screenOptions={{ headerShown: false }}>`.

The portal layout's existing chrome (`ConsumerProtectionBanner`, `TopBar` portal variant, `CategoryNavigation`) remains as siblings rendered above the `<Tabs>`. BottomBar is no longer a sibling rendered below — it becomes the `tabBar` render prop of `<Tabs>`.

The tabs the portal layout exposes correspond to the four BottomBar destinations:

- Home tab — composed of the `(public)/` route group, wrapped in its own `<Stack>` (see 3.3)
- Favorites tab — `(secured)/favorites.tsx`
- Notifications tab — `(secured)/notifications.tsx`
- Messages tab — `(secured)/messages.tsx`

Each tab is a `<Tabs.Screen>` entry. The Favorites, Notifications, and Messages tabs do not need their own internal `<Stack>` in Φ2 because they are single-screen surfaces. (Messages remains single-screen per Fork D — the chat-list-to-detail toggle stays state-based for now.)

**`selectedBaseSite` guard requirement.** Because the root Stack is now always mounted (per §3.1), the portal layout renders before `AppContext` bootstrap completes. At that point `selectedBaseSite` is undefined and the chrome above the Tabs (`TopBar`, `CategoryNavigation`, `ConsumerProtectionBanner`) crashes when it reads `selectedBaseSite.iconId` or similar fields. The portal layout therefore guards the chrome rendering on `selectedBaseSite` being defined. The `<Tabs>` navigator itself stays always mounted — only the chrome above it is guarded.

### 3.3 Public route group — `app/(portal)/(public)/_layout.tsx`

Becomes `<Stack screenOptions={{ headerShown: false }}>`. This is the Home tab's stack. It composes the home-tab-reachable screens:

- `index.tsx` (home)
- `catalog/[...categories].tsx`
- `product/[...productData].tsx`
- `user/[...userData].tsx`
- `about.tsx`, `pricing.tsx`, `privacy.tsx`, `terms.tsx`
- `blog/free-zone.tsx`

The existing `setPortalScope('portal')` side effect remains.

Navigating to a product detail from the home screen now pushes onto this Stack. The user gets a native slide-in transition and iOS swipe-back. Switching to a different tab preserves this Stack's state. Returning to the Home tab lands on the same screen the user was viewing.

### 3.4 Secured route group — no layout file

The `(secured)` route group has no `_layout.tsx`. The directory exists as a transparent route group; its three screens (`favorites.tsx`, `notifications.tsx`, `messages.tsx`) are hoisted by expo-router into the portal `<Tabs>` navigator as direct tab children.

Each of the three screens carries its own inline auth guard at the top of its function body, matching the pattern Φ1 established in the layout files:

```tsx
const user = useAuthStore((s) => s.user);
const hasHydrated = useAuthStore((s) => s._hasHydrated);

if (!hasHydrated) return null;
if (!user) return <Redirect href="/" />;
```

**Why no layout file:** expo-router's `Tabs` (and `Stack`) navigator collapses a route group with a `_layout.tsx` into a single navigator screen. With `(secured)/_layout.tsx` present, the portal `<Tabs>` would see two children (`(public)` and `(secured)`) instead of the four required tab destinations (Home / Favorites / Notifications / Messages). The constraint was discovered during Φ2 Brief 1 implementation; the inline-guard pattern is the resolution. See `decisions.md` 2026-05-25 "Expo-router route-group layout files" for the full constraint.

**Auth guard semantics preserved.** Φ1's F13 verification (audit Section 5) confirmed that `<Redirect>` returns `null` and uses `useFocusEffect` to call `router.replace()`. Whether the guard lives in a layout or inline in each screen, the behavior is identical: the child content never mounts when `user` is null. The three inline guards are functionally equivalent to the original single-layout guard, distributed across three sites.

### 3.5 Owner layout — `app/owner/_layout.tsx`

Auth guard from Φ1 remains unchanged. After the guard, the layout returns `<Stack screenOptions={{ headerShown: false }}>`. The Stack composes the dashboard surface.

Existing chrome (marketplace label, TopBar dashboard variant, DashboardSidebar) remains as siblings around the Stack. DashboardSidebar continues to use absolute positioning for its slide-out panel.

### 3.6 Dashboard route group — `app/owner/dashboard/_layout.tsx`

Auth guard from Φ1 remains unchanged. After the guard, the layout returns `<Stack screenOptions={{ headerShown: false }}>` instead of `<Slot />`. The Stack composes the dashboard screens (user, products list, product detail, balance, follows, reviews, analytics, account-verification, not-ready).

### 3.7 The `+not-found.tsx` screen

Per Fork B, this screen uses the custom TopBar pattern. The existing `<Stack.Screen options={{ title: 'Oops!' }} />` at line 7 is removed. The screen inherits `headerShown: false` from the root Stack's screenOptions.

If the not-found screen does not already render TopBar, the engineer adds it for visual consistency with the rest of the app.

### 3.8 Directory restructuring

The engineer chooses the exact file layout that composes best with expo-router's group syntax. The constraint is the navigator architecture above; the file paths may shift if expo-router conventions require it. The engineer documents directory changes in the implementation brief's session summary.

## 4. Component changes

### 4.1 BottomBar — `src/components/navigation/BottomBar.tsx`

Becomes a component that accepts `BottomTabBarProps` from React Navigation. The visual rendering is unchanged. The behavior changes are:

**Active tab detection.** Replaces `usePathname()` + `pathname.startsWith()` with reading from the `state` prop:

```tsx
const currentRoute = props.state.routes[props.state.index];
// e.g., currentRoute.name === 'favorites'
```

The `getCurrentColor` function is rewritten to take a route name and compare against the current tab.

**Tab navigation.** Replaces `router.push('/favorites')` with `props.navigation.navigate('favorites')` (route name, not URL path).

**Auth gate (`openDialogSafe`).** Preserved. The gate runs before `navigation.navigate()` fires. The `_hasHydrated` and `user` reads remain via `useAuthStore` selectors.

**Non-tab elements.** The center "new product" button and the user-menu trigger are not tabs. They remain rendered inside BottomBar and dispatch to dialogs as today. They do not appear in `state.routes`. This is intentional: the navigator's tab list covers only Home, Favorites, Notifications, Messages; the new-product button and user menu are action triggers that happen to share the bar's visual space.

**Store subscriptions.** The whole-store subscriptions noted in the audit (auth, favorites, notification, chat) remain a Φ3 concern. Φ2 does not introduce selectors. (Engineers may use selectors if a change touches the line, but it's not in Φ2's scope to refactor existing subscriptions.)

### 4.2 TopBar — `src/components/navigation/TopBar.tsx`

No changes. Continues to render inside the portal and owner layouts as a sibling above the `<Stack>` or `<Tabs>` content area. The `<Stack>`'s `headerShown: false` prevents the native header from competing.

### 4.3 DashboardSidebar — `src/components/dashboard/layout/DashboardSidebar.tsx`

Navigation calls change from `router.push(url)` to `router.replace(url)` per Fork C. This prevents back-stack accumulation when the user navigates through dashboard screens via the sidebar.

The component-body `Dimensions.get('window').width` at line 40 is replaced with `useWindowDimensions().width` per F26. The animation target updates correctly when dimensions change.

### 4.4 Module-scope `Dimensions.get('window')` replacements (F26)

Four sites:

- `src/components/FloatingButton.tsx:6` — module scope; replace with `useWindowDimensions()` inside the component
- `src/components/FullScreenImageViewer.tsx:8` — module scope; same fix
- `src/components/product/ProductList.tsx:23` — module scope; same fix
- `src/components/dashboard/layout/DashboardSidebar.tsx:40` — component-body but non-reactive; same fix (also part of 4.3)

The app remains portrait-locked per `app.config.ts`. The risk is latent (foldables, iPad split-screen) but the fix is correct regardless.

## 5. The 14 seams and their resolutions

Each seam from the audit (Section 9) maps to the component/layout work above.

| Seam | Resolution location |
|------|---------------------|
| 1 — BottomBar active tab via `usePathname` | 4.1 (use `state.routes[state.index]`) |
| 2 — BottomBar mounted as sibling | 3.2 + 4.1 (become `tabBar` prop) |
| 3 — `router.push` would push instead of switch | 4.1 (use `navigation.navigate`) |
| 4 — DashboardSidebar accumulates stack | 4.3 (use `router.replace`) |
| 5 — `router.back()` works correctly | No change needed — it's the desired behavior |
| 6 — TopBar rendering pattern | 3.2 + 3.5 (`headerShown: false`, TopBar above Stack) |
| 7 — Portal layout flat structure | 3.2 (becomes `<Tabs>` with chrome above) |
| 8 — Route groups don't map to tabs | 3.2 + 3.3 (nested `<Stack>` per tab; Home tab wraps `(public)/`) |
| 9 — `+not-found.tsx` orphaned `<Stack.Screen>` | 3.7 (remove options, use custom TopBar) |
| 10 — Owner layout TopBar + Sidebar around Slot | 3.5 (Stack between, chrome remains sibling) |
| 11 — KAV offset assumes layout heights | Engineer judgment at implementation; measure empirically |
| 12 — Messages state-based toggle | Out of scope (Fork D — deferred to chat B) |
| 13 — UserMenu broken paths | Out of scope (deferred to Ω) |
| 14 — DashboardSidebar non-reactive dimensions | 4.3 + 4.4 (covered by F26) |

## 6. Trust boundaries

Φ2 does not change trust boundaries. The auth guards from Φ1 (server-derived `user` from `useAuthStore` after Firebase sync) remain the only mechanism gating secured surfaces. No new client-supplied values reach moderation, authorization, or state-transition decisions.

The deep-link inventory (audit Section 7) confirmed no custom deep-link handling code exists. Expo-router auto-handles deep links through layout trees; Φ1's auth guards intercept correctly before secured screens mount.

## 7. Definition of done

- All six layouts use native navigators per section 3 (no remaining `<Slot />` in any layout file).
- BottomBar is wired as `<Tabs>`'s `tabBar` prop and navigates via `navigation.navigate()`.
- DashboardSidebar navigates via `router.replace()`.
- All four `Dimensions.get` sites use `useWindowDimensions()`.
- `+not-found.tsx` renders the custom TopBar pattern (no native header).
- All `<Stack>` and `<Tabs>` navigators have `screenOptions={{ headerShown: false }}` (or equivalent per-screen).
- `npx tsc --noEmit` exit 0, zero errors.
- `npm run lint` zero errors. Pre-existing warnings unchanged (82 baseline).
- `npm test` ≥ 109 passed. New tests may be added if a refactor needs coverage, but Φ2 is structural — most existing tests should pass unchanged.
- `npx expo-doctor` no new failures vs Φ1 baseline.
- Manual smoke on real device pending per section 10 (status flips to `shipped` after smoke clears).

## 8. Definition of "verifying" (the smoke phase)

After code lands and CI is green, Igor runs the following on a real device (Pixel 4a or equivalent mid-range Android plus iPhone):

- Tab switches preserve scroll position and screen state per tab (open product detail in Home tab → switch to Favorites → return to Home → product detail still visible).
- iOS swipe-back gesture works on every Stack-pushable screen.
- Android hardware back button pops the current Stack screen correctly.
- Native slide transition visible between screens.
- BottomBar active-tab indicator updates correctly on tab switches.
- Auth guards still redirect unauthenticated users on `(secured)/`, `owner/`, `owner/dashboard/` deep links (test with the app cold-started via `oglasino://favorites` or equivalent).
- DashboardSidebar navigation doesn't accumulate back stack (Products → Balance → User → press back → exits dashboard, doesn't retrace).
- Foreground re-validation, ban dialog, restoration dialog (Φ1 surfaces) all continue to work.
- Push notification taps route to the correct screen (mobile push surface from Φ1).
- `+not-found.tsx` renders with custom TopBar visible.

Bugs surfaced during smoke get fixed in patch sessions before Φ2 closes. Once smoke clears, `state.md` flips Φ2 to `shipped` and chat Φ3 opens.

## 9. Engineer brief sequencing

Six briefs expected. One per session per conventions Part 5. Order:

**Brief 1 — Root and portal navigators (F12 partial + Seams 1–3, 6, 7).** Root layout becomes `<Stack>`. Portal layout becomes `<Tabs>` with BottomBar as `tabBar` prop. BottomBar refactored to accept `BottomTabBarProps`, read active tab from `state`, navigate via `navigation.navigate()`. Home-tab nested `<Stack>` in the `(public)/` group.

**Brief 2 — Owner navigators (F12 remainder + Seam 10).** Owner layout (`app/owner/_layout.tsx`) becomes `<Stack screenOptions={{ headerShown: false }}>`. Dashboard layout (`app/owner/dashboard/_layout.tsx`) becomes `<Stack screenOptions={{ headerShown: false }}>`. Auth guards in both layouts remain unchanged (they don't conflict with `<Tabs>` because `owner/` is a top-level Stack screen, not a tab child).

The secured route group is intentionally skipped — Brief 1 established the inline-guard pattern there per Section 3.4. No work in `app/(portal)/(secured)/` for this brief.

**Brief 3 — DashboardSidebar refactor (Seam 4).** Navigation calls switch to `router.replace()`. Verify the dashboard surface composes correctly with the new owner `<Stack>`.

**Brief 4 — F26 (Dimensions.get → useWindowDimensions).** Four sites: FloatingButton, FullScreenImageViewer, ProductList, DashboardSidebar. Mechanical, low risk.

**Brief 5 — `+not-found.tsx` cleanup (Seam 9).** Remove orphaned `<Stack.Screen>` options. Ensure TopBar renders or document why it doesn't.

**Brief 6 — Stabilization and verification (KAV offset adjustment if needed per Seam 11, any patch-up).** Runs after manual smoke surfaces issues. May be skipped if smoke is clean.

The engineer may consolidate Brief 4 into Brief 1 or 2 if the navigator work touches the same files. The engineer may split Brief 1 further if scope grows during implementation. Mastermind drafts the briefs one at a time, sees the result, then drafts the next.

## 10. Status

Φ2 status flips to `shipped` only after:

- All section 7 definition-of-done items confirmed in the final engineer session summary.
- Manual smoke per section 8 clears on real device.

Until both clear, Φ2 status is `verifying` (per the Φ1 precedent in `state.md`).

## 11. Factual vs inferred

**Factual:**
- 14 seams, their current code state, their resolutions (audit + Fork answers).
- F13 working with `<Redirect>` under nested navigators (verified from expo-router source in audit).
- F26 inventory (4 sites confirmed in audit).
- Provider stack at root layout (audit Section 8).
- Forks A/B/C/D/E resolutions (Igor confirmed 2026-05-25).
- Expo-router constraint (Section 3.4): layout files collapse route groups into single navigator children. Verified from expo-router source `flattenDirectoryTreeToRoutes` (`getRoutesCore.js`) during Φ2 Brief 1 (2026-05-25).

**Inferred:**
- Brief sequencing in section 9 — engineer judgment may amend.
- Engineer chooses the exact directory restructuring shape under nested-navigator constraint (section 3.8).
- Section 8 smoke checklist composition — based on audit findings; Igor's smoke may surface additional checks.
- Engineer-judgment items: KAV offset adjustment (Seam 11), whether to consolidate F26 into earlier briefs.
- The original §3.1 assumed `<Stack>` could replace `<Slot />` inside the existing conditional-rendering pattern (Stack alongside loading / maintenance / select-base-site branches). This was inferred from the working `<Slot />`-based pattern, not verified against expo-router's navigator-lifecycle behavior. During Φ2 manual smoke (2026-05-27) the assumption proved wrong: expo-router reinitializes the navigation container when a navigator mounts, which remounted the root layout in an infinite loop. §3.1 was rewritten to the always-mounted pattern; §3.2 was extended with the chrome guard required by the new pattern.

## 12. References

- Structural audit: `sessions/audit-expo-structural.md` (2026-05-24)
- Φ2 navigation audit: `sessions/audit-phi2-navigation.md` (2026-05-25)
- Φ1 spec: `features/expo-auth-lifecycle.md`
- Program spec: `features/expo-structural-foundation.md`
- Expo-router source for `<Redirect>` behavior: `node_modules/expo-router/build/link/Redirect.js`