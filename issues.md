# Issues

Append-only log of out-of-scope findings. Newest at the top. Each entry has a date, severity, status, and a short body. Status values: `open`, `fixed`, `wontfix`, `parked`.

---

## 2026-05-19 — Registration `displayName` never persists to backend

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/lib/store/useAuthStore.ts` (register action); `oglasino-web/src/lib/service/reactCalls/authService.ts` (registerUserFirebase); `oglasino-backend` firebase-sync handler.
**Detail:** Registration form collects `displayName`, but it never reaches Firebase Auth, Firestore, or the backend's `users` row. The pre-existing flow created a Firebase user via `createUserWithEmailAndPassword(email, password)` (no displayName), POSTed `{ allowPreferenceCookies }` to firebase-sync (no displayName), then patched the in-memory DTO with the form value before storing. The patching gave the UI a few-second illusion of "looks right" before the next sync overwrote it.

Frontend Observation-1 patch (2026-05-19) removed the patching as part of the listener-is-sole-hydrator design. The underlying gap is now visible: post-register, the user sees whatever the backend defaulted to.

**Proposed fix (chosen design):** extend the `firebase-sync` POST body to carry `displayName`. Backend `firebase-sync` handler reads it from the body and persists to the `users` row's `displayName` column. The backend owns the canonical store of `displayName`; Firebase Auth carries identity only.

Architecture: backend as source of truth aligns with how `displayName` changes are handled later in the user's lifecycle (the edit-profile flow writes only to the backend). Symmetric pattern across register + update means one place to maintain.

Out of scope for this feature (registration is its own surface). Schedule as a separate brief post-launch or whenever a registration-pass is scheduled.

---

## 2026-05-19 — Admin Users detail page indicators stale after in-page ban/unban toggle

**Severity:** low (cosmetic)
**Status:** open
**Found in:** `oglasino-web/app/[locale]/admin/users/[userId]/page.tsx`; `oglasino-web/src/components/admin/users/EnableDisableButton.tsx`.
**Detail:** The detail page renders state indicators from initial-fetch `userData.disabled`. When an admin toggles ban/unban via the `EnableDisableButton` on the same page, the button's local `disabled` state flips and its icon swaps, but the page's `userData.disabled` is not lifted to a shared source. The new "Banned" badge does not appear until the page is refetched or re-navigated.

**Fix path:** mirror the list-page `UserRow` pattern — lift `disabled` (and `lockedFromDeletion` if a lock toggle is added here later) to page state, add an optional `onDisabledChange` callback to `EnableDisableButton` matching `EnableDisableIcon`'s controlled-mode contract.

Out of scope for the indicator-only brief that surfaced it.

---

## 2026-05-19 — Backend → web cache revalidation may not work in production via oglasino.com domain

**Severity:** medium (verification pending)
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultWebRevalidationService.java`; Cloudflare router worker (`oglasino-router`); backend prod/stage env var `WEB_REVALIDATE_URL`.
**Detail:** Group 2/3's backend → web cache-revalidation call POSTs to a URL provided via the `WEB_REVALIDATE_URL` env var. If that URL points at the `oglasino.com` domain (e.g., `https://oglasino.com/api/revalidate`), the Cloudflare router worker routes `/api/*` paths to the backend — meaning the backend POSTs to itself, hits no `/api/revalidate` endpoint, returns 404, and the web-side cache is never invalidated.

**Mitigation chosen:** Igor will point `WEB_REVALIDATE_URL` directly at the Vercel deployment URL (e.g., `https://oglasino-web.vercel.app/api/revalidate` or whatever the prod Vercel hostname is) to bypass the Cloudflare worker entirely.

**Verification needed:** confirm direct-Vercel-URL works in production. Two failure modes possible: (a) Vercel deployment URL changes per environment (preview vs prod) and breaks if not maintained; (b) backend's HTTP client treats the direct hostname differently (TLS, redirect handling, etc.) than the worker-fronted hostname.

**Alternative architectural fixes** (if direct-URL proves fragile):
- Add `/api/revalidate` as a Cloudflare worker exception that routes to Vercel
- Rename the frontend endpoint to a non-`/api/*` path (`/_revalidate` or `/internal/revalidate`)
- Use a separate internal domain

Verification first. If direct-URL works, close as "verified working" post-launch. If not, follow-up brief ships one of the alternatives.

---

## 2026-05-19 — Home page only loads first page; possibly affects other paginated pages

**Severity:** medium (not blocking User Deletion launch)
**Status:** open
**Found in:** home page; possibly other paginated views (not yet audited).
**Detail:** Surfaced during User Deletion manual smoke testing. Out of scope for User Deletion feature — log here for follow-up. Pagination loads only the first page; subsequent pages may not be fetchable.

Investigation needed. Schedule as its own brief.

---

## 2026-05-19 — `/api/revalidate` used wrong cache-invalidation profile (silent no-op)

**Severity:** medium (resolved)
**Status:** fixed (2026-05-19)
**Found in:** `oglasino-web/app/api/revalidate/route.ts:82` (now corrected).
**Detail:** Route handler called `revalidateTag(tag, 'default')`, which under Next.js 16 is the long-cache profile (renews tag lifetime under the default profile), not immediate invalidation. Backend revalidation calls succeeded at HTTP level but did not actually invalidate the SSR cache. Users saw stale `UserInfoDTO` for up to 5 minutes after state transitions.

**Resolved by:** changing the call to `revalidateTag(tag, { expire: 0 })` — `CacheLifeConfig` with zero expiry forces immediate invalidation. Manual smoke confirmed: ban → unban → normal refresh now shows correct state (no hard refresh needed).

Round-3 handoff flagged this as a project-wide pattern. The `/api/revalidate` site is now fixed; other call sites (`oglasino-web/app/actions/cacheActions.ts` and possibly others) still need audit/fix. Separate task.

---

