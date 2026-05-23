# Google Analytics v1

**Status:** `in-progress-web`
**Branch:** `dev`
**Mastermind chat:** 2026-05-20
**Spec authority:** this document. Any sibling-repo doc that disagrees defers to this file per conventions Part 1.

**Depends on:** [`consent-mode-v2.md`](consent-mode-v2.md) shipped first. GA4 v1 cannot fire its first event until the consent foundation is in place.

---

## Summary

Add Google Analytics 4 instrumentation to `oglasino-web`. Client-side via `gtag.js`, loaded behind Consent Mode v2. Twelve events plus automatic `page_view`. Separate measurement IDs for stage and prod via `NEXT_PUBLIC_GA4_MEASUREMENT_ID`. Web only — mobile gets its own conversation in a future Mastermind chat. After the feature ships in code, Igor sets up the GA4 admin side using the runbook in this spec.

---

## Scope

In scope:

- `gtag.js` script load via `next/script` with `strategy="afterInteractive"`
- `gtag('config', ...)` initialization gated on consent and on `useAuthResolved`
- Twelve events fired from web (full event list below)
- Automatic `page_view` firing on pathname change (not on filter change)
- A `track(event, params)` helper that consumes the `gtag` global
- A `trackError(err, context)` helper wired into both error boundaries plus a global `window.onerror` / `unhandledrejection` listener
- A `filter_change` event in place of `page_view` for filter mutations on catalog/home
- Env-var-driven measurement ID, separately scoped on stage and prod Vercel projects
- A GA4 admin-side setup runbook for Igor
- Sign-up vs login discriminator via `wasRegister` returned from `syncUserToBackend`

Out of scope:

- Server-side GA4 (Measurement Protocol). v1 is client-side only.
- GTM. We use `gtag.js` directly.
- `select_promotion`. Dropped from v1 per D7. Returns when a promotion surface lands.
- Anonymous client-ID join (correlating pre-login activity with post-login `user_id`). `_ga` covers anonymous; v1 doesn't try to bridge.
- Backend changes. All instrumentation lives in web.
- Mobile. Separate Mastermind chat.

---

## Event catalog

Twelve events plus automatic `page_view`. Each event lists its trigger, the file where the firing call lives, and the parameters it carries.

All events carry `user_id` (when `useAuthResolved()` is `true` and `user.id` is present) as a global param set via `gtag('set', { user_id })` on resolution. Events fired before resolution land without `user_id`; this is acceptable for `page_view`, `search`, `view_search_results`, `product_view`, and `filter_change`. Events that semantically require an identified user (`sign_up`, `login`, `product_create_started`, `product_create_completed`, `contact_seller_clicked`, `message_sent`, `form_submit_failed`) always fire after auth resolution, by construction of their firing surfaces.

| # | Event | Trigger | File |
|---|-------|---------|------|
| 1 | `page_view` (automatic via `gtag.js`) | Pathname change | `app/[locale]/layout.tsx` route listener (new client component) |
| 2 | `sign_up` | Backend sync returns after register | `UseTokenRefresh.tsx` post-`setUser` |
| 3 | `login` | Backend sync returns after non-register auth event | `UseTokenRefresh.tsx` post-`setUser` |
| 4 | `product_view` | Product detail page mounts | New `ProductViewTracker` client component mounted from `app/[locale]/(portal)/(public)/product/.../page.tsx` |
| 5 | `product_create_started` | The create wizard's first user-facing step (`currentStep === 0`, `ImageSelectionProductDialog`) mounts with fresh state | `CreateNewProductDialog.tsx` mount effect |
| 6 | `product_create_completed` | `createNewProduct` returns `type: 'success'` | `UploadedProductDialog.tsx:62-65` |
| 7 | `contact_seller_clicked` | Call / Message / Favorite buttons clicked | `CallUserButton.tsx`, `StartMessageButton.tsx`, `FavoriteButton.tsx`. `CallUserButton.tsx` accepts a required `productId` prop plumbed from its single call site (`ProductFunctions.tsx`); the other two already carry product context. |
| 8 | `message_sent` | Firestore commit resolves | `useChatStore.ts:sendMessage` (at `src/messages/store/`), post-`batch.commit()` |
| 9 | `search` | Global header search submitted | `SearchInput.tsx:commitSearch` |
| 10 | `view_search_results` | Catalog page renders with `search_text` param | Client component hooked into catalog page hydration |
| 11 | `filter_change` | Filter mutation on catalog/home | `FilterManager.tsx` URL-sync effect |
| 12 | `exception` | Error boundary fires; `window.onerror`; `unhandledrejection` | `app/error.tsx`, `app/global-error.tsx`, new global listener |
| 13 | `form_submit_failed` | Form validation error surfaces | Each form's existing `setError` / `setProductErrors` site |

