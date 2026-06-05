# Google Analytics v1

**Status:** web `shipped`; mobile `mobile-stable` (see `state.md`)
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
| Web      | `shipped`        | Shipped to dev; verified via DebugView 2026-05-23.            |
| Mobile   | `mobile-stable`  | Shipped on `new-expo-dev` via `@react-native-firebase/analytics`; all 13 events verified on-device via Firebase DebugView on iOS + Android 2026-06-02. See Platform adoption — Mobile (Expo) below. |

---

## Platform adoption — Mobile (Expo)

**Status:** `mobile-stable` — shipped on `new-expo-dev`; all 13 events verified on-device via Firebase DebugView on iOS + Android 2026-06-02 (with `page_view` confirmed firing and no parallel auto `screen_view`). See [decisions.md](../decisions.md) 2026-06-02.

**Directive:** strict mirror of the web catalog in this spec. Same 13 events, same event names, same parameter keys, same PII rules. Mobile changes the plumbing, never the catalog. No mobile-only events in v1.

### What mirrors web exactly (do not redesign)

The 13-event catalog and every parameter spec above are authoritative for mobile. Mobile sends the **exact same event names and parameter keys** (`product_id`, `seller_id`, `user_id`, `is_owner_view`, `is_new_chat`, `block_count`, `has_text`, `has_images`, `image_count`, `results_count`, `filters_active`, `sort_order`, `search_term`, `error_code`, `form_name`, `page_path`, `error_name`, `error_message`, `boundary`, `method`) — no camelCase drift, no renames — so both platforms land in the same GA4 custom dimensions. Same `search_term` 100-char truncation. Same `exception` no-stack-trace rule. Same PII discipline: numeric IDs only; never name, email, phone, message body, or image URL/keys in a payload.

### GA4 routing

Mobile reports into the **same GA4 property as web, via a new mobile data stream** (not a separate property). One app+web property keeps the mirrored events in unified reports. Igor executes the admin-side stream creation.

### Plumbing that differs from web (mobile-specific)