## 2026-05-19 — Manual testing of messaging gating for deleted and banned users not yet performed

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/components/Messages.tsx`; `oglasino-backend` ban + deletion lifecycle.
**Detail:** The User Deletion feature's messaging-gating logic is shipped in code on both repos but has not been end-to-end manually verified. Two surfaces are affected:

- **Pending-deletion gating** (round 3): per spec §3.2 / §14.8, when a user is in `PENDING_DELETION`, counterparties should see the "Scheduled for deletion" badge in chat header, the message input should be disabled, and the notice "This user has scheduled their account for deletion and cannot receive messages." should render. The deleting user themselves should be signed out and unable to send/receive.
- **Banned-user gating** (B2, 2026-05-19): per the same spec section extended for ban handling, when a user is banned, counterparties should see the "Banned" badge in chat header, the message input should be disabled, and the notice "This user has been banned and cannot receive messages." should render. Banned users cannot sign in at all so the counterparty-side gating is the only visible surface.

The smoke test documented in the frontend B2 session summary (`.agent/2026-05-19-oglasino-web-user-deletion-<n>.md`, see "Tests" section) cannot run today because messaging is broken in current local stack. A separate fix is queued. Once messaging is working, the full smoke sequence must run:

1. User A messages User B; admin bans User B; counterparty (User A) sees badge / disabled input / notice in chat with User B.
2. Admin unbans User B; counterparty's chat returns to normal.
3. Third user (User C) put in pending-deletion via danger zone; counterparty sees pending-deletion badge / disabled input / notice.
4. User C cancels deletion by signing back in; counterparty's chat returns to normal.
5. Edge case: pending-deletion user is then admin-banned; counterparty sees "Banned" badge (not "Scheduled for deletion") — banned wins over pending-deletion in the UI per backend's `UserState.resolve()` composition.

Until this manual verification completes, the User Deletion feature's messaging-gating surface is considered not-fully-tested. Code review and unit tests on both repos passed (backend 487, web 154 green); behavior under live local stack remains unverified.

Fix scope: zero code work pending. This entry tracks the verification gap; close it after the manual smoke runs successfully.

---

## 2026-05-17 — `UserInfoDTO.isFollowingCurrent` seed is non-authoritative because `getUserForId` uses `skipAuth: true`

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/lib/service/nextCalls/userService.ts:13-19` (the `getUserForId` call), `oglasino-web/src/lib/config/fetchApi.ts:15-19, 38-45` (the `skipAuth` semantics), and the consumer at `oglasino-web/src/components/client/UserDetails.tsx:119` (`isFollowing={userDetails.isFollowingCurrent}` passed to `FollowUserButton`).
**Detail:** Server-side `getUserForId` fetches `/public/user?id=<id>` with `skipAuth: true`. Per the `fetchApi.ts` documentation comment, `skipAuth: true` explicitly bypasses the `cookies()` call AND the Firebase Bearer header — so the backend cannot identify the requesting viewer on this code path. Yet the returned `UserInfoDTO` carries two viewer-dependent boolean fields, `iamActive` and `isFollowingCurrent`, both of which require knowing who is asking to populate correctly. `iamActive`'s consequence is mitigated in `UserDetails.tsx:53-59` by an internal client-side recomputation against `useAuthStore.user.id`; `isFollowingCurrent` has no such mitigation — it is passed straight into `<FollowUserButton isFollowing={userDetails.isFollowingCurrent} />` (`UserDetails.tsx:119`), which uses it as the seed for its local `following` state. Observable consequence: a signed-in viewer who already follows User X opens `/user/<X-id>` (or any `/product/<…>` owned by X) and sees the Follow icon (UserPlus) instead of the Unfollow icon (UserMinus). One click then *unfollows* them, contrary to the user's mental model. Compounding: there is no `revalidateTag('user:${userId}')` call anywhere downstream of `markFollowUser`, so the cached `UserInfoDTO` (Next.js Data Cache, 5-minute `revalidate` with tag `user:${userId}` per `userService.ts:17-18`) does not refresh after a follow action either. Fix scope: either split `getUserForId` into a public-skipAuth path that omits the viewer-dependent fields and an authenticated path that does forward auth (matching the `/public/*` vs `/secure/*` split the backend already has), or drop `isFollowingCurrent` from this public endpoint and seed the button via a separate small auth-required call, or compute the field client-side post-mount via a Zustand follows-store. Project-wide implications similar to `useAuthResolved`-style hydration handling — likely its own focused chat. Found by Docs/QA during the QA-Preparation `follow-flow` topic authoring.

---

## 2026-05-17 — `/owner/follows` `UserCard` calls `notify.error()` for both success and failure paths

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/owner/follows/UserCard.tsx:19-33`.
**Detail:** The `onUnfollow` handler in `UserCard` has two branches based on the `markFollowUser` result. The `result === undefined` branch (line 21-25) calls `notify.error()` and is dead code — `markFollowUser` always returns a `boolean` (`followService.ts:8,12,15`), never `undefined`. The other branch (line 26-31), explicitly meant for the success case (`tCommon(result ? 'user.follow.added' : 'user.follow.removed')`), also calls `notify.error()` instead of `notify.success()`. Result: every unfollow action from the owner dashboard toasts in the error/red style even when the operation committed successfully. Trivial wording fix — switch the success branch to `notify.success()` and drop the dead `undefined` branch (or repurpose it to call `notify.error` for a real error path returned by a future revision of `markFollowUser`). Found by Docs/QA during the QA-Preparation `follow-flow` topic authoring.

---

## 2026-05-17 — `/owner/follows` `UserCard` does not remove the unfollowed row from the list

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/owner/follows/UserCard.tsx:19-33` (the handler) and `oglasino-web/app/[locale]/owner/follows/page.tsx:75-91` (the consumer that owns the list state).
**Detail:** `UserCard.onUnfollow` calls `markFollowUser` and fires a toast but does not update the parent `followsUsers` state. The card stays visible in the list with its "Unfollow" button still active and labelled — clicking it again *re-follows* the user (because the underlying endpoint is a toggle: first call adds, second call removes, etc.). The list only re-aligns with reality after a manual page reload or a `loadMore` call against the backend. Owner sees a stale list with confusing toggle semantics. Fix scope: pass a parent callback into `UserCard` (or call a parent-supplied filter) that removes the row from `followsUsers.users` on a successful unfollow response. Related to the dead-row-state class of bug; same root pattern as the `/owner/follows` UI not auto-refreshing. Found by Docs/QA during the QA-Preparation `follow-flow` topic authoring.

---

## 2026-05-17 — Start-Message product context (`tempProductReason`) cleared on MessageInput mount, drops `productId` from the first message

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/components/MessageInput.tsx:57-68` (consumer) and `oglasino-web/src/messages/store/useChatStore.ts:499-501, 516-518, 575-578` (producer + intended cleanup site).
**Detail:** When a user starts a conversation from a product page via `StartMessageButton`, the store action `setTempProductReason(product)` records the originating product so that the first `MessageRequest` written to Firestore can carry its `productId`. `MessageInput`'s mount `useEffect` reads `tempProductReason` to compose the `initial.product.message` suggestion text, then immediately calls `setTempProductReason(undefined)` — clearing the store field *before* the user types and sends. By the time `useChatStore.sendMessage` runs and checks `if (state.tempProductReason) { messageRequest.productId = state.tempProductReason.id; }`, the field is already `undefined`, so `productId` is silently dropped from the first message of every product-path conversation. The store's own end-of-send cleanup (`set({ tempReceiver: null, tempProductReason: null })` at `useChatStore.ts:575-578`) further confirms the intent was for the clear to happen on send, not on mount. Visible symptom: every conversation initiated via a product page lacks the originating-product link in its first message — any consumer of `MessageRequest.productId` (audit, "this chat started about Product X" surfacing, analytics) sees no data. The text prefill itself still works because `MessageInput`'s `suggestion` is stored in local state. Fix scope: move the `setTempProductReason(undefined)` call out of `MessageInput`'s mount effect; rely on the store's existing post-send cleanup in `sendMessage` instead. Found by Docs/QA during the QA-Preparation `start-message-flow` topic authoring.

---

## 2026-05-17 — Hardcoded Serbian "Kategorije iz <X>" breadcrumb button label

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/server/OglasinoBreadcrumbs.tsx:81`
**Detail:** The "jump into subcategories" button rendered at the tail of a breadcrumb chain (when the chain has ≤ 2 levels) uses the literal Serbian string `Kategorije iz {t(category.labelKey)}` instead of a next-intl key — EN / RU / ME visitors see Serbian. A Part 6 (translations) violation in `oglasino-web` source, same family as the existing 2026-05-14 *Hardcoded Serbian tooltips on the product detail page* entry. Found by Docs/QA during the QA-Preparation `category-navigation` topic authoring.

---