### Event parameter specifications

Each event below lists allowed parameters. PII analysis from the audit (scope item 5) is enforced: free-text fields (name, description, displayName, email, phone) never appear.

**`sign_up`**
- `method`: `'email'` | `'google'` | `'facebook'`
- `user_id`: backend `id` (numeric, set via `gtag('set')`)
- No `displayName`, no `email`.

**`login`**
- `method`: same as `sign_up`
- `user_id`: backend `id`

**`product_view`**
- `product_id`: numeric
- `top_category_id`, `sub_category_id`, `final_category_id`: numeric (if available)
- `price`: numeric
- `currency`: ISO code
- `is_owner_view`: boolean (so engagement metrics can be filtered to non-owner traffic)

**`product_create_started`**
- No parameters beyond the global `user_id`. Marks the funnel entry.

**`product_create_completed`**
- `product_id`: returned by the success response
- `top_category_id`, `sub_category_id`, `final_category_id`: numeric
- `price`: numeric
- `currency`: ISO code
- `image_count`: numeric

**`contact_seller_clicked`**
- `method`: `'call'` | `'message'` | `'favorite'`
- `product_id`: numeric
- `seller_id`: numeric (backend `id` only)
- No seller name, no phone number.

**`message_sent`**
- `is_new_chat`: boolean
- `receiver_id`: numeric (backend `id`)
- `product_id`: numeric (if message has product context)
- `block_count`: numeric (count of `MessageContent.blocks`, not bodies)
- `has_text`: boolean
- `has_images`: boolean
- No body text, no image URLs, no receiver name.

**`search`**
- `search_term`: the raw search term. **Allowed.** GA4 reports treat `search_term` specifically; the value isn't free-text PII in any meaningful sense (it's what the user typed into a public search box). However: any search term over 100 characters gets truncated to 100 chars to protect against accidental paste of PII-bearing strings.

**`view_search_results`**
- `search_term`: same handling as `search`
- `results_count`: numeric, taken from the rendered page's `totalNumberOfProducts`

**`filter_change`**
- `filters_active`: comma-joined list of which filter categories are set (e.g., `'price,currency,region'`). Categories only, not values, to avoid high-cardinality.
- `sort_order`: the active sort (e.g., `'newest'`, `'price_asc'`)
- `pathname`: the current pathname (e.g., `/catalog/cars`)

**`exception`**
- `error_name`: error constructor name
- `error_message`: error message, truncated to 200 chars
- `boundary`: `'global'` | `'route'` | `'window_onerror'` | `'unhandled_rejection'`
- No stack trace (too large and may contain PII fragments from URL params).

**`form_submit_failed`**
- `form_name`: e.g., `'register'`, `'login'`, `'product_create_step_1'`, `'product_create_step_4'`, `'product_update'`, `'profile_update'`
- `error_code`: source depends on the form's validation path. Forms with backend validation (product create, product update) fire with the conventions Part 7 code (e.g., `'NAME_BANNED_WORDS'`, `'EMAIL_BANNED'`). This requires preserving the `code` field alongside `translationKey` through `parseProductValidationErrors` in `src/lib/utils/parseProductValidationErrors.ts` — small refactor folded into the form-failure brief. Forms with only client-side Zod validation (register, login) fire with a synthetic code derived from the Zod failure path; the engineer authoring the brief defines the synthetic-code set (e.g., `'EMAIL_FORMAT'`, `'PASSWORD_TOO_SHORT'`, `'DISPLAY_NAME_TOO_SHORT'`). Synthetic codes are documented inline in the relevant form component.
- `field`: the field name from the wire shape, or `null` for object-level violations

---

## Architecture

### The `track` helper

Single entry point for all events:

```ts
// src/lib/analytics/track.ts
export function track(event: string, params: Record<string, unknown> = {}) {
  if (typeof window === 'undefined') return;
  if (!window.__og_consent_loaded) return;
  if (!window.gtag) return;
  window.gtag('event', event, params);
}
```