- **SDK.** No `gtag.js` / `next/script`. Use `@react-native-firebase/analytics` + `@react-native-firebase/app` (new dependencies; require a native rebuild). Every event goes through one mobile `track(event, params)` wrapper over `logEvent` — no raw `logEvent` at call sites, mirroring web's single-entry-point discipline. The web guards collapse to mobile equivalents: not-initialized, consent-not-granted, SDK-absent.
- **`page_view`, not `screen_view`.** Mobile fires a custom `page_view` (web's event name, for report parity) on route change via an expo-router `usePathname()` listener mounted **inside the navigation context** (a `_layout` under the portal `<Stack>`, not `AppInit` — `usePathname()` does not resolve outside the navigator). Payload `{ page_path }`. **Disable Firebase Analytics' automatic `screen_view` collection** at init so views are not double-counted.
- **ATT (iOS).** Add `expo-tracking-transparency`; show the App Tracking Transparency prompt at launch **before** analytics SDK init. Apple rejects analytics SDKs without it.
- **Consent + ATT gating.** Analytics init gates on `(ATT-allows-on-iOS, or Android) AND isAnalyticsConsentGranted()`. The gate (`src/lib/consent/analyticsGate.ts`) is already shipped (consent-mode-mobile). F is its first live caller. Subscribe to the consent store so toggling analytics off stops collection without an app restart.
- **`user_id`.** Set via the Firebase Analytics `setUserId` equivalent once auth resolves; clear on sign-out. Mirrors web's `GA4UserIdSync`.
- **Delete `src/lib/client/firebaseAnalytics.ts`.** Dead web-SDK vestige (`typeof window` guard, always `null` on device), zero importers. The F implementation deletes it.

### Firing surfaces and the decisions that shaped them

- **`page_view`** — expo-router `usePathname()` listener inside the navigator. Fires after boot (Stack mounts only at `bootStatus` `ready`/`updating`), so a missing pre-boot `user_id` is expected and allowed.
- **`sign_up` / `login`** — fire from the **explicit auth methods** `login()` / `register()` / `loginWithGoogle()` in `src/lib/store/authStore.ts`. The `initAuthListener` `onIdTokenChanged` branch fires **nothing** — it is a token-rotation/cold-start-rehydration handler, and firing there would emit a spurious `login` on every app relaunch. `sign_up` vs `login` is discriminated by `wasRegister` (see below). `method` ∈ `email | google | facebook` is read from `firebaseUser.providerData[0].providerId`. Facebook is a dead stub today, so only `email` and `google` paths fire — correct; instrument what exists.
- **`wasRegister` plumbing (required, mobile-side only, zero backend work).** The backend already returns `wasRegister: boolean` on the shared `/auth/firebase-sync` response, but the mobile `AuthUserDTO` type omits it and `syncUserToBackend` drops it. F adds `wasRegister: boolean` to the mobile `AuthUserDTO` type and surfaces it through `syncUserToBackend` to the explicit auth methods, which use it to pick `sign_up` vs `login`.
- **`product_view`** — product detail screen (`app/(portal)/(public)/product/[...productData].tsx`), effect after the product + owner load. `is_owner_view` = `owner.iamActive`. Coerce `price` (typed `string` on the DTO) to a number; pass numeric `product_id` + category IDs, never the product name.
- **`product_create_started`** — `AddUpdateProductDialog` first-step mount, **gated to the create instance only** (the dialog hard-codes `mode: 'create'` at `UploadedProductDialog.tsx`; F confirms create-only so it does not fire on edit).
- **`product_create_completed`** — `UploadedProductDialog` success branch. Coerce `price` (string → number); read currency as the ISO code off the `CurrencyDTO` object (`.code`), not the object itself.
- **`contact_seller_clicked`** — three call sites: `CallUserButton` (Call), `StartMessageButton` (Message), `FavoriteButton` (Favorite). `CallUserButton` needs `product_id` plumbed one hop from its parent `ProductFunctions.tsx` (same plumbing web needed); it carries `seller_id` (`userId`) but no product context. Never put the seller phone/name in the payload — `seller_id` numeric only.
- **`message_sent`** — `useActiveChatStore.sendMessage`, after `batch.commit()` resolves. `receiver_id` = `receiver.id` (numeric), NOT `receiver.firebaseUid`. Payload carries only `is_new_chat`, `block_count`, `has_text`, `has_images`, `product_id` — never the message body text or image keys.
- **`search`** — `SearchInput` commit (the "search for X" footer press). Tapping an autocomplete suggestion is a product navigation, not a search — does not fire `search`.
- **`view_search_results`** — `FilteredProductList` on page-0 resolution when `searchText` is non-empty. `results_count` = `totalNumberOfProducts` from the response.
- **`filter_change`** — a **debounced effect on `FilteredProductList.filtersData`, excluding `searchText`** (which belongs to `search`). Mobile keeps filters in Zustand with no URL-sync, so this memo is the single convergence point every mutation flows through. `filters_active` carries filter categories only, never values.
- **`exception`** — neither an error boundary nor a global handler exists on mobile today; both are built by F (RN-specific — web's `error.tsx`/`global-error.tsx`/`window` listeners do not port): an expo-router `ErrorBoundary` export (`boundary: 'route'`) and `ErrorUtils.setGlobalHandler` + promise-rejection tracking (`boundary: 'global'`). Same payload as web (`error_name`, `error_message` truncated to 200, no stack trace).
- **`form_submit_failed`** — two error shapes on mobile. Backend-validated forms (product create/update) expose the structured `{field, code, translationKey}` shape via `parseServiceError`; fire with the real Part-7 `code`. Register/login use imperative `validateForm` branches (no Zod on mobile), so F defines a **synthetic** `error_code` set from those branches (e.g. `EMAIL_EMPTY`, `EMAIL_FORMAT`, `PASSWORD_EMPTY`, `PASSWORD_TOO_SHORT`, `DISPLAY_NAME_EMPTY` — F enumerates the exact branches at build time).

### Definition of done (mobile)

- All 13 events fire from the surfaces above with web-identical names and parameter keys.
- ATT prompt shown at launch; SDK init gated on ATT + `isAnalyticsConsentGranted()`.
- `user_id` set on auth resolve, cleared on sign-out.
- `wasRegister` threaded; `sign_up` vs `login` discriminates correctly across email and google.
- Auto-`screen_view` disabled; `page_view` fires on route change.
- `firebaseAnalytics.ts` deleted; `CallUserButton` carries `product_id`.
- New mobile GA4 data stream in the web property; Igor verifies each event in Firebase DebugView on a real device.

---

## Session log

### Mobile (Expo) — `new-expo-dev`

- **2026-06-01 — `oglasino-expo-google-analytics-v1-1`** — read-only audit (produced `audit-google-analytics-v1.md`): current-state mapping of firing surfaces and the mobile SDK story.
- **2026-06-01 — `oglasino-expo-google-analytics-v1-2`** — foundation + auth/page_view: `track`/`trackError` wrapper over `logEvent` with the three no-op guards, consent+ATT-gated init, `user_id` on auth resolve / cleared on sign-out, `page_view`, `sign_up`, `login`, `wasRegister` via a type-only `AuthUserDTO` change; deleted the dead `firebaseAnalytics.ts`.
- **2026-06-01 — `oglasino-expo-google-analytics-v1-3`** — `exception` (RN-native route `ErrorBoundary` + `ErrorUtils` global handler + promise-rejection tracking) and `form_submit_failed` (real Part-7 codes on backend paths; synthetic codes on client-validated paths).
- **2026-06-02 — `oglasino-expo-google-analytics-v1-4`** — the eight transactional events (`product_view`, `product_create_started`, `product_create_completed`, `contact_seller_clicked`, `message_sent`, `search`, `view_search_results`, `filter_change`). Corrects the gap a premature closure had assumed shipped; 395/395 tests green.
- **2026-06-02 — `oglasino-expo-ios-firebase-nonmodular-fix-1`** — native toolchain fix unblocking the iOS build (`buildReactNativeFromSource: true`); not a feature event, but required to ship the SDK. See [decisions.md](../decisions.md) 2026-06-02.
- **2026-06-02** — on-device Firebase DebugView smoke passed on iOS + Android; feature flipped to `mobile-stable`.