## 2026-05-16 — SEO JSON-LD delivered as `<meta>` tags instead of `<script>`, project-wide

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web` — nine `src/metadata/generate*Metadata.ts` files: Home, About, Pricing, FreeZone, Catalog, Product, User, Privacy, Terms. Plus the inline `<script>` at `app/[locale]/(portal)/(public)/privacy/page.tsx:21-26`.
**Detail:** Every public page emits its Schema.org document into `metadata.other['application/ld+json']`. Next.js renders `metadata.other` entries as `<meta name="application/ld+json" content="...">` tags; search engines parse JSON-LD only from `<script type="application/ld+json">` blocks. Result: every public page in the codebase has a structured-data block that no crawler will read. Real SEO impact pre-launch. Fix scope: export the structured-data helpers from each metadata file and emit inline `<script>` tags on each page. Nine-page touch plus helper-export plumbing. Privacy additionally has a malformed inline `<script>` that stringifies `generatePrivacyPageMetadata`'s return value (a Next.js `Metadata` object — `{ title, description, alternates, openGraph, twitter, other }`) instead of the dedicated `generatePrivacyPageStructuredData` helper output; a clean fix on Privacy is part of this work. Feature-sized. Subsumes the 2026-05-14 entry "Terms page does not emit JSON-LD; Privacy page does" — see that entry's amended body. Surfaced by the terms-json-ld-1 session.

---

## 2026-05-16 — Report-submit endpoint trust-boundary verification unknown

**Severity:** medium-pending-audit
**Status:** open
**Found in:** `oglasino-web/src/lib/service/reactCalls/reportService.ts:12` (client submit call site); backend home is `oglasino-backend` (controller + service for `POST /secure/report/add`).
**Detail:** `reportedUserId` is client-supplied to `POST /secure/report/add` from both `ReportButton.tsx` (public user page) and the new `Messages.tsx` dropdown wiring. Whether the backend verifies (a) the id resolves to a real user, and (b) the authenticated reporter is authorized to report this target (participant in a conversation, viewer of a public user page, etc.) cannot be determined from the web repo alone. If the backend doesn't validate, any logged-in user can file false reports against any other user by manipulating the request payload — an abuse vector regardless of how the UI surfaces the action. Pre-existing if it's a gap (the `ReportButton` call site has the same shape since before this chat's work). Needs a one-shot backend audit of the `report/add` controller and service to confirm; severity finalizes after the audit. Surfaced by the messages-report-wiring-1 session.

---

## 2026-05-16 — `useAuthResolved` adoption pending across the app

**Severity:** low
**Status:** open
**Found in:** `oglasino-web` — `src/lib/hooks/useAuthResolved.ts` (new), `src/components/client/HeaderNavButtons.tsx`, `src/components/client/SessionGuard.tsx`, `src/lib/store/useAuthStore.ts`.
**Detail:** The `useAuthResolved` hook introduced by the hydration-flashes-1 session is the right shared primitive, but adoption is partial. `HeaderNavButtons` still inlines the same Firebase-ready + store-user-loaded gate that the hook now formalizes (the precedent the hook mirrored). `SessionGuard` uses `auth.authStateReady()` for whole-route gating, a related-but-different technique. `useAuthStore` exposes no synchronous "initialized" flag, which is why every consumer that cares about the flash window has to bolt a Firebase listener onto the side. Scope: audit every auth-gated component across the app, consolidate around `useAuthResolved` where in-component gating is needed and `SessionGuard`-style for route-level, and consider adding an `initialized: boolean` flag to `useAuthStore` so the side-bolt becomes unnecessary. Likely its own focused session, possibly a Mastermind feature chat. Surfaced by the hydration-flashes-1 session.

---

## 2026-05-16 — `isAllowedPath()` allowlist anti-pattern

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx:52-66`.
**Detail:** `isAllowedPath()` is an opt-in-by-string-matching list of paths embedded inside `FilterManager`. It grows one path at a time and forgets dynamic segments (the `[userId]` exclusion was its second documented failure). The mechanism has produced two real bugs so far: the original `/admin/products/[userId]` chip-no-op, and the in-app-nav broken state on `/admin/products` and `/owner/products` (the second was latent until the admin-filter-pipeline audit surfaced it). A different model — a per-route `enableFilterSync` opt-in prop, or moving `FilterManager` from layouts to page-level mounts — would remove the footgun. Feature-sized refactor; not bug-fix scope. Surfaced by the admin-filter-pipeline audit and fix sessions.

---

## 2026-05-16 — Sitemap generation fetches full product payload to read a count

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/sitemap.ts:79-83` — `getProductCountForBaseSite`.
**Detail:** `getProductCountForBaseSite` calls `getProductsPage(baseSite, 0, 1)` purely to read `totalNumberOfProducts`, paying for one full product record's payload (and any backend hydration cost on a 1-row page) when only a count is needed. A count-only backend endpoint (`GET /api/public/product/count?baseSite=...` or equivalent) would be cheaper at sitemap-generation time. Cross-repo — needs a backend addition before the sitemap can switch over. Surfaced by the empty-overviews-consolidation-2 session.

---

## 2026-05-16 — `/admin/products` filter chips don't take effect after in-app nav from disallowed admin route

**Severity:** medium
**Status:** fixed (2026-05-16, session oglasino-web-admin-filter-pipeline-fix-1)
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx`.
**Detail:** Surfaced and fixed in the same chat. `FilterManager`'s HYDRATE FROM URL effect had deps `[baseSite]` and ignored `pathname` changes; in-app nav from a disallowed admin path (e.g., `/admin/users` → click "Products") into `/admin/products` left `hydratedRef.current === false` because the effect's deps never re-fired, and the SYNC TO URL effect silently early-returned on every subsequent chip click. URL never updated, list never refetched. Hard reload was the only way to escape the gated state. Fixed by replacing `hydratedRef` boolean with a per-path `lastHydratedPathRef`, adding `pathname` to the HYDRATE effect deps, and updating the SYNC effect gate accordingly. Same root cause behind the existing 2026-05-14 entry on `/admin/products/[userId]` chip changes; both entries close on the same fix. The same deps-array bug had a latent twin on `/owner/products` (in-app nav from disallowed owner sidebar routes like `/owner/balance`, `/owner/reviews`, `/owner/follows`, `/owner/analytics`) — incidentally fixed by the same change.

---

## 2026-05-16 — Global header search input has no Enter-key submission handler

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/components/client/SearchInput.tsx:134-141`
**Detail:** The `<Input>` rendering the global header search has no `onKeyDown` handler and is not wrapped in a `<form>`, so pressing Enter while focused in the input does nothing — no submit, no popup open, no navigation. The only path that persists a typed term into the destination listing page's `searchText` chip and URL is clicking the submit button at the bottom of the autocomplete dropdown (same file, lines 164-194), and that button only renders when the autocomplete returned at least one product. Users almost universally try Enter first; the dead key reads as "search is broken" in a way that's hard to diagnose without reading the component. Affects all three scopes (portal, owner dashboard, admin) — the same `SearchInput` is reused via `SelectableSearchInputWrapper`. Cross-referenced from the `global-header-search` QA topic as a "Known issue" pitfall; the topic's `qaChecklist` includes a standing check that pins current behaviour and will need flipping when this is fixed. Found by Docs/QA during the QA-Preparation `global-header-search` topic authoring (2026-05-16).

---

## 2026-05-16 — Five loose `^N` major-only version constraints remain in `package.json`

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/package.json` — `@tailwindcss/postcss`, `@types/react`, `@types/react-dom`, `tailwindcss`, `typescript`
**Detail:** Five direct dependencies are declared with major-only `^N` constraints (e.g. `"typescript": "^5"`, `"tailwindcss": "^4"`). Under these constraints, the `package.json` floor doesn't enforce a meaningful minimum — `npm install` drifts forward freely as soon as a new minor lands within the major. The actual installed version sits in `package-lock.json` only. Down from six since the 2026-05-16 dependency-upgrade brief tightened `@types/node ^20` to `^22`. A future small cleanup pass should normalize these to `^MAJOR.MINOR.PATCH` floors reflecting what's actually installed. Found by Docs/QA in the web dependency audit (2026-05-16) and confirmed in the upgrade session summary.

---