Three guards:

1. `window === undefined`: server-side render, no-op.
2. `__og_consent_loaded`: the SSR consent snippet didn't run, treat as no-op rather than firing without consent state.
3. `window.gtag` absent: GA4 hasn't loaded yet (consent denied analytics_storage, or script load failed). No-op silently.

Every event call site uses `track(...)`. No direct `window.gtag(...)` calls outside `track`.

### The `trackError` helper

```ts
// src/lib/analytics/trackError.ts
export function trackError(
  err: unknown,
  context: { boundary: 'global' | 'route' | 'window_onerror' | 'unhandled_rejection' }
) {
  const error = err instanceof Error ? err : new Error(String(err));
  track('exception', {
    error_name: error.name,
    error_message: error.message.slice(0, 200),
    boundary: context.boundary,
  });
}
```

Both error boundaries (`app/error.tsx`, `app/global-error.tsx`) replace their `console.error` with `trackError(error, { boundary: 'route' | 'global' })`. A new initializer mounted from `AppInit.tsx` registers `window.addEventListener('error', ...)` and `window.addEventListener('unhandledrejection', ...)` as the catch-all for what the boundaries miss.

### Script load

In `app/layout.tsx`, after the SSR consent snippet, add:

```tsx
import Script from 'next/script';

const MEASUREMENT_ID = process.env.NEXT_PUBLIC_GA4_MEASUREMENT_ID;

{MEASUREMENT_ID && (
  <>
    <Script
      src={`https://www.googletagmanager.com/gtag/js?id=${MEASUREMENT_ID}`}
      strategy="afterInteractive"
    />
    <Script id="ga4-init" strategy="afterInteractive">
      {`
        gtag('js', new Date());
        gtag('config', '${MEASUREMENT_ID}', { send_page_view: false });
      `}
    </Script>
  </>
)}
```

Two notes:

- `send_page_view: false` disables the automatic `page_view` that fires on `gtag('config', ...)`. We fire `page_view` ourselves on pathname change so locale-aware client-routing is captured correctly.
- The `MEASUREMENT_ID &&` guard means: in environments where the env var isn't set (local dev by default, preview builds without the secret), GA4 doesn't load at all. The `track` helper's `window.gtag` guard then no-ops every event.
- `next/script` is the standard Next.js script-loading primitive; this feature introduces its first use in `oglasino-web`. No prior `<Script>` usage existed in the codebase as of 2026-05-21.

### Page-view firing

A new client component `GA4RouteListener` mounts at `app/[locale]/layout.tsx`, sibling to `NavigationProgressBar`. It uses `usePathname()` and fires `page_view` on pathname change only. `useSearchParams()` is read but not subscribed — filter changes don't trigger this listener.

```tsx
'use client';
import { usePathname } from 'next/navigation';
import { useEffect, useRef } from 'react';
import { track } from '@/lib/analytics/track';

export function GA4RouteListener() {
  const pathname = usePathname();
  const lastPathname = useRef<string | null>(null);

  useEffect(() => {
    if (lastPathname.current === pathname) return;
    lastPathname.current = pathname;
    track('page_view', { page_path: pathname });
  }, [pathname]);

  return null;
}
```

Filter changes are handled separately in `FilterManager.tsx`'s URL-sync effect: on a `router.replace(newUrl, ...)` call that changes only `searchParams` (same pathname), fire `track('filter_change', { ... })`. On a true navigation (pathname change), the `GA4RouteListener` handles it.

### `user_id` resolution

`useAuthResolved` from `src/lib/hooks/useAuthResolved.ts` is the gate. A new initializer mounted from `AppInit.tsx` watches the auth store and calls `gtag('set', { user_id })` once `useAuthResolved()` becomes `true` and `user.id` is set:

```tsx
'use client';
import { useEffect } from 'react';
import { useAuthStore } from '@/lib/store/useAuthStore';
import { useAuthResolved } from '@/lib/hooks/useAuthResolved';

