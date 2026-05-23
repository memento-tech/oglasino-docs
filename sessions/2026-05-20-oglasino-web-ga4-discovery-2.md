# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit `oglasino-web` for the current state of analytics instrumentation, identity availability, page-view firing surfaces, event-firing surfaces, and the infrastructure pieces a v1 GA4 build needs. Code is ground truth.

> **Audit note.** A prior file `2026-05-19-oglasino-web-ga4-discovery-1.md` exists in `.agent/`, so per conventions Part 5 this session is `<n>=2`. The brief's parenthetical "this is the first analytics audit" is therefore inconsistent with on-disk state. I respected the brief's explicit "do not read prior session summaries about analytics" instruction and did not open the 2026-05-19 file. Flagging the discrepancy for Mastermind (see "For Mastermind").

---

## Findings

Status legend: `present` / `partial` / `absent` / `n/a`. Severity legend: `blocking` / `friction` / `nice-to-have`.

### 1. Existing analytics SDKs and tracking code

- **No GA4, GA, GTM, or `dataLayer` references anywhere in source.** Grep for `gtag|dataLayer|googletagmanager|google-analytics|analytics\.google\.com` across `src/`, `app/`, `jobs/`, and the root config files returns hits only in `package-lock.json` (transitive resolution, no direct dep).
  - **Files:** none in source. Verified at `package.json:27-92` (no `gtag.js`, `react-ga4`, `@vercel/analytics`, or any analytics dep listed).
  - **Status:** `absent`
  - **Severity:** `blocking` — v1 cannot ship without adding the SDK; this is the work itself, not a gap.
  - **Note:** Greenfield. No prior wiring to migrate or remove.

- **No competing analytics tools (Mixpanel, PostHog, Amplitude, Segment, Heap).** Grep across source: zero hits.
  - **Files:** none.
  - **Status:** `absent`
  - **Severity:** `nice-to-have` — confirms the "GA4 only" decision has no contradicting precedent to displace.

- **No `next/script` usage at all in the repo.** Grep for `from 'next/script'` and `<Script` across `src/` and `app/`: zero hits.
  - **Files:** none.
  - **Status:** `absent`
  - **Severity:** `friction` — there is no precedent in this repo for `next/script` strategy choice; the GA4 work picks the strategy from scratch.

- **`next.config.ts` has no analytics-related script configuration.** The file's only declared concerns are bundle-analyzer wrapping (`bundleAnalyzer({ enabled: process.env.ANALYZE === 'true' })`), next-intl plugin wrapping, `reactStrictMode: false`, and a `serverActions.allowedOrigins` allow-list for the Cloudflare-fronted hostnames.
  - **File:** `next.config.ts:1-32`
  - **Status:** `n/a`
  - **Severity:** `nice-to-have`
  - **Note:** v1 likely does not need `next.config.ts` edits at all — script tags can be rendered from the layout components.

### 2. Identity — who is the user