## 2026-05-16 — 211 pre-existing ESLint warnings in `oglasino-web`

**Severity:** low
**Status:** open
**Found in:** `oglasino-web` — repo-wide
**Detail:** `npm run lint` reports `✖ 211 problems (0 errors, 211 warnings)` on `dev`. All 211 are pre-existing — none introduced by the 2026-05-16 dependency-upgrade work that surfaced them. Categories: `react-hooks/exhaustive-deps`, `@typescript-eslint/no-explicit-any`, `@next/next/no-img-element`. Pre-launch this is tolerable; the count is the kind of slow drift that becomes invisible. Worth one focused lint-cleanup brief at some point, but not blocking anything. Found by the web dependency-upgrade engineer (2026-05-16).

---

## 2026-05-16 — Three `-Xlint` compile warnings in `oglasino-backend`, pre-existence unverified

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionController.java`, `src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/ProductIndexer.java`, `src/test/java/com/memento/tech/oglasino/exception/GlobalExceptionHandlerTest.java`
**Detail:** `./mvnw clean compile` and `./mvnw test` emit three `-Xlint` notices: deprecated-API use in `AppVersionController` and `GlobalExceptionHandlerTest`, unchecked/unsafe operations in `ProductIndexer`. Surfaced during the 2026-05-16 dependency-upgrade session (`firebase-admin 9.8.0 → 9.9.0`, AWS SDK 2.42 → 2.44, `postgresql` 42.7.10 → 42.7.11). The session did not capture a pre-upgrade baseline, so it cannot prove these are pre-existing — but none of the three files reference Firebase Admin or AWS SDK classes by name, suggesting they are pre-existing codebase notices unrelated to the upgrade. **Recommended quick check:** a `-Xlint:deprecation,unchecked` compile on a fresh checkout of the pre-upgrade `pom.xml` would confirm pre-existence cheaply. If pre-existing, normal cleanup; if newly-surfaced, one of the upgraded SDKs is implicated.

---

## 2026-05-16 — Verify whether `opentelemetry-semconv` pin remains necessary post-Firebase-upgrade

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/pom.xml` — `<dependencyManagement>` `io.opentelemetry.semconv:opentelemetry-semconv 1.41.1`
**Detail:** The pin was added on 2026-05-15 because `firebase-admin → google-cloud-storage` was pulling in `opentelemetry-semconv 1.29.0-alpha`, which lacked `DbAttributes` and caused Elasticsearch client startup failure (see 2026-05-15 entry, status `fixed`). On 2026-05-16, `firebase-admin` was upgraded 9.8.0 → 9.9.0 (transitively pulling `google-cloud-storage` 2.63.0 → 2.64.0). `dependency:tree` on both before and after resolves `opentelemetry-semconv` to `1.41.1` — but that's the effective post-mediation version, dominated by the pin. The real question — whether Maven would still mediate to `1.29.0-alpha` without the pin — needs `./mvnw dependency:tree -Dverbose=true` on the post-upgrade pom to surface omitted candidates. **If the verbose output no longer shows `1.29.0-alpha` in the candidate set, the pin is redundant and can be removed.** If `1.29.0-alpha` is still in the set, the pin remains load-bearing. Cheap to run in one read-only session.

---

## 2026-05-15 — `oglasino-web` tsconfig has `strict: false`

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/tsconfig.json`
**Detail:** Strict null-checks are not enforced. The filtersHelper-return-type-1 session surfaced the consequence: declared-optional fields can be de-`?.`-chained without tsc rejection, even when the field's type declaration says they may be undefined. This means the typechecker is silent on a class of real bugs — anywhere a `Type | undefined` value is used without a null guard, tsc will not complain. Moving to `"strict": true` (or at minimum `"strictNullChecks": true`) would surface every such site as a tsc error; some will be true bugs, most will be type-tightening work where the producer always emits but the declaration says otherwise. Scope is project-wide, not a single-file fix. Surfaced by the filtersHelper-return-type-1 session addendum.

---

## 2026-05-15 — `printStackTrace()` instead of logger in GlobalIndexerService

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/GlobalIndexerService.java:29`
**Detail:** Per-indexer failures during boot reindex are printed via `ex.printStackTrace()` instead of using the project's logger. Conflicts with conventions Part 4 ("No `System.out.println`... Logger calls fitting the existing logging strategy are fine"). Diagnostic-output style only; no user-visible impact.

---

## 2026-05-15 — Main method uses Java 21 preview signature, blocks `mvn spring-boot:run`

**Severity:** high
**Status:** fixed
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/OglasinoApplication.java:11` and `pom.xml` `spring-boot-maven-plugin` configuration
**Detail:** `OglasinoApplication.main` is declared `static void main(String[] args)` — package-private. This signature is only valid as a JVM entry point under Java 21 preview feature JEP 463 (Instance Main Methods). The Maven compile and surefire plugins pass `--enable-preview`, but `spring-boot-maven-plugin` does not, so `mvn spring-boot:run` fails immediately with `Error: Main method not found in class com.memento.tech.oglasino.OglasinoApplication`. IDE runs likely add the preview flag automatically, which is why this has not been noticed. Effect: every brief that mandates a `mvn spring-boot:run` verification step is currently un-verifiable from the command line. Fixed by changing `main` to `public static void main`. Verified end-to-end via Docker dev environment.

---

## 2026-05-15 — Backend fails to start: missing OpenTelemetry semconv class

**Severity:** high
**Status:** fixed
**Found in:** `oglasino-backend` — startup fails during `GlobalIndexerService.onAppReady()` → `ProductIndexer.reindexAll()` → Elasticsearch client index creation
**Detail:** Spring Boot startup fails with `java.lang.NoClassDefFoundError: io/opentelemetry/semconv/DbAttributes`. The Elasticsearch Java client (9.2.6) has built-in OpenTelemetry instrumentation that requires the `opentelemetry-semconv` library. Either it is not on the classpath or a transitive version is too old to contain `DbAttributes` (the class lives in `opentelemetry-semconv` 1.27+). Suspected cause: a recent dependency upgrade (Spring Boot 4.0.5, Spring 7.0.6, Elasticsearch Java 9.2.6) bumped the Elasticsearch client to a version that needs a newer semconv than what's resolved. App does not start locally. Fixed by pinning `opentelemetry-semconv` to 1.41.1 in pom.xml dependencyManagement. See commit 5df54c9cde9507109f0e78bf662b27c994a16be2.

---

## 2026-05-15 — `use.backend.check` read sits outside the worker's fail-open try/catch

**Severity:** medium
**Status:** open
**Found in:** `oglasino-router/src/index.ts:91-93`
**Detail:** The fail-open `try`/`catch` in the router worker wraps the reads for `maintenance.active` and `admin.bypass.disabled`, but the `CONFIG.get("use.backend.check")` call sits outside it. If that specific KV read throws (transient KV outage), the exception propagates and the request fails — the opposite of the worker's documented fail-open behavior. Trivial fix: wrap or move the read inside the existing `try`. Found by Mastermind during the maintenance-split-propagation audit (2026-05-15). Linked from [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md) Known limitations.

---

## 2026-05-15 — Worker matrix comment doesn't mention `use.backend.check`

**Severity:** low
**Status:** open
**Found in:** `oglasino-router/src/index.ts:1-13`
**Detail:** The matrix-comment block at the top of the worker is the closest thing to an in-source spec for the gate logic, but it documents only the two split flags (`maintenance.active` and `admin.bypass.disabled`). `use.backend.check` is a third input to `maintenanceActive` that can independently force the gate on; a reader treating the header comment as the spec misses a real input. Documentation-only fix — extend the comment to cover all three inputs. Found by Mastermind during the maintenance-split-propagation audit (2026-05-15). Linked from [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md) Known limitations.

---

## 2026-05-15 — Favorites page has brittle positional type-narrowing on initialProductsData

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-favorites-narrowing-1)
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/favorites/page.tsx:52,64`
**Detail:** The page accesses `initialProductsData.products?.map(...)` where `initialProductsData` is typed `ProductOverviewsDTO | undefined`. The preceding `if (loading) return <LoadingOverlay />` guard prevents an actual undefined deref at runtime (`getFavorites` always resolves via `EMPTY_OVERVIEWS` on error, and `.then` runs before `.finally`), but the narrowing is purely positional and brittle to any refactor that moves the access or the guard. Type-tightening cleanup, no user-visible bug. Found by Docs/QA during the QA-Preparation Notifications + Favorites batch.