export function GA4UserIdSync() {
  const resolved = useAuthResolved();
  const user = useAuthStore((s) => s.user);

  useEffect(() => {
    if (!resolved || !window.gtag) return;
    if (user?.id) {
      window.gtag('set', { user_id: String(user.id) });
    } else {
      // signed out — clear it
      window.gtag('set', { user_id: undefined });
    }
  }, [resolved, user?.id]);

  return null;
}
```

`user_id` is the backend numeric `id` (stable, non-PII), stringified per GA4's expected type.

### Sign-up vs login discriminator

Per D6, `syncUserToBackend` returns `{ user, wasRegister }`. The `wasRegister` value is authoritative from the backend — the `/auth/firebase-sync` response carries a new `wasRegister: boolean` field set to `true` when the backend created a new `users` row, `false` when an existing row was matched. This captures registration on every auth method (email+password, Google, Facebook) rather than only the email+password flow that previously set `nextRegisterDisplayName`.

```ts
// authService.ts (changed)
export async function syncUserToBackend(firebaseUser: FirebaseUser): Promise<{
  user: AuthUserDTO | null;
  wasRegister: boolean;
}> {
  // ... existing sync logic ...
  // Response now includes wasRegister; pass it through.
  return { user, wasRegister: response.wasRegister };
}
```

```ts
// UseTokenRefresh.tsx (changed)
const { user: backendUser, wasRegister } = await syncUserToBackend(firebaseUser);
useAuthStore.getState().setUser(backendUser);