- **Authenticated user lives in `useAuthStore.user: AuthUserDTO | null`.** `AuthUserDTO` carries `id: number` (backend's userId), `firebaseUid: string`, `displayName: string`, `email: string`, and the rest of the profile. The numeric `id` is the canonical, stable, non-PII identifier across web and backend; `firebaseUid` is the Firebase stable identifier; both are safe for GA4 `user_id`.
  - **Files:** `src/lib/store/useAuthStore.ts:126-295` (store), `src/lib/types/user/AuthUserDTO.ts:4-22` (shape).
  - **Status:** `present`
  - **Severity:** `nice-to-have` — identity is fully available; GA4 wiring just consumes it.

- **Single sign-in completion seam at `UseTokenRefresh.tsx:38-42`.** `onIdTokenChanged` is the sole listener (a superset of `onAuthStateChanged` — fires on sign-in, sign-out, user replacement, and token rotation), `syncUserToBackend(firebaseUser)` returns the freshly synced `backendUser`, and the listener calls `useAuthStore.getState().setUser(backendUser)`. This is the unified completion signal for every sign-in path (email/password, Google, Facebook, register) and every token-rotation refresh.
  - **File:** `src/components/client/initializers/UseTokenRefresh.tsx:18-43`
  - **Status:** `present`
  - **Severity:** `nice-to-have` — a single instrumentation point covers both `sign_up` and `login`, but the listener does **not** today have a way to tell `sign_up` apart from `login` (see Finding under Scope 4, event 1).

- **Unauthenticated users: no stable cross-session identifier exists in web today.** `useAuthStore.user` is `null` until Firebase resolves; no anonymous client ID is generated, persisted, or sent. The `getGlobalCookie()` cookie carries portal preferences (card size, cookie-consent, etc.) but no analytics-style anonymous ID.
  - **Files:** `src/lib/service/oglasinoCookies.ts` (cookie shape), `src/lib/types/cookie/GlobalCookie.ts`
  - **Status:** `absent`
  - **Severity:** `friction` — GA4's `gtag.js` generates its own `_ga` client ID for anonymous users, so v1 can ship without a custom one. The friction is that custom anonymous joining (e.g., correlating pre-login activity with the user_id after login) is not possible from the existing cookie surface.

- **`useAuthResolved` is the existing gate for auth-dependent rendering.** Returns `true` once Firebase has reported auth state AND, if a Firebase user exists, the auth store's `backendUser` is non-null. Auth-gated UI renders `null` until resolved.
  - **File:** `src/lib/hooks/useAuthResolved.ts:13-27`
  - **Status:** `present`
  - **Severity:** `friction` — GA4 should hook into this resolution timing for any event whose payload includes a `user_id`. Events fired before resolution would be missing the parameter. The natural pattern: fire `page_view` immediately on route change (it doesn't need `user_id`), but defer or re-fire events that include `user_id` until after `useAuthResolved()` is `true`.

### 3. Page-view firing surface — App Router + locales

- **No global client component currently listens for route changes for analytics.** The repo has exactly one client component reading `usePathname()` + `useSearchParams()` at the locale layout level: `NavigationProgressBar.tsx`. It uses the URL transition as a "navigation arrived" signal to reset its in-flight indicator. There is no other top-level pathname/searchParams listener.
  - **Files:** `src/components/client/NavigationProgressBar.tsx:20-26` (the existing listener), `app/[locale]/layout.tsx:38` (mounted in the locale layout)
  - **Status:** `partial` (the hook host exists; the analytics consumer doesn't yet)
  - **Severity:** `nice-to-have` — `app/[locale]/layout.tsx` is the natural host for a sibling `GA4RouteListener` client component that fires `page_view` on URL change. The architectural pattern is already proven by `NavigationProgressBar`.

- **`app/layout.tsx` (root) is a server component.** It renders `<html><body>` with `FirebaseWorkerInit`, `ConfigProvider`, `ThemeProvider`. The `<head>` is implicit (no explicit `<head>` element).
  - **File:** `app/layout.tsx:32-51`
  - **Status:** `present` (as the natural injection home for the GA4 `<script>` tag via `next/script`)
  - **Severity:** `nice-to-have`

- **`app/[locale]/layout.tsx` is also a server component, fronted by `NextIntlClientProvider`.** This is where the route-listener client component naturally mounts (same level as `NavigationProgressBar`).
  - **File:** `app/[locale]/layout.tsx:37-63`
  - **Status:** `present`
  - **Severity:** `nice-to-have`

- **Catalog page is a fully server-rendered route.** `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:82` reads both `params` and `searchParams` on the server; the page's `getPortalProducts(...)` call uses the search params for the initial fetch. Pagination's "load more" pages are fetched client-side, but the initial product list is server-rendered on every URL change.
  - **File:** `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:82-178`
  - **Status:** `present`
  - **Severity:** `nice-to-have`

- **Filter changes drive a client-side URL replacement that triggers server re-render.** `FilterManager.tsx:316-318` calls `router.replace(newUrl, { scroll: false })` then `setTimeout(() => router.refresh(), 0)`. The browser URL changes, `usePathname()` stays the same, but `useSearchParams()` flips — a listener on `[pathname, searchParams]` will see every filter mutation as a "navigation."
  - **File:** `src/components/client/initializers/FilterManager.tsx:228-329` (the URL-sync effect, including the `router.replace` + `router.refresh` at lines 316-317)
  - **Status:** `present`
  - **Severity:** `friction` — the volume of `page_view` firings per session would inflate sharply if every filter tweak (price slider, currency switch, region toggle, sort order) counts as a page-view. The decision belongs to Mastermind; flagged in "For Mastermind."

- **Home page (`/`) is also a fully server-rendered catalog-like route, also reading searchParams** at `app/[locale]/(portal)/(public)/page.tsx:24-67`. Same `page_view`-on-filter-change question applies.
  - **File:** `app/[locale]/(portal)/(public)/page.tsx:24-67`
  - **Status:** `present`
  - **Severity:** `friction` (same question as catalog)

### 4. Event-firing surfaces, per the 12-event set

- **`sign_up`** — completion signal lands at `UseTokenRefresh.tsx:38` (`const backendUser = await syncUserToBackend(firebaseUser);` followed by `setUser(backendUser)` at line 39). The listener fires uniformly for sign-up AND login; the only "this was a registration" hint is the `nextRegisterDisplayName` one-shot cell at `authService.ts:28,141-142,187`, which is **consumed and nulled inside `syncUserToBackend` (line 142) before the listener sees the result.**
  - **File:** `src/components/client/initializers/UseTokenRefresh.tsx:38-42`; signal source `src/lib/service/reactCalls/authService.ts:28,141-142,187`
  - **Status:** `partial` — surface exists but the `sign_up` vs `login` discriminator is consumed too early for the listener to use.
  - **Severity:** `friction` — fix is local (carry the register flag through the sync result, or fire `sign_up` from the `registerUserFirebase` call site itself and accept that the user isn't yet in the store at that moment).
  - **Note:** Single handler if the discriminator is moved; otherwise instrumentation must be split between `registerUserFirebase` (sign_up at primitive completion) and `UseTokenRefresh` (login at listener firing).

- **`login`** — same surface as `sign_up`: `UseTokenRefresh.tsx:38-42` after `syncUserToBackend` returns a non-null `backendUser`. Discriminator: `nextRegisterDisplayName` is `null` at sync time → it's a login. (Same caveat as above.)
  - **File:** `src/components/client/initializers/UseTokenRefresh.tsx:38-42`
  - **Status:** `present`
  - **Severity:** `nice-to-have` — single handler, simple wiring.

- **`product_view`** — product detail page is the server component at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:48-106`. A server-side `markAsSeen(productDetails.id)` is already called conditionally at line 84 (when `!owner.iamActive`). GA4 events fire from the **client**, so the firing surface must be a client child: the natural candidates are `src/components/server/ProductDetails.tsx` (currently a server component — would need a sibling client component) or a dedicated `ProductViewTracker` mounted from the page.
  - **Files:** `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:48-106`; client children `src/components/client/ProductFunctions.tsx:22-76`, `src/components/client/UserDetails.tsx`
  - **Status:** `partial` — server surface present; client firing surface absent.
  - **Severity:** `friction` — straightforward but requires adding a new client component.

- **`product_create_started`** — entry to step 1 of the wizard. The wizard is a multi-step dialog: `src/components/popups/dialogs/CreateNewProductDialog.tsx:87-159`. Initial `productData` state is set at the effect at lines 101-116; step 1 (`ImageSelectionProductDialog`) is rendered at lines 139-147.
  - **File:** `src/components/popups/dialogs/CreateNewProductDialog.tsx:101-116` (state init = "started")
  - **Status:** `present`
  - **Severity:** `nice-to-have` — single handler on mount.

- **`product_create_completed`** — success branch at `src/components/popups/components/UploadedProductDialog.tsx:62-65`. After `createNewProduct(productData, {...})` resolves with `result.type === 'success'`, `setUploadSuccess(true)` and `onFinish?.()` fire. The backend's response shape includes `id` and `name` (used at line 63 for the product URL).
  - **File:** `src/components/popups/components/UploadedProductDialog.tsx:62-65`
  - **Status:** `present`
  - **Severity:** `nice-to-have` — single handler.

- **`contact_seller_clicked`** — three-button surface on the product detail page (`ProductFunctions.tsx:40-75`):
  1. **Phone call** — `src/components/client/buttons/CallUserButton.tsx:37-64`, fires at line 63 (`window.location.href = 'tel:...'`). Logged-out and not-allowed paths intercept earlier and don't constitute a "clicked" event.
  2. **Start message** — `src/components/client/StartMessageButton.tsx:33-72`, navigation to `/messages` happens at lines 62 and 70 (post-unblock path also lands at line 62).
  3. **Favorite** — `src/components/client/buttons/FavoriteButton.tsx` (not read in this audit; the click is the surface).
  - Plus, per the brief, **messages-page entry buttons** that initiate a message from QA-Preparation's `messages-page` topic — these were not enumerated in this read-only pass beyond the product-page surface; flag for confirmation. The messages-page entry point lives in `src/messages/`.
  - **Files:** see above.
  - **Status:** `partial` — three product-page buttons inventoried; messages-page entry buttons not exhaustively enumerated.
  - **Severity:** `friction` — fragmented across at least three click handlers; benefits from a shared `track('contact_seller_clicked', { method })` helper rather than three inline `gtag(...)` calls.

- **`message_sent`** — single sender at `src/messages/store/useChatStore.ts:372` (`sendMessage`). Success-only firing must wait for the Firestore commit; the two paths are:
  - **New chat:** `await batch.commit()` at line 505 (creates chat doc + both userchats refs + first message in one batch).
  - **Existing chat:** `await addDoc(collection(chatRef, 'messages'), messageRequest)` at line 520.
  Optimistic message replacement (with the real Firestore ID) happens at lines 562-573 inside the `try`; firing after replacement guarantees success. The `catch` at line 581 swallows errors and does **not** fire — correct for "success only."
  - **File:** `src/messages/store/useChatStore.ts:372-580`
  - **Status:** `present`
  - **Severity:** `nice-to-have` — single handler. Note: the brief flags an "is this the FIRST message in the conversation" condition; `isNewChat` at line 391 already discriminates.

- **`search`** — global header search submit at `src/components/client/SearchInput.tsx:117-133` (`commitSearch`). Fired by Enter-key (line 140 → `commitSearch(trimmed)`) and by the "View all results" button (line 199 → `commitSearch(debouncedTerm)`). The same `SearchInput` is used in three scopes (portal header, owner dashboard, admin) via `SelectableSearchInputWrapper`, but they all route through `commitSearch`.
  - **File:** `src/components/client/SearchInput.tsx:117-133`
  - **Status:** `present`
  - **Severity:** `nice-to-have` — single handler.

- **`view_search_results`** — discriminator is the URL query param `search_text`, set by `SearchInput.tsx:132` (`router.push('${targetPath}?search_text=${toQueryParam(rawTerm)}')`). The catalog/home page renders server-side with that param available; the client-side hydration of the same value lives in `FilterManager.tsx:115-117` (`setSearchText(params.get('search_text') ? ... : undefined)`). Firing at either side works; the client-side hook from `FilterManager` or from the route listener is the more natural home.
  - **Files:** `src/components/client/initializers/FilterManager.tsx:113-117`; setter at `src/components/client/SearchInput.tsx:132`
  - **Status:** `present` (as discriminator); `absent` (no firing yet)
  - **Severity:** `nice-to-have`

- **`select_promotion`** — **the repo has no "promoted product" / "popular" / "featured" surface today.** Grep for `promoted|isPromoted|popular|featured|highlighted` in `src/` and `app/` returns hits only in `app/[locale]/design/topics.ts` (QA-prep doc strings). The Free Zone page exists at `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx` but is an informational/marketing page with `JoinFreeZoneButton.tsx:16-26` (a CTA, not a promoted-product click). The "extra products" carousel at `src/components/client/product/ExtraProductsComponent.tsx:18-50` is a generic related-products surface, not a promotion-distinguished surface.
  - **Files:** `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx`; `src/components/client/JoinFreeZoneButton.tsx`; `src/components/client/product/ExtraProductsComponent.tsx`
  - **Status:** `absent`
  - **Severity:** `blocking` — `select_promotion` requires a "what is a promotion?" answer that the codebase does not yet provide. Flag for Mastermind.

- **`exception`** — two error boundaries plus scattered `console.error` calls:
  - **Global error boundary:** `app/global-error.tsx:5-46` — `console.error` at line 7 inside `useEffect`. Renders a full-page fallback with a "Try again" button.
  - **Route error boundary:** `app/error.tsx:5-23` — same pattern, `console.error` at line 7.
  - **Scattered `console.error` calls:** auth store (`useAuthStore.ts:185, 202, 219, 290`), chat store (`useChatStore.ts:582`), service-layer helpers (`src/lib/utils/serviceLog.ts` is the centralizing helper).
  - **Files:** `app/global-error.tsx:7`, `app/error.tsx:7`; `src/lib/utils/serviceLog.ts`
  - **Status:** `partial` — boundaries exist; no telemetry layer.
  - **Severity:** `friction` — instrumentation must wire into the error boundary `useEffect` AND a `window.onerror` / `unhandledrejection` listener to catch errors that don't trip a boundary.

- **`form_submit_failed`** — multiple form-error surfaces, each with its own `setError` / `setErrorMessage` state pattern. Inventory:
  - `RegisterDialog.tsx:58-60` (`setError`), display at the form's error block (around line 133).
  - `LogInDialog.tsx:45-47` (analogous `setError`).
  - `BasicInfoProductDialog.tsx:60-143` — combined Zod (client) + `preValidate` (backend) errors merged into `setProductErrors(zodErrors|result.errors)` at lines 104 and 119.
  - `UploadedProductDialog.tsx:66-72` — backend validation errors from `createNewProduct` (the step-4 final submit).
  - `app/[locale]/owner/user/page.tsx:98-200` — profile-update form (existing `errorMessage` state).
  - **Files:** see above.
  - **Status:** `partial` — error-rendering surfaces present; no shared firing pattern.
  - **Severity:** `friction` — fragmented across forms. The natural wiring is to instrument each `setError(...) / setErrorMessage(...) / setProductErrors(...)` site with a `track('form_submit_failed', { form, code })` adjacent call, but no helper exists today and the conventions Part 6/Part 7 error code shape (`product.<field>.<code>`) is the natural payload.

### 5. PII surface analysis

GA4 forbids raw email, phone number, raw address, payment data, and any other PII in event parameters. The v1 events shouldn't carry PII by design, but the temptation surfaces per event are:

- **`sign_up`** — `RegisterUserRequestDTO` (`src/lib/types/user/RegisterUserRequestDTO.ts:1-5`) carries `displayName`, `email`, `password`. The post-sync `AuthUserDTO` carries `email` and `displayName` plus the safe `id: number` and `firebaseUid: string`. **Safe to send:** `id`, `firebaseUid`, `baseSiteCode`, optionally registration provider (email/google/facebook). **Do not send:** `email`, `password`, `displayName` (display names are user-supplied free text; even when they appear to be a nickname, they may be a real name).
  - **Status:** `present` (clear separation between PII and non-PII fields)
  - **Severity:** `nice-to-have`

- **`contact_seller_clicked`** — seller is a `UserInfoDTO` (`src/lib/types/user/UserInfoDTO.ts:4-20`) with `id: number`, `firebaseUid: string`, `displayName`, `profileImageKey`, plus product context. **Safe to send:** seller's `id`, the product's `id`, the contact method (`call|message|favorite|share`). **Do not send:** seller's `displayName`, the seller's phone number (returned by `getUserPhoneNumber(userId)` in `CallUserButton.tsx:53`).
  - **Status:** `present`
  - **Severity:** `nice-to-have`

- **`message_sent`** — sender and receiver are both `UserInfoDTO`. **Safe to send:** sender's `id`, receiver's `id`, chat-id (the deterministic `[uid1, uid2].sort().join('_')` from `useChatStore.ts:380` is derived from Firebase UIDs — not PII per GA4's definition but not user-friendly either; safer to derive a hashed-or-numeric chat id), the message-type metadata (text vs images vs both — see `MessageContent.blocks`). **Do not send:** message body text, image URLs, recipient `displayName`, recipient phone number.
  - **Status:** `present`
  - **Severity:** `nice-to-have`

- **`product_create_completed`** — `NewProductRequestDTO` (`src/lib/types/product/NewProductRequestDTO.ts:6-17`) carries `name`, `description`, `price`, `currency`, three category levels, filters, image keys and data. **Safe to send:** product `id` (post-success), top/sub/final category ids, price, currency code, image count. **Do not send:** product `name` (free text), product `description` (user free text — and per the brief, may legitimately contain a published phone number, which is user-published but still PII under GA4's blanket rule).
  - **Status:** `present`
  - **Severity:** `nice-to-have`

### 6. Performance, SSR, and debug observability

- **GA4 script tag injection point.** `app/layout.tsx:38-50` is the root server layout; it renders `<html lang>...<body>` and is the natural home for the GA4 `<script>` via `next/script`. The body already mounts `FirebaseWorkerInit`, `ConfigProvider`, and `ThemeProvider`; the GA4 `<Script>` would sit alongside as a sibling.
  - **File:** `app/layout.tsx:32-51`
  - **Status:** `partial` (the host exists; nothing renders a script today)
  - **Severity:** `nice-to-have`

- **No precedent for `next/script` strategy choice.** Repo has zero `next/script` usages. The closest equivalents are:
  - `next/font/google` (Inter at `app/layout.tsx:14-18`) — uses Next.js's font system, not `<script>`.
  - Firebase JS SDK — imported as a module via `firebase` npm package (no script tag).
  - `react-google-recaptcha` (`ReCaptchaWrapper.tsx:5,30`) — the library injects the reCAPTCHA script on its own when the component mounts; no `<script>` in the repo.
  - Firebase messaging service worker — `FirebaseWorkerInit.tsx:11` calls `navigator.serviceWorker.register('/firebase-messaging-sw.js')` on mount.
  - **Status:** `absent` (precedent for `next/script`)
  - **Severity:** `friction` — strategy choice (`afterInteractive` recommended; GA4 doesn't need to block hydration) has no template in the repo to follow.

- **Heavy client initialization tree under the locale layout.** `AppInit.tsx:15-34` mounts `ChatsInit`, `PushInitializer`, `ChatsWatcher`, `InitFavoritesStore`, `UseTokenRefresh`, `AccountStateDialogsInit`, `SyncCardSizeFromCookie`, `ScrollToTop`. `app/[locale]/layout.tsx:38-43` additionally mounts `NavigationProgressBar`, `AuthInit`, `QuickRecommendButton`, `BaseSiteInit`. Adding the GA4 listener client component here is consistent with the pattern.
  - **Files:** `src/components/client/initializers/AppInit.tsx:15-34`; `app/[locale]/layout.tsx:38-43`
  - **Status:** `present`
  - **Severity:** `nice-to-have`

- **No debug-mode flag anywhere in the repo today.** Grep for environment-driven debug branches in `src/`: zero hits beyond `process.env.ANALYZE === 'true'` (next.config.ts) and `process.env.NEXT_PUBLIC_WATERMARK_ENABLED === 'true'` (`src/lib/images/variants.ts:14`). There is no shared `if (DEBUG)` pattern, no `useDebug()` hook, no logger abstraction.
  - **Files:** `next.config.ts:6`; `src/lib/images/variants.ts:14`
  - **Status:** `absent`
  - **Severity:** `friction` — the natural wiring is a `NEXT_PUBLIC_GA4_DEBUG` env var consumed at the GA4 init site. No existing pattern to mirror; greenfield.

- **No logger / observability abstraction.** `src/lib/utils/serviceLog.ts` exists (used by `authService.ts` for service-error logging) but is a thin wrapper around `console.error`. No Sentry, no DataDog, no LogRocket.
  - **File:** `src/lib/utils/serviceLog.ts`
  - **Status:** `absent` (as a coordinating layer for GA4 `exception` events)
  - **Severity:** `friction` — `exception` instrumentation either (a) wires into every `console.error` call site, or (b) introduces a small `trackError(err, context)` helper and migrates the existing `console.error` calls to it. (b) is the conventions Part 4a-conscious choice.

### 7. Environment separation

- **Stage and prod are separate Vercel projects with separate env-var scopes.** `.github/workflows/deploy-stage.yml:35-44` and `deploy-prod.yml:33-42` use distinct `VERCEL_PROJECT_ID_STAGE` vs `VERCEL_PROJECT_ID` secrets and distinct `CF_KV_NAMESPACE_ID_STAGE` vs `CF_KV_NAMESPACE_ID`. Source code does **not** read `process.env.NODE_ENV` or `process.env.VERCEL_ENV` anywhere (grep across `src/` and `app/`: zero hits). Environment differentiation is fully delegated to the env-var values Vercel injects per scope (Production / Preview / Development).
  - **Files:** `.github/workflows/deploy-stage.yml`, `.github/workflows/deploy-prod.yml`; runtime env consumers `src/lib/config/firebaseClient.ts:16-22`, `src/lib/config/api.ts:7`, `src/lib/config/fetchApi.ts:3`, `src/lib/images/variants.ts:13`, `src/components/recaptcha/ReCaptchaWrapper.tsx:7`.
  - **Status:** `present`
  - **Severity:** `nice-to-have` — there is a strong, consistent precedent. A new `NEXT_PUBLIC_GA4_MEASUREMENT_ID` env var, set to the stage measurement ID on the stage Vercel project and the prod measurement ID on the prod Vercel project, fits the existing pattern exactly.

- **Existing env-aware config values follow `NEXT_PUBLIC_*` naming.** The `.env.local.example` documents seven NEXT_PUBLIC_* values (Firebase config × 7, plus API URL, CDN URL, watermark flag, reCAPTCHA site key) and one server-only secret (`REVALIDATE_SECRET`). All public-bundled values use the `NEXT_PUBLIC_` prefix per Next.js convention.
  - **File:** `.env.local.example:1-63`
  - **Status:** `present`
  - **Severity:** `nice-to-have`

- **Vercel's three env scopes are Production, Preview, Development.** The workflows use `vercel pull --yes --environment=production` for both stage and prod, but the project IDs differ. This means stage's "production" Vercel scope holds the stage measurement ID; prod's "production" scope holds the prod measurement ID. Preview deploys (PR builds against the prod project) would inherit prod's measurement ID unless explicitly scoped — a configuration nuance worth flagging.
  - **Files:** `deploy-stage.yml:108-118`, `deploy-prod.yml:101-111`
  - **Status:** `present`
  - **Severity:** `friction` — preview deploys could pollute prod GA4 data if the env var isn't scoped correctly. Configuration concern, not a code change.

### 8. Cross-repo seams

These are the assumptions GA4 instrumentation makes about backend / Firestore behavior. No backend changes are expected per the brief; this documents the dependencies so the feature spec can capture them.

- **`sign_up` / `login`** — depend on `POST /auth/firebase-sync` returning an `AuthUserDTO` with non-null `id`, `firebaseUid`, and `disabled === false`. The single listener at `UseTokenRefresh.tsx:38-42` is the integration point. **Seam:** `authService.ts:145-167` (`syncUserToBackend`) handles `disabled === true`, `EMAIL_BANNED`, `USER_BANNED` by signing out and not setting `user` — instrumentation must not fire `login` / `sign_up` in those paths.

- **`product_create_completed`** — depends on `createNewProduct(...)` returning a discriminated union `{ type: 'success', data: { id, name } } | { type: 'validation', errors } | { type: ... }` (`UploadedProductDialog.tsx:52-72`). The shape is frozen by `product-validation` feature work and is stable.

- **`contact_seller_clicked`** — pure web. No backend roundtrip on click. Note: `CallUserButton.tsx:53` does call `getUserPhoneNumber(userId)` lazily on the first call attempt — that's a backend roundtrip, but the "click" semantics fire before the response.

- **`message_sent`** — Firestore write success at `useChatStore.ts:505` (new chat batch commit) or `:520` (existing chat addDoc). **Seam:** Firestore Security Rules (`oglasino-firestore-rules` repo, sixth agent) must continue to permit these writes; the instrumentation depends on rule-allowed paths firing the `.then(...)` continuation. If rules tighten in the queued Firestore rules feature (per the QA-Preparation session-12 trust-boundary content for `start-message-flow`), the firing semantics don't change but the write may start rejecting in new edge cases.

- **`product_view`** — server component renders `productDetails` fetched from backend (`getPortalProductDetails(productId)`). The firing client component would consume the same product props. **Seam:** the SSR re-renders on every navigation to `/product/<id>/<name>`; a client mount-effect fires once per mount per product.

- **`search` / `view_search_results`** — pure URL-driven; no backend dependency for the firing decision. The `searchText` query param is the discriminator.

- **`exception`** — purely client. No seam.

- **`form_submit_failed`** — error codes returned by backend conform to the conventions Part 7 wire shape (`{field, code, translationKey}` for product-validation; `{field, code}` for older features). The instrumentation can use `code` as a GA4 parameter directly.

- **`select_promotion`** — **no seam exists yet** because no promotion surface exists. Backend has no "is this product promoted" flag on the product overview/details DTOs as far as the audit could see. The feature spec needs to define promotion semantics first.

---

## Files touched

- none (read-only)

## Tests

- n/a (read-only)

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed (read-only audit, no edits)
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): one observation flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 11 (trust boundaries) — N/A this session (no DTO contract changes audited beyond identity inventory); Part 7 (error contract) — referenced as the wire-shape for `form_submit_failed` payload, no changes

## Known gaps / TODOs

- The brief's parenthetical "this is the first analytics audit" conflicts with the on-disk `.agent/2026-05-19-oglasino-web-ga4-discovery-1.md`. I respected the explicit instruction not to read prior analytics summaries, but the discrepancy is flagged below for Mastermind.
- Messages-page entry buttons for `contact_seller_clicked` (per the QA-Preparation work in `messages-page` and `start-message-flow` topics) were not exhaustively enumerated in this pass. The product-page surfaces are inventoried; the messages-page surfaces deserve a focused pass if Mastermind wants the full inventory.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit; no code changes)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Page-view-on-filter-change question (per the brief).** Filter mutations on the catalog and home pages produce URL changes via `router.replace(newUrl, { scroll: false })` + `router.refresh()` at `FilterManager.tsx:316-317`. A listener on `[pathname, searchParams]` will see every filter tweak (price slider, currency switch, region toggle, sort order) as a navigation. Two paths:
  1. Treat every URL change as a `page_view` — simple, but inflates session event count and skews engagement metrics.
  2. Distinguish "real navigation" (pathname change) from "filter mutation" (same pathname, only searchParams differ) and fire `page_view` only on the former, with a separate `filter_change` event (not in the v1 set) for the latter.
  The audit doesn't decide. Mastermind input requested before the feature spec.

- **`select_promotion` has no host surface.** The 12-event set includes `select_promotion`, but the codebase has no "promoted product" / "popular" / "featured" surface that distinguishes a promoted card from organic results. The Free Zone page is informational (CTA → `JoinFreeZoneButton`, not a product card). The "extra products" carousel (`ExtraProductsComponent.tsx`) is a generic related-products surface, not promotion-distinguished. Either (a) the v1 event set drops `select_promotion` until a promotion surface exists, or (b) the feature spec defines what counts as a promotion and what surfaces render one. Mastermind decision needed.

- **`sign_up` vs `login` discriminator is consumed too early.** `nextRegisterDisplayName` (`authService.ts:28,141-142,187`) is the only signal that distinguishes a register from a login, but it's read and nulled inside `syncUserToBackend` (line 142) before the `UseTokenRefresh` listener sees the result. Three options:
  1. Carry a `wasRegister: boolean` flag through `syncUserToBackend`'s return shape so the listener can branch.
  2. Fire `sign_up` from `registerUserFirebase` directly (line 187) at primitive completion, accepting that `useAuthStore.user` is still null at that moment (no `user_id` parameter available — fire it without `user_id`, or wait for the listener to populate then queue-and-flush).
  3. Use a one-shot cell parallel to `nextRegisterDisplayName` that survives the sync and is consumed by the listener post-`setUser`.
  Mastermind input on which fits the listener-is-sole-hydrator contract best.

- **Cross-repo seams summary** (per scope 8): no backend changes expected, but the GA4 feature spec needs to capture the assumptions:
  - `sign_up` / `login` depend on the `/auth/firebase-sync` response shape (`disabled`, `id`, `firebaseUid`).
  - `product_create_completed` depends on the `createNewProduct` discriminated-union response shape (frozen by product-validation).
  - `message_sent` depends on Firestore Rules permitting the chat-write paths (rules feature is queued in `oglasino-firestore-rules`).
  - `select_promotion` depends on a backend "is promoted" signal that does not exist yet.

- **Audit-vs-disk discrepancy on the "first analytics audit" claim.** The brief states "there are none yet — this is the first analytics audit," but `.agent/2026-05-19-oglasino-web-ga4-discovery-1.md` exists. I did not open the file per the brief's "do not read prior session summaries about analytics" instruction. The filename increment to `<n>=2` follows conventions Part 5. If the 2026-05-19 file is a stale draft, a different feature's misnamed file, or a precursor pass whose findings Mastermind intentionally wants superseded, that decision belongs upstream. Flagging so the existence of that file is not lost.

- **Part 4b adjacent observation: route-error-boundary's `console.error` is identical to global-error's.** `app/error.tsx:7` and `app/global-error.tsx:7` both log `'Global error boundary caught:'` / `'Route error boundary caught:'` with the same shape. When `exception` instrumentation lands, both `useEffect`s should call the same `trackError(err, context)` helper rather than `console.error` directly, to keep the seam in one place. Severity: low. I did not fix this because it is out of scope.

- **No drafted config-file text this session.** No changes to `conventions.md`, `decisions.md`, `state.md`, or `issues.md` required. If Mastermind decides to spec the GA4 feature, a `decisions.md` entry (GA4-only / Consent Mode v2 / separate IDs per env / v1 event set) and a `state.md` Active Feature row are the likely Docs/QA edits.