**Fix:** `useState` retyped from `ProductOverviewsDTO | undefined` to `ProductOverviewsDTO` with explicit empty initial value; positional `?.` chains on `.products` at both `excludeIds` sites removed; `if (loading)` UX guard retained.

---

## 2026-05-15 — Favorites "recently viewed" carousel is misnamed

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/favorites/page.tsx:50`
**Detail:** The bottom carousel uses the `recently.viewed.title` translated heading, but its filter is `{ excludeIds: <favorited-ids> }` — no "recently viewed" criterion is sent. The heading promises recently-viewed products; the filter delivers something else. Same family of label/filter mismatch as the `user-page` "Similar products" carousel (documented there as a pitfall of the unfinished extra-products feature). If the extra-products feature is still WIP, this may be a known rough edge rather than a defect to fix — triage accordingly. Found by Docs/QA during the QA-Preparation Notifications + Favorites batch.

---

## 2026-05-14 — Owner-view hydration flash on UserDetails action buttons

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-hydration-flashes-1)
**Found in:** `oglasino-web/src/components/client/UserDetails.tsx:51-57`
**Detail:** `iamActive` is computed via `useMemo` over the auth store. On first paint after hydration, a signed-in owner viewing their own `/user/<id>` briefly sees the Follow / Send-Message / Report buttons before they disappear once the auth store resolves. Same pattern as the open row "Owner-view hydration flash on the ProductFunctions bar" (`ProductFunctions.tsx:36-44`) but a separate component — a fix touches `UserDetails` independently. Found by Docs/QA during the QA-Preparation public User page authoring.

**Fix:** auth-derived button visibility now gated on a new shared hook `useAuthResolved` that composes Firebase `onAuthStateChanged` ready-state with `useAuthStore` user-loaded state, mirroring the existing `HeaderNavButtons` precedent. Replaces the `useEffect`/`useMemo` patterns that flashed because they updated state after first paint. The hook is at `src/lib/hooks/useAuthResolved.ts`.

---

## [HIGH] ProductErrorCode is a dumping ground for cross-cutting error codes

**Status:** open
**Surfaced:** 2026-05-14, connection-pool BugFixer chat (CurrentLanguageFilter Phase 2 work)
**Severity:** high — not a runtime bug, but a structural problem that actively misleads. The "Cross-cutting (system-level)" group inside `ProductErrorCode` is a precedent that gets cited and copied; every new cross-cutting code added there makes the next misplacement look more justified.

### The problem

`ProductErrorCode` contains error codes that have nothing to do with products. It already has an explicit "Cross-cutting (system-level)" group, and known occupants include:

- the rate-limit code consumed by `RateLimitFilter`
- `LANG_MISSING_OR_INVALID`, added during this chat by `CurrentLanguageFilter` Phase 2 — the engineer correctly flagged the placement felt wrong but followed the existing precedent because no proper home existed

There may be others — a full audit of `ProductErrorCode.values()` is the first step.

### Why it wasn't fixed in the chat that surfaced it

The chat that introduced `LANG_MISSING_OR_INVALID` is a BugFixer chat. Fixing _only_ the one code we added would leave `ProductErrorCode` inconsistently cleaned — some cross-cutting codes moved, others not — which is worse than leaving it whole, because it removes even the bad-but-consistent pattern. The honest fix is the full refactor: create a proper cross-cutting error-code enum and move _all_ non-product cross-cutting codes out. That refactor touches every consumer of the moved codes, the error-contract wiring, the `ProductErrorCodeTest` seed-row enforcement (which iterates `ProductErrorCode.values()`), and the translation seed files. That is feature-sized work, out of scope for a bug-fixer chat — so it is being routed here deliberately, not parked to dodge it.

### What the fix involves (for whoever picks this up)

1. Audit `ProductErrorCode` — list every constant that is not genuinely product-specific.
2. Decide the target enum name and create it (candidates discussed but not settled: `GeneralErrorCode`, `SystemErrorCode`, `RequestErrorCode`). Confirm whether any suitable enum already exists before creating one.
3. Move all non-product cross-cutting codes to the new enum.
4. Update every consumer — `RateLimitFilter`, `CurrentLanguageFilter`, and whatever else references the moved codes.
5. Update `ProductErrorCodeTest` — and check whether an equivalent seed-enforcement test should exist for the new enum.
6. Translation seed rows for the moved codes: confirm whether the `translationKey` values change (if the keys stay `product.system.*` that's itself a smell worth fixing; if they become `system.*` / `general.*` the seed rows move/rename).
7. `./mvnw spotless:check` + `./mvnw test` green.

### Known scope notes

- `LANG_MISSING_OR_INVALID` currently lives in `ProductErrorCode` with translation key `product.system.lang_missing_or_invalid` and seed rows in all four locale files (`0001-data-web-translations-{EN,RS,RU,CNR}.sql`). The `product.system.*` key prefix on a non-product code is part of the same smell.
- This is a refactor, not a behavior change — done right, no error response changes shape or code value from the client's perspective except the codes' _enum home_ (and possibly their `translationKey` if you choose to rename).

---

## 2026-05-14 — NumberOfViews fires a redundant duplicate fetch

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-numberofviews-redundant-fetch-1)
**Found in:** `oglasino-web/src/components/client/NumberOfViews.tsx:11-13`
**Detail:** The view-count `useEffect` depends on `[numberOfViews]`. It fires once with state at 0, fetches, then `setNumberOfViews(res)` changes the dependency and it fires again — two requests per render for one value, identical responses. No user-visible bug, perf-only. Fix: change the dependency to `[productId]`. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

**Fix:** `useEffect` dependency changed from `[numberOfViews]` to `[productId]`. Same session also added a pre-fetch render guard (`if (numberOfViews <= 0) return null`) to hide the placeholder flash, and a `.catch(() => setNumberOfViews(0))` on the promise chain to swallow rejection cleanly (service layer continues to log via `logServiceError`).

---

## 2026-05-14 — Hardcoded Serbian tooltips on the product detail page

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/server/ProductDetails.tsx:72`, `oglasino-web/src/components/client/NumberOfViews.tsx:16`
**Detail:** The favorites-count and views-count tooltips use literal Serbian strings (`"Koliko je puta proizvod sacuvan"`, `"Koliko je puta proizvod pogledan"`) instead of next-intl keys — EN / RU / ME visitors see untranslated Serbian. A Part 6 (translations) violation in `oglasino-web` source. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

---

## 2026-05-14 — Owner-view hydration flash on the ProductFunctions bar

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-hydration-flashes-1)
**Found in:** `oglasino-web/src/components/client/ProductFunctions.tsx:36-44`
**Detail:** The actions bar's visibility is gated on a `useEffect`-driven `allow` flag (`setAllow(user.id === owner.id)`). On first paint after hydration the signed-in owner briefly sees the Call / Message / Favorite / Share / Review / Report bar before it disappears once the auth store resolves. The owner/stranger boundary itself is correct (captured as a topic pitfall); the flash is unintended. Fix: gate the bar with the auth selector synchronously, or render null while auth is loading. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