if (backendUser) {
  const providerId = firebaseUser.providerData[0]?.providerId ?? 'password';
  const method =
    providerId === 'google.com' ? 'google' :
    providerId === 'facebook.com' ? 'facebook' :
    'email';

  if (wasRegister) {
    track('sign_up', { method, user_id: String(backendUser.id) });
  } else {
    track('login', { method, user_id: String(backendUser.id) });
  }
}
```

Provider mapping: `password` → `email`, `google.com` → `google`, `facebook.com` → `facebook`. Defensive `?? 'password'` fallback handles the (n/a in practice) empty-`providerData` case.

The `disabled === true` / `EMAIL_BANNED` / `USER_BANNED` paths continue to return `user: null`. Events fire only when `backendUser` is non-null, so banned-sign-in attempts produce no `login` or `sign_up` event.

The existing `nextRegisterDisplayName` module-scoped cell in `authService.ts` continues to serve its display-name role (set by `registerUserFirebase`, consumed by `syncUserToBackend` for the `displayName` field on the `firebase-sync` POST). It is no longer consulted as the sign-up discriminator. A follow-up cleanup could remove it entirely once display-name propagation has another path; not in scope for this feature.

---

## Environment configuration

### Env var

`NEXT_PUBLIC_GA4_MEASUREMENT_ID`

Set per Vercel project:
- Stage project: `G-STAGE_ID_HERE`
- Prod project: `G-PROD_ID_HERE`
- Local dev: unset by default; developer can set it in `.env.local` if they want to test against the stage property.
- Preview deploys against the prod project: leave unset, to avoid prod-data pollution. Vercel env-var scoping (Production vs Preview vs Development) supports this.

Add to `.env.local.example`:

```
# Google Analytics 4 measurement ID (set on stage and prod Vercel projects only)
NEXT_PUBLIC_GA4_MEASUREMENT_ID=
```

### Debug mode

GA4 has a Debug View in its admin UI that shows events as they arrive in real time, useful for QA. Activating it requires `debug_mode: true` on every event payload.

Add a second env var:

`NEXT_PUBLIC_GA4_DEBUG_MODE=true`

When set, the `track` helper injects `debug_mode: true` into every event params. Default is unset (no debug mode in production). Engineers verifying instrumentation locally or on stage set this flag temporarily.

---

## Implementation order — Phase 5 brief sequence

1. **Foundation**: `track` and `trackError` helpers. `GA4RouteListener` component (no-op stub). `GA4UserIdSync` component (no-op stub). `.env.local.example` updated.
2. **Script load**: `app/layout.tsx` adds the `<Script>` blocks gated on `NEXT_PUBLIC_GA4_MEASUREMENT_ID`. `GA4RouteListener` mounted from `app/[locale]/layout.tsx`. `GA4UserIdSync` mounted from `AppInit.tsx`. Pathname-only `page_view` firing wired up.
3. **Backend `wasRegister` flag**: `oglasino-backend` adds `wasRegister: boolean` to the `firebase-sync` response. Backend brief — separate session per conventions Part 3 (no cross-repo briefs).
4. **Auth events**: `syncUserToBackend` returns `{ user, wasRegister }` consumed from the backend response. `UseTokenRefresh.tsx` fires `sign_up` and `login`.
5. **Product events**: `product_view`, `product_create_started`, `product_create_completed`. New `ProductViewTracker` client component.
6. **Engagement events**: `contact_seller_clicked` (three buttons), `message_sent` (chat store).
7. **Search and filter events**: `search`, `view_search_results`, `filter_change`. `GA4RouteListener` excludes filter changes; `FilterManager.tsx` fires them.
8. **Error events**: `trackError` wired into both boundaries plus global window listeners.
9. **Form-failure events**: each form's error site wired with `track('form_submit_failed', ...)`.
10. **Stage verification**: Igor sets up the stage GA4 property per the runbook below, runs through every event, verifies via Debug View.
11. **Prod cutover**: Igor sets up the prod GA4 property per the runbook, prod env var set on Vercel, deploy.

Briefs 1-3 don't fire any user events; they just lay the wiring. Brief 4 onward fires real events.

---

## GA4 admin-side setup runbook

This runs after the code is shipped. Igor executes; the runbook lives here so it's reproducible.

### Step 1: Create the stage GA4 property

1. Go to https://analytics.google.com/
2. Admin (gear icon, bottom-left) → "Create" → "Property"
3. Property name: "Oglasino Stage"
4. Reporting time zone: "Europe/Belgrade" (or Igor's preference)
5. Currency: EUR (or RSD if preferred)
6. Industry: "Shopping" → "General Merchandise" (closest category)
7. Business size: appropriate to current size
8. Use cases: select "Examine user behavior" and "Measure customer engagement"
9. Save

### Step 2: Create the data stream

1. In the new property → Admin → "Data Streams" → "Add stream" → "Web"
2. Website URL: the stage URL
3. Stream name: "Oglasino Stage Web"
4. Enhanced measurement: leave default ON (it adds scroll, outbound clicks, etc. — useful, no PII concerns)
5. Save
6. Copy the Measurement ID (looks like `G-XXXXXXXXXX`). This goes into the stage Vercel project's `NEXT_PUBLIC_GA4_MEASUREMENT_ID`.

### Step 3: Configure custom dimensions

For each event parameter that isn't a GA4 built-in, register it as a custom dimension. Without registration, the parameter is sent but not queryable in reports.

Admin → "Custom definitions" → "Custom dimensions" → "Create"

Register these (scope: Event):

- `method` (already a GA4 recommended param, may not need explicit registration; verify in UI)
- `product_id`
- `top_category_id`, `sub_category_id`, `final_category_id`
- `seller_id`
- `receiver_id`
- `is_owner_view`
- `is_new_chat`
- `block_count`
- `has_text`
- `has_images`
- `image_count`
- `results_count`
- `filters_active`
- `sort_order`
- `form_name`
- `error_code`
- `error_name`
- `boundary`
- `field`

Scope: User dimensions — none in v1 (all user identification is via the standard `user_id`).

Note: GA4 has a 50-event-scoped-custom-dimension limit per property. This list is comfortably under.

### Step 4: Mark conversion events

Admin → "Events" → wait for at least one event of each type to arrive, then mark these as Conversions:

- `sign_up`
- `product_create_completed`
- `contact_seller_clicked`
- `message_sent`

Conversions are the events that matter to the funnel. The others are engagement data.

### Step 5: Configure Consent Mode

Admin → "Property settings" → "Consent settings" (if available; UI varies). Enable behavioral and conversion modeling — this lets GA4 model the impact of users who declined `analytics_storage`.

### Step 6: Set up Debug View

Admin → "DebugView" should be accessible directly. To send debug events, the engineer sets `NEXT_PUBLIC_GA4_DEBUG_MODE=true` locally and reproduces user flows; the events arrive in DebugView in real time.

### Step 7: Internal traffic filter

Admin → "Data Streams" → click the stream → "Configure tag settings" → "Show all" → "Define internal traffic" → add Igor's IP range. Then Admin → "Data Settings" → "Data filters" → set the internal-traffic filter to "Active" (it starts as "Testing" by default).

### Step 8: Retention settings

Admin → "Data Settings" → "Data Retention" → set "Event data retention" to the longest available (14 months on the free tier). User-data retention follows the same.

### Step 9: Linking (optional, can be deferred)

- Google Search Console: Admin → "Property settings" → "Product links" → "Search Console Links" → Link. Pulls organic search data into GA4.
- Google Ads: not needed in v1 (no ad campaigns).

### Step 10: Repeat for prod

After stage is verified working end-to-end, repeat Steps 1-9 for a separate "Oglasino Prod" property and stream. The prod Measurement ID goes into the prod Vercel project's env var.

**Crucially**: stage and prod are separate **properties**, not separate streams in one property. This guarantees stage traffic never appears in prod reports.

---

## Test plan

### Unit
- `track` helper: guards correctly when `window`, `__og_consent_loaded`, or `gtag` is absent.
- `trackError` helper: extracts `name`, `message`, truncates message to 200 chars.
- Provider mapping: `password` → `email`, `google.com` → `google`, `facebook.com` → `facebook`.
- `syncUserToBackend` returns `{ user, wasRegister }` correctly in register vs login flows.

### Component
- `GA4RouteListener`: fires `page_view` on pathname change, doesn't fire on searchParams-only change.
- `GA4UserIdSync`: calls `gtag('set', { user_id })` after auth resolves; clears on sign-out.
- Each event firing surface: the right event with the right params lands when the user action happens.

### Integration
- Full register flow: `sign_up` fires with `method: email`, then no spurious `login`.
- Full login flow: `login` fires, no `sign_up`.
- Create-product wizard: `product_create_started` on step 1, `product_create_completed` on success.
- Send first message: `message_sent` with `is_new_chat: true`. Send second message in same chat: `is_new_chat: false`.

### Manual (Igor, in DebugView)
- Each of the 12 events visible in DebugView when triggered.
- `user_id` populated correctly for authenticated events.
- No PII (free-text) appearing in any event payload.
- Filter changes do NOT fire `page_view`. They DO fire `filter_change`.
- Consent: revoke `analytics_storage` in the banner, verify `gtag` calls produce no network requests to `g/collect`. Re-grant, verify events resume.

---

## Cross-repo seams

- **Backend.** `firebase-sync` response gains a `wasRegister: boolean` field. Backend sets it to `true` when a new `users` row is created during sync, `false` when matching an existing row. The `error_code` parameter on `form_submit_failed` consumes the existing wire shape from conventions Part 7; no other backend changes.
- **Firestore Rules.** No changes. `message_sent` fires after Firestore write completes; rule changes (queued in `oglasino-firestore-rules`) don't affect the firing decision, only whether the write succeeds in the first place.
- **Router worker.** No changes.
- **Mobile.** Out of scope. Mobile gets its own analytics conversation in a future Mastermind chat.

---

## Open items at draft time

- **Privacy Policy paragraph for GA4.** Pre-launch action items in `state.md` already list lawyer review pending. The Privacy Policy already references analytics processing in general terms; the lawyer review at consent-feature close will determine whether the GA4-specific paragraph needs amendment. Docs/QA notifies legal-drafts chat as part of this feature's closure, same pattern as consent.

---

## Definition of done

- All 10 implementation steps shipped and merged to `dev`.
- Test coverage per Test plan above.
- Stage GA4 property created, verified end-to-end via DebugView for each of the 13 event types.
- Prod GA4 property created, env var set on the prod Vercel project.
- `.env.local.example` updated.
- Internal-traffic filter active on both stage and prod properties (Igor's IP).
- Conversion events marked: `sign_up`, `product_create_completed`, `contact_seller_clicked`, `message_sent`.
- Custom dimensions registered.
- Stale `2026-05-19-oglasino-web-ga4-discovery-1.md` file in `.agent/` archived (Docs/QA, at feature close).
- Docs/QA notifies legal-drafts chat that analytics is live.
- `state.md` updated: GA4 v1 status `shipped`.
- `decisions.md` carries the closing entry summarizing the feature.

---

## Platform adoption

| Platform | Status        | Notes                                                          |
|----------|---------------|----------------------------------------------------------------|
| Web      | `planned`     | This spec.                                                     |
| Mobile   | `not-started` | Separate Mastermind chat. Mobile has its own analytics SDK story (`expo-firebase-analytics`, the `@react-native-firebase/analytics` package, or GA4-direct via measurement protocol) plus App Tracking Transparency on iOS. Not folded in here. |