**Fix:** auth-derived button visibility now gated on a new shared hook `useAuthResolved` that composes Firebase `onAuthStateChanged` ready-state with `useAuthStore` user-loaded state, mirroring the existing `HeaderNavButtons` precedent. Replaces the `useEffect`/`useMemo` patterns that flashed because they updated state after first paint. The hook is at `src/lib/hooks/useAuthResolved.ts`.

---

## 2026-05-14 — Messages "Report" dropdown item is inert

**Severity:** medium (escalated from low during 2026-05-16 triage)
**Status:** fixed (2026-05-16, session oglasino-web-messages-report-wiring-1)
**Found in:** `oglasino-web/src/messages/components/Messages.tsx:158-160`
**Detail:** The conversation-header kebab dropdown renders a "Report" item (`tMessages('func.report')` label) with no `onClick` handler — visible to every user who opens the dropdown, does nothing on click. May fall under the `state.md` backlog item "Chat & messaging cleanup" (`planned`), but no `issues.md` entry recorded it. Found by Docs/QA during the QA-Preparation Messages page authoring.

**Fix:** wired the inert "Report" dropdown item in `Messages.tsx` to open the existing `REPORT_DIALOG` via `useDialogStore.openDialog(DialogId.REPORT_DIALOG, ...)`, mirroring the payload shape used by `ReportButton.tsx` on the public user page. `reportedUserId` is derived from `activeChat.withUser.id` (the same state the conversation header uses for the avatar/name). Addendum also removed the dead `onSendReport` field from `ReportDialogProps` (zero code consumers; `grep` confirmed) and the corresponding `as ReportDialogProps` cast plus unused import in both `ReportButton.tsx` and `Messages.tsx`.

---

## 2026-05-14 — Messages chat list capped at 15 in the rendered UI

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/components/Chats.tsx`
**Detail:** The conversation list renders only the most-recent Firestore page (limit 15). The chat store implements `loadMoreChats` and tracks `hasMoreChats` / `loadingMoreChats`, but `Chats.tsx` renders no "load more" affordance. A user with more than 15 conversations only ever sees the 15 most-recent by `lastUpdated`; older threads resurface only when the other party sends a new message that bumps `lastUpdated`. Invisible data-loss in the UX — the store supports paging, the UI doesn't expose it. Found by Docs/QA during the QA-Preparation Messages page authoring.

---

## 2026-05-14 — Messages silent text-only send failure

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/store/useChatStore.ts` (`sendMessage` catch branch)
**Detail:** A Firestore write failure on a text-only message send logs `console.error`, runs orphan-image cleanup (a no-op for text), and rolls back the optimistic message by filtering out its tempId. No toast, no inline error, no surfaced state — the sender sees the message simply vanish. Image-upload failures in `MessageInput` do surface as `notify.error` toasts, so this is an asymmetry: a critical action fails silently on the text path. Found by Docs/QA during the QA-Preparation Messages page authoring.

---

## 2026-05-14 — Privacy and Terms render English markdown across all locales

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/privacy/page.tsx:14` (and the terms equivalent)
**Detail:** Both legal pages fetch a hardcoded GitHub raw markdown URL (`memento-tech/oglasino-platform/main/privacy.md` / `terms.md`) with no locale variant — every locale (EN / SR / RU / ME) renders the same English content. `privacy/page.tsx` carries a `// TODO Create locale privacy.md`. Found by Docs/QA during the QA-Preparation simple-pages batch. Surfaced to QA users as a "Known issue" pitfall on the `privacy-page` and `terms-page` QA topics; this entry tracks the underlying code fix.

---

## 2026-05-14 — `markdown.fild.load` translation key is a typo

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/server/MarkdownViewer.tsx:18,22`
**Detail:** `MarkdownViewer` references `tErrors('markdown.fild.load')` — "fild" is almost certainly meant to be "file". The rendered string is whatever the translation row says, so there is no user-visible break unless the row is also misnamed, but the key should be corrected across its registration and all locales. Found by Docs/QA during the QA-Preparation simple-pages batch.

---

## 2026-05-14 — Terms page does not emit JSON-LD; Privacy page does

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/terms/page.tsx`
**Detail:** `privacy/page.tsx` emits a JSON-LD `<script>` tag for SEO; `terms/page.tsx` does not. The two legal pages are otherwise the same shape. SEO inconsistency — either Terms should emit JSON-LD too, or the difference should be deliberate and documented. Found by Docs/QA during the QA-Preparation simple-pages batch.

**Investigated 2026-05-16 (terms-json-ld-1 session).** The visible asymmetry is a symptom of a broader SEO delivery bug — both pages already emit JSON-LD via `metadata.other['application/ld+json']`, but Next.js renders that as `<meta>` tags which search engines do not parse as structured data. Privacy additionally has a malformed inline `<script>` that stringifies the Next.js `Metadata` object instead of the Schema.org helper. Both findings rolled into the 2026-05-16 entry "SEO JSON-LD delivered as `<meta>` tags instead of `<script>`, project-wide." This entry remains open and will close when the broader fix lands.

---

## 2026-05-14 — Pricing hero renders the same translation key on both subtitle lines

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/pricing/page.tsx:32-34`
**Detail:** The hero H1 calls `tPricing('hero.subtitle.line1')` on both of its two lines — almost certainly meant to be `line1` then `line2`. Result is a visible duplicate line under the hero title in all locales. Found by Docs/QA during the QA-Preparation simple-pages batch.

---

## 2026-05-14 — Bad catalog slug produces two different failure modes depending on position

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:92-94` (server-side first-slug redirect to home); `oglasino-web/src/components/client/product/ProductBreadcrumbs.tsx:55-61` (client-side fallback to `window.location.href = '/error'` when the breadcrumb chain resolves empty on `/catalog`).
**Detail:** A catalog slug that does not resolve produces two different observable outcomes depending on where it sits in the URL: an invalid first slug server-redirects to home, while a partially-broken breadcrumb chain has a client-side fallback to `/error`. Same root cause — a slug that doesn't resolve — two different results. Intended behavior per Igor: messing with catalog routing should show a "category not found" error, consistently. Same root cause should produce the same outcome. Related to the silent-degradation entry below — same root cause, different observable outcome. Cross-referenced from the `category-navigation` and `catalog-page` QA topics as Known-issue pitfalls; when this bug is fixed, the topic pitfalls must flip to reflect the new behaviour.

---

## 2026-05-14 — Invalid catalog sub-slug degrades silently to parent category

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:38-44` (sub/final slug resolution — `subCategory` and `finalCategory` stay `undefined` on a bogus slug, and the page renders against the deepest level that did resolve with no error and no redirect).
**Detail:** An invalid second or third URL slug on the catalog route degrades silently to the parent category level — `/catalog/cars/bogus` renders identically to `/catalog/cars`, with no error, no redirect, no "category not found". A bad sub-slug produces a result that looks like the parent category, which can read as the parent filter being broken. Intended behavior per Igor: a bad slug at any level should show a "category not found" error, not silently fall back to the parent. Related to the two-failure-modes entry above — same root cause. Cross-referenced from the `category-navigation` and `catalog-page` QA topics as Known-issue pitfalls; when this bug is fixed, the topic pitfalls must flip to reflect the new behaviour.

---

## 2026-05-14 — `perPage` is uncapped server-side

**Severity:** medium
**Status:** open
**Found in:** `oglasino-backend` — `PagingDTO.getPageable()` → `PageRequest.of(page, perPage)`, called from `DefaultProductsFilterQueryBuilder.buildQuery`
**Detail:** All product-search endpoints (`/api/secure/products`, `/api/secure/admin/products`, `/api/public/product/search`, plus their three autocomplete siblings) pass client-supplied `perPage` straight to Spring Data with no maximum. `RateLimitFilter` mitigates abuse, but a hard cap (e.g. 100) is appropriate hygiene. Discovered in the product-filtering bug sweep (backend audit, Q3).

## 2026-05-14 — `PORTAL_SEARCH` silently drops body `productStates` / `moderationStates`

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/ProductStateQueryGenerator.java` (`PORTAL_SEARCH` branch)
**Detail:** Public product search hard-pins `productState=ACTIVE AND moderationState=APPROVED` and ignores any client-supplied state filters. This is correct and intentional behavior — a public caller must not be able to request PENDING/REJECTED products — but the shared `ProductsFilterDTO` wire shape doesn't communicate that these fields are inert on the portal path. No code fix wanted; logged so the silent-drop is recorded. Discovered in the product-filtering bug sweep (backend audit, opportunistic flags).

## 2026-05-14 — `baseSite` tenant scoping was never audited

**Severity:** medium
**Status:** open
**Found in:** `oglasino-backend` — `BaseSiteQueryGenerator`, `BaseSiteContext`
**Detail:** `BaseSiteQueryGenerator` and `BaseSiteContext` were referenced but not opened during the product-filtering audit. Whether tenant isolation is header-derived or body-derived is unconfirmed. Needs its own focused audit; will likely become its own documentation page. Discovered in the product-filtering bug sweep (backend audit, out-of-scope note).

## 2026-05-14 — Backend errors are swallowed and rendered as "empty results"

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/lib/service/nextCalls/productsSearchService.ts` (all three `getXProducts` calls)
**Detail:** Every product-search server call catches errors and returns `EMPTY_OVERVIEWS = { products: [], totalNumberOfProducts: 0 }`. A backend outage is therefore indistinguishable from a genuinely empty result set — the user sees "no products" either way. Pre-launch this is tolerable; for production it warrants at least server-side logging and ideally a distinguishable error state. Discovered in the product-filtering bug sweep (web audit, all surfaces).

## 2026-05-14 — No request cancellation on the pagination path

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/product/ProductList.tsx`, `src/lib/service/reactCalls/productsSearchService.ts` (paging branch)
**Detail:** `AbortController` was added to autocomplete during the sweep, but paging requests still have no cancellation — rapid page-button clicks can let a stale response resolve last and win. Deferred deliberately; would need a coordinated pattern across `ProductList` and the service layer. Discovered in the product-filtering bug sweep.

## 2026-05-14 — `parseFiltersFromQueryParams` has a dead `| undefined` return type

**Severity:** low
**Status:** fixed (2026-05-15, session oglasino-web-filtersHelper-return-type-1)
**Found in:** `oglasino-web/src/lib/utils/filtersHelper.ts:44`
**Detail:** The function is typed `ProductsFilterDTO | undefined` but no code path returns `undefined`; the home page's `else` branch handling that case (`app/[locale]/(portal)/(public)/page.tsx:38-43`) is unreachable. Dead code, no behavior impact — a type-tightening cleanup. Discovered in the product-filtering bug sweep (web session summary).

**Fix:** return type narrowed; two consumer-side dead branches removed (home `if/else`, catalog `filtersData &&` guard); three other callers confirmed already-correct and untouched.

## 2026-05-14 — `/admin/products/[userId]` renders an empty fragment on zero products

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-admin-filter-pipeline-fix-1)
**Found in:** `oglasino-web/app/[locale]/admin/products/[userId]/page.tsx:34`
**Detail:** When an admin views a user with no products, the page returns `<></>` — no message, no context. Should show a "no products for this user" state. Discovered in the product-filtering bug sweep (web audit, Suspected Bug context).

**Fix:** replaced `<></>` with the same conditional empty-state-vs-list pattern used by the sibling `/admin/products` page. Renders `tDash(filtersApplied ? 'products.filters.empty.list' : 'products.empty.list')` in a centred italic empty-state box, with the filter chip row and side panel still visible. Reused existing `DASHBOARD_PAGES` keys; no new translation key needed.

## 2026-05-14 — `/admin/products/[userId]` filter chips change without refetch or URL sync

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-admin-filter-pipeline-fix-1)
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx:52-66` (`isAllowedPath()` excludes `/admin/products/[userId]`)
**Detail:** Two root causes, both addressed in the fix session: (1) `FilterManager.tsx`'s HYDRATE FROM URL effect had deps `[baseSite]` and ignored `pathname` changes, so in-app nav from a disallowed admin route left `hydratedRef.current === false` and the SYNC effect silently early-returned on chip clicks — the dominant root cause shared with the 2026-05-16 entry on `/admin/products`. (2) `/admin/products/[userId]` was excluded from `FilterManager.isAllowedPath()` entirely; even with the deps-array bug fixed, the path was still gated off. Fix: replaced `hydratedRef` boolean with per-path `lastHydratedPathRef` and added `pathname` to deps; extended `isAllowedPath()` to use `pathname.startsWith('/admin/products')` instead of exact match; added defensive `setTimeout(() => router.refresh(), 0)` after `router.replace` to match the established admin pattern at `src/components/admin/FiltersPanel.tsx:65-69`. The same deps-array bug had a latent twin on `/owner/products`; incidentally fixed by the same change.

## 2026-05-14 — `conventions.md` says `Auth: Firebase Auth (JWT)` and references "claims from the JWT"

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-docs/meta/conventions.md` Part 9 (stack table) and Part 11 (trust-boundary text)
**Detail:** Part 9's stack table and Part 11's trust-boundary text both describe auth as JWT-based. The actual system uses Firebase auth as the source of truth (Firebase-context-derived identity server-side), confirmed by the backend filtering audit (`SearchModeQueryGenerator.DASHBOARD_SEARCH` reads from `SecurityContextHolder` populated by `FirebaseAuthFilter`). Same class of error as the recently-corrected Gradle/Maven mismatch — a convention that contradicts reality and will mislead a future agent. Needs an Igor-and-Mastermind correction pass on Parts 9 and 11. Discovered in the product-filtering bug sweep. **Resolved 2026-05-15:** Parts 9 and 11 corrections drafted in `.agent/2026-05-15-oglasino-docs-connection-pool-closeout-1.md` (For Mastermind section) and pending Igor's apply to `conventions.md` — Docs/QA does not edit `conventions.md` per CLAUDE.md hard rule. The drafted change replaces the JWT-mechanism framing with Firebase ID token + `SecurityContextHolder`-populated `OglasinoAuthentication`, using the precise codebase terms (`FirebaseAuthFilter`, `redisUserAuth`).

## 2026-05-14 — Product-validation feature doc needs a retro-fit pass against current conventions

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-docs/features/product-validation.md`
**Detail:** Written before the `### Images` convention (Part 1) and the Part 5 session-naming change existed. It should be brought in line with the current rulebook. Not urgent — tagged for a future docs session, not part of any active work. **Resolved 2026-05-15:** audited against current conventions during the connection-pool close-out session. Part 1 `### Images` is satisfied — 4 image references at lines 417, 465, 505, 566 all use kebab-case lowercase descriptive filenames under `assets/`, all have a preceding HTML comment describing the intended screenshot, and all use standard markdown syntax with descriptive alt text. Part 5 (session-file naming) governs `.agent/` summary files, not feature-spec docs, so it is N/A for this file. No retro-fit edits needed.

## 2026-05-14 — Manual test reminder: pagination overlap across all five surfaces

**Severity:** low
**Status:** open
**Found in:** product-filtering bug sweep — manual QA queue
**Detail:** After the product-filtering web fixes, verify on a live stack that dashboard/admin/public-user pages 2+ show different products than page 1, on a scope with 25+ products. Igor ran an initial check during the sweep and it passed; this entry is the standing regression reminder. **Not a bug** — verification step that should not be lost. Surfaces in scope: home, category, owner dashboard, admin product list (including `[userId]`), public user-products.

## 2026-05-14 — Manual test reminder: random-ordering stability across pages

**Severity:** low
**Status:** open
**Found in:** product-filtering bug sweep — manual QA queue
**Detail:** Verify that paginating home/catalog with random ordering keeps a consistent shuffle across pages within one session, and reshuffles only on refresh (because `randomSeed` is regenerated per SSR call but stable across paging requests within one render). **Not a bug** — verification step.

## 2026-05-14 — Dead `free` field on backend `NewProductRequestDTO`

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java` (field `private boolean free` plus `isFree()`/`setFree()`)
**Detail:** Web removed `free` from its `NewProductRequestDTO` interface in web session 4 (free-zone derived from `topCategory.freeZone` instead). Backend still carries the field with getter/setter; no production code reads it (grep across `src/main/java` shows zero call sites that consume `request.isFree()` or `requestDTO.isFree()` on a product request). Jackson tolerates the absent field on incoming JSON. Dead field worth deleting in a follow-up chore to match the trust-boundary direction (free-zone is server-derived).

## 2026-05-14 — `regionAndCity` declared in `createProductSchema` but unread by validator

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-regionAndCity-cleanup-1)
**Found in:** `oglasino-web/src/lib/validators/productSchemas.ts`
**Detail:** `createProductSchema` still declares `regionAndCity: z.object({...}).nullable().optional()` but `validateProduct` does not consult that field — region and city are user-derived and not user-settable on the create wizard. Harmless as-is; cosmetic cleanup.

**Fix:** one-line deletion in `productSchemas.ts`; grep confirmed zero consumers and zero type derivations. `priceSchema` `export` investigated as adjacent and kept (legitimate cross-module consumer in `productValidator.ts`).

## 2026-05-14 — `field-price` scroll target is a no-op on update page

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-field-price-scroll-target-2)
**Found in:** `oglasino-web/app/[locale]/owner/products/[productId]/page.tsx`
**Detail:** When a price-only error fires on save, the page calls `document.getElementById('field-price')?.scrollIntoView(...)`, but the two price `<Input>` renders generate their own internal ids and do not accept an external `id`. Inline error still renders correctly next to the price input; only the scroll-into-view is a no-op. Fix is to wrap the price inputs in a `<div id="field-price">`.

**Fix:** rewrote to viewport-aware `data-field="price"` lookup on both responsive wrappers (mobile and desktop). Both client-validation and server-error scroll pickers updated to pick the visible element via `querySelectorAll` + `offsetParent`. Same session also moved `field-images` id from the conditional error `<p>` to the unconditional outer wrapper for pattern consistency, and added a system-error scroll case (`SYSTEM_ERROR_KEY` branch wrapping the system-error `<p>` in `<div id="field-system-error">`) so rate-limit-style errors auto-scroll like per-field errors.

## 2026-05-14 — Boot-time moderation config audit logs ERROR but does not fail boot

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/moderation/ContentValidationConfig.java` (`auditRequiredConfig`)
**Detail:** Missing/blank required keys are surfaced at boot via `log.error(...)` but the listener does not throw — the first runtime moderation call against an absent key throws `IllegalStateException`. Acceptable for dev environments that may run with an un-seeded config table. A `@Profile`-gated boot-fail (or env-toggle) for production would convert the silent-misconfig window into a hard startup failure. Framed as enhancement, not defect.

## 2026-05-14 — Web component-render test coverage gap (testing-library not installed)

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/package.json`
**Detail:** `@testing-library/react`, `@testing-library/user-event`, `happy-dom`, and `jsdom` are all absent. Web tests are pure-function and axios-mock only — dialog-rendering assertions (button-disabled, spinner present, inline-error elements) are covered indirectly via logic-level tests against `validateProduct`, `preValidateProductBasics`, and `productStepMapping`. End-to-end smoke is the first surface that exercises the dialog rendering. Future infra chore: install the testing stack so dialog tests can land in thin follow-ups.

## 2026-05-14 — Keyword-stuffing ratio multipliers not lifted to config

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/moderation/analyzer/KeywordStuffingAnalyzer.java`, `SpammyDescriptionAnalyzer.java`
**Detail:** Both analyzers pass a `KeywordStuffingDetector.Tuning` record with hardcoded `baseRatio`, `ratioDecayPerWord`, `ratioFloor`, and `minOccurrenceRatio` values. Other thresholds are admin-configurable via `ConfigurationService`; these heuristic ratios are not. Deliberately out of scope for the validation refactor — lifting them is a separate, larger conversation about how much of the heuristic should be admin-tunable.

## 2026-05-14 — Malformed 429 leaves create-wizard silently blocked

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/lib/service/reactCalls/productService.ts`
**Detail:** Web session 4 removed the defensive `ensureSystemErrorKey` helper that synthesised a `__system` rate-limit entry if the 429 response body was malformed. Trade-off was made deliberately (trust the contract, surface upstream bugs instead of papering over them). Consequence: a malformed 429 (no `field:null + translationKey` entry) leaves the create wizard's Next button blocked without showing the inline rate-limit reason — the user is not advanced through, just not told why. If a malformed 429 is ever observed in manual/QA testing, the fix belongs at the backend/transport source, not at the client.

## 2026-05-13 — Legacy unused regex rows in data-configuration.sql

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/resources/data/configuration/data-configuration.sql` ids 2–7
**Detail:** Six pre-feature config rows (`validation.regex.banned.words`, `validation.regex.repeated.chars`, `validation.regex.punctuation`, `validation.regex.emojis`, `validation.regex.promo` without language suffix, `validation.regex.spam.description`) remain seeded but have zero call sites — superseded by the per-language `validation.regex.*.{lang}` and `validation.banned_words.{lang}` rows. Backend session 3 flagged these for Mastermind as a follow-up "config-seed garbage collection" PR. Safe to delete in a chore.

## 2026-05-13 — 429 rate-limit response shape diverges from error contract

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/security/filter/RateLimitFilter.java`
**Detail:** Original report: on HTTP 429, `RateLimitFilter` returned `{"error": "rate_limit_exceeded"}` with a `Retry-After` header, mismatching the project's `{errors: [{field, code, translationKey}]}` contract. **Resolved in backend session 1**: filter now writes `{"errors":[{"field":null,"code":"RATE_LIMITED","translationKey":"product.system.rate_limited"}]}` (see `RATE_LIMITED_BODY` constant). The unified shape applies to all rate-limited endpoints.

## 2026-05-13 — Commented-out `REGION_REQUIRED` annotation in `NewProductRequestDTO`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`
**Detail:** Original report: a commented-out `@NotNull(message = "REGION_REQUIRED")` line sat above the `filters` field. Region is derived from the user, not the request. **Resolved**: the commented line is no longer present in the file (deleted during one of the cleanup passes on the validation-refactor branch).

## 2026-05-13 — Pre-existing spotless violations on validation-refactor branch

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`, `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java`
**Detail:** Two files on `feature/validation-refactor` failed `mvn spotless:check` before the pre-validate session began. One needed re-indentation of a commented line; the other had an unused `Region` import. Both fixed by Igor via `mvn spotless:apply` and committed separately under `chore: fix pre-existing spotless violations`.
