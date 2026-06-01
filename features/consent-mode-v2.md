# Consent Mode v2

**Status:** `shipped`
**Branch:** `feature/consent-mode-v2` (off `dev`)
**Mastermind chat:** 2026-05-20
**Spec authority:** this document. Any sibling-repo doc that disagrees defers to this file per conventions Part 1.

---

## Summary

Rebuild the cookie consent infrastructure in `oglasino-web` to support Google Consent Mode v2. The current `{ necessary, preference }` two-category model expands to a four-signal model matching Google's wire shape. A new SSR-aware default-consent snippet runs inline in `<head>` before any third-party script loads. The banner UX gains a Reject-all CTA as a twin primary to Accept-all, with a Customize expander for per-category control. A footer "Manage cookie preferences" link reopens the same Drawer for logged-out users. Consent storage splits into its own dedicated cookie. Card-size and language cookies — currently written regardless of consent — move behind the preference category.

This feature is a prerequisite for `google-analytics-v1.md`. GA4 v1 cannot ship until this feature does.

This feature also has implications for the legal drafts in `oglasino-docs/legal/`. At feature close, Docs/QA notifies the legal-drafts chat so privacy-policy-draft.md can be reviewed for paragraph updates reflecting the new consent UX.

---

## Scope

In scope:

- New `ConsentData` shape carrying all four Consent Mode v2 signals
- New `og_consent` cookie, separate from `globalCookie`
- Inline SSR consent-defaults snippet emitted from `app/layout.tsx`
- Banner UX rebuild: twin Accept-all / Reject-all primaries, Customize expander
- Footer "Manage cookie preferences" link, reopens the same Drawer
- Card-size and language cookies gated behind `preference` category
- Logged-in settings toggle at `/owner/cookies` updated to write back to `og_consent`
- Dead-code cleanup: delete `firebaseAnalytics.ts` and the `PREFERENCE_COOKIE_NAME` alias
- Translation seeds for the new banner copy across EN / RS / RU / CNR

Out of scope:

- Loading `gtag.js` or any analytics SDK — that's GA4 v1's job
- Wiring reCAPTCHA's load to consent — separate concern flagged in Privacy Policy review
- Service worker gating — strictly necessary, stays unconditional
- Mobile (`oglasino-expo`) — separate Mastermind chat
- Admin surface banner mounting — admin users are signed-in operators, consent UX is a portal/owner-dashboard concern
- Backend changes to the `AuthUserDTO.allowPreferenceCookies` mirror shape — see Cross-repo seams

---

## Consent model

### The four signals

Consent Mode v2 expects four signals. Modeled in full now, even though ad-* signals are permanently denied per the no-marketing-pixels commitment in the Privacy Policy.

```ts
export type ConsentSignal = 'granted' | 'denied';

export type ConsentData = {
  // Strictly necessary — implicit, always granted, not user-toggleable
  necessary: 'granted';

  // User-toggleable
  analytics_storage: ConsentSignal;
  preference: ConsentSignal;

  // Permanently denied per Privacy Policy
  ad_storage: 'denied';
  ad_user_data: 'denied';
  ad_personalization: 'denied';

  // Metadata
  version: 1;
  decidedAt: number; // unix seconds
};
```

`necessary` and the three `ad_*` signals are typed as literal values so TypeScript catches any attempt to set them otherwise.

`version: 1` future-proofs the cookie. If the shape ever changes, a future reader can use the version field to adapt.

### Default state for first-time visitors

Before any decision is made, defaults are `denied` for everything except `necessary`. This is what the SSR snippet emits when `og_consent` is absent:

```js
gtag('consent', 'default', {
  ad_storage: 'denied',
  ad_user_data: 'denied',
  ad_personalization: 'denied',
  analytics_storage: 'denied',
  wait_for_update: 500,
});
```

`wait_for_update: 500` tells Google's tooling to wait up to 500ms for an `update` call before sending the first ping, giving the client banner time to dispatch its decision if the user clicks Accept/Reject quickly.

### After a decision

Banner Accept-all → `analytics_storage: granted`, `preference: granted`, ad-* stay denied.
Banner Reject-all → `analytics_storage: denied`, `preference: denied`, ad-* stay denied.
Banner Customize + per-toggle → reflects the toggles, ad-* stay denied.

---

## Storage

### The new cookie

Name: `og_consent`
Value: URL-encoded JSON of `ConsentData`
Attributes: `Secure; SameSite=Lax; Path=/; Max-Age=31536000` (1 year)

The `Secure` flag is acceptable in production (HTTPS-only) and tolerated in localhost development by all modern browsers when served from `https://`. In pure-HTTP local dev, the helper falls back to non-Secure with a one-line dev-only comment.

### What stays in `globalCookie`

`globalCookie` keeps `dashboardCardSize`, `portalCardSize`, `lang`. The `cookieConsent` field is removed from the type.

### Card-size gating

`globalCookie.portalCardSize` and `globalCookie.dashboardCardSize` are written today regardless of consent. Per the gating policy:

- If `og_consent.preference === 'granted'`, the helpers write the cookie as today.
- If `og_consent.preference === 'denied'`, the helpers no-op the write. The in-memory Zustand store still updates; the user's choice survives the session but doesn't persist across visits.
- If `og_consent` is absent (user hasn't decided yet), the helpers also no-op the write — defaults are denied per Consent Mode v2 framing.

Language persistence is URL-only in this codebase today; no language cookie write exists. The next-Mastermind handoff brief at `.agent/handoffs/consent-mode-v2-followups.md` queues the work to persist language and theme as preference cookies, which will inherit the same gating helper without further design.

### Migration

**Not built.** The originally-planned one-time `globalCookie.cookieConsent → og_consent` migration (a client-side banner-mount read plus a server-side SSR fallback) was never implemented — it was superseded by the cookies-closing work. There is no migration code in `oglasino-web`: `og_consent` is the sole consent cookie and there is no legacy-cookie fallback path. (Re-confirmed against current `oglasino-web` code, 2026-06-01.)

The legacy `cookieConsent` field was removed from the `GlobalCookie` type during this feature. Any code that writes `globalCookie` after this feature no longer includes `cookieConsent`, so the legacy value naturally expires from cookies as users get re-served.

---

## SSR consent-defaults snippet

### Where it lives

`app/layout.tsx` is the root server layout. The snippet is rendered there, inside `<head>`, before any third-party script loads. This is the only place in `oglasino-web` that emits this snippet.

### Why not `next/script`

`next/script` defers execution. Consent Mode v2 requires the `gtag('consent', 'default', ...)` call to execute *before* `gtag.js` loads — synchronously, inline in `<head>`. A raw `<script>` tag (not `<Script>`) is the right tool. This is the one and only place in the codebase that uses a raw script tag; everywhere else continues to use `next/script` for non-blocking loads.

### The snippet

```tsx
// app/layout.tsx (server component)
import { cookies } from 'next/headers';
import { readConsentForSsr } from '@/lib/consent/ssr';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const consent = await readConsentForSsr(cookies());

  return (
    <html lang="en">
      <head>
        <script
          dangerouslySetInnerHTML={{
            __html: `
              window.dataLayer = window.dataLayer || [];
              function gtag(){dataLayer.push(arguments);}
              gtag('consent', 'default', {
                ad_storage: '${consent.ad_storage}',
                ad_user_data: '${consent.ad_user_data}',
                ad_personalization: '${consent.ad_personalization}',
                analytics_storage: '${consent.analytics_storage}',
                wait_for_update: 500,
              });
              window.__og_consent_loaded = true;
            `,
          }}
        />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

`window.__og_consent_loaded = true` is a sentinel the GA4 v1 init logic checks before calling `gtag('config', ...)`. If for any reason the snippet didn't run, GA4 falls back to safe defaults rather than initializing with no consent state.

`readConsentForSsr` is in `src/lib/consent/ssr.ts`. It reads `og_consent` and returns a fully-populated `ConsentData` (defaults to all-denied when the cookie is absent). There is no legacy-cookie fallback — the migration shim was never built. The function never throws.

The values are interpolated as string literals. They are typed `'granted' | 'denied'` so SQL-injection-style concerns don't apply, but the parser still validates the values match the literal union before interpolation, as defense in depth.

### Client-side `gtag('consent', 'update', ...)`

When the user makes a decision in the banner or the settings toggle, the new code calls:

```ts
window.gtag?.('consent', 'update', {
  analytics_storage: newConsent.analytics_storage,
  ad_storage: 'denied',
  ad_user_data: 'denied',
  ad_personalization: 'denied',
});
```

`window.gtag` is set up by GA4 v1's script load. In this consent feature, the helper exists and is called, but it's a no-op until GA4 v1 ships. That's fine — `window.gtag?.` is a safe guard.

---

## Banner UX

### Drawer layout

Two states:

**Default state — Twin primaries + Customize trigger:**

```
┌─────────────────────────────────────────────────┐
│ We use cookies                                  │
│                                                 │
│ We use cookies and similar technologies to     │
│ provide essential site functions, remember     │
│ your preferences, and (with your permission)   │
│ measure how the site is used.                   │
│                                                 │
│ [Customize]                                     │
│                                                 │
│ [Reject all]              [Accept all]          │
└─────────────────────────────────────────────────┘
```

**Customize state — Per-category toggles:**

```
┌─────────────────────────────────────────────────┐
│ We use cookies                                  │
│                                                 │
│ Choose which categories to allow.               │
│                                                 │
│ ◉ Strictly necessary             [ON, locked]   │
│ ○ Preference cookies             [toggle]       │
│ ○ Analytics cookies              [toggle]       │
│                                                 │
│ Marketing cookies: we do not use them.          │
│                                                 │
│ [Back]    [Reject all]   [Save my choices]      │
└─────────────────────────────────────────────────┘
```

### CTA semantics

- **Accept all**: `analytics_storage: granted`, `preference: granted`. ad-* stay denied.
- **Reject all**: `analytics_storage: denied`, `preference: denied`. ad-* stay denied.
- **Save my choices**: reflects whatever the user toggled.
- **Customize**: expands to per-category view; doesn't persist anything.
- **Back**: returns to default view; doesn't persist anything.

All paths write `og_consent` with the chosen values, set `decidedAt = now`, close the Drawer, and call `window.gtag?.('consent', 'update', ...)`.

### Drawer behavior

- First-time visitor: Drawer opens on mount, default state.
- Returning visitor with `og_consent` set: Drawer does not open.
- Footer link click: Drawer opens, **showing the current state** of consent (toggles reflect what's persisted).
- Drawer cannot be dismissed by clicking outside or pressing Escape on first-time-visitor display. It can be dismissed (via close X) when reopened via the footer link, because at that point a decision already exists.

This asymmetry is intentional: first-time visitors must make a decision; returning visitors are revisiting an existing one.

---

## Footer link

Insert a new entry "Cookies" in `companyNavigations.tsx` as the last entry, after Terms of Use. The entry is a client component button that calls a `reopenConsentBanner()` helper, which sets a Zustand flag the banner subscribes to.

Display order in the footer's company navigation after this change: About → Pricing → Privacy → Terms → Cookies.

The footer is mounted only by `(portal)/(public)/layout.tsx`, `(portal)/(protected)/favorites/layout.tsx`, and `(portal)/(protected)/notifications/layout.tsx`. The "Cookies" entry is reachable on those pages and not on `/owner/*`, `/admin/*`, `/messages`, or any other page that doesn't mount the footer. This is a pre-existing pattern in the codebase, not a Consent Mode v2 design choice.

The gap is accepted. Owner, admin, and messages surfaces are reachable only by authenticated users; those users revisit their consent decision through the dedicated `/owner/cookies` page (see § Logged-in user settings).

Translation key: `cookies.footer.link.label` in the `COOKIES` namespace. Locale seeds: EN / RS / RU / CNR via the feature's translation seed pass.

---

## Logged-in user settings

Logged-in users revisit their consent decisions on a dedicated page at `/owner/cookies`. Sidebar entry: `User → [Settings, Cookies]` — Cookies is a sibling to Settings under the User section, labeled with the new `NAVIGATION.owner.account.cookies.label` key.

The page hosts three toggles, vertically stacked:

- **Necessary** — always-on, disabled. Labels: `cookies.settings.necessary.label`, `cookies.settings.necessary.description`, `cookies.settings.necessary.sub.description`.
- **Preference** — toggleable. Labels: `cookies.settings.preference.label`, `cookies.settings.preference.description`.
- **Analytics** — toggleable. Labels: `cookies.settings.analytics.label`, `cookies.settings.analytics.description`.

Immediate-write semantics: each toggle's `checked` derives from `useConsentStore` via a live hook selector. Flipping a toggle calls `applyConsent(next)` immediately — writes `og_consent`, fires `gtag('consent', 'update', ...)`. No Save button. The page's UX matches the banner: a decision flips state immediately, no commit step.

Hydration UX: the two toggleable switches are disabled until `useConsentStore.hydrate()` resolves. After hydration, the live selector takes over.

No backend mirror. Consent state is browser-local. Cross-device propagation is out of scope for v1 and tracked as a follow-up in the next-Mastermind handoff brief.

---

## Translation seeds

New keys in the `COOKIES` namespace (20 keys, brief 8):

```
banner.title
banner.body
banner.accept_all.label
banner.reject_all.label
banner.customize.label
banner.save.label
banner.back.label
banner.close.label
category.necessary.label
category.necessary.description
category.preference.label
category.preference.description
category.analytics.label
category.analytics.description
category.marketing.note
footer.link.label
settings.preference.label
settings.preference.description
settings.analytics.label
settings.analytics.description
```

Additional keys seeded for the dedicated `/owner/cookies` page (4 keys, brief A):

```
settings.necessary.label
settings.necessary.description
settings.necessary.sub.description
page.title
```

One additional key in the `NAVIGATION` namespace for the new sidebar entry (1 key, brief C):

```
owner.account.cookies.label
```

Total: 25 keys across 2 namespaces.

Seed rows in all four locales (EN / RS / RU / CNR). **Inline-append** to the end of each existing `0001-data-web-translations-{LOCALE}.sql` file's relevant namespace section. Conventions Part 6 Rule 3's dedicated-file exception is reserved for multi-namespace seeds and does not apply to single-namespace seeds regardless of key count.

The legacy keys deleted as part of this feature's cleanup (briefs 7b + C):
- COOKIES.banner.header, banner.description, banner.required.label, banner.preference.cookies, banner.accept.label (5 keys, brief 7b)
- COOKIES.config.label, config.description (2 keys, brief 7b)
- COOKIES.required.label, required.description, required.sub.description (3 keys, brief C)

EN copy is final per Mastermind. RS / RU / CNR copy lands as placeholders pending native-translator review, matching the 2026-05-19 User Deletion precedent. Each placeholder block carries an in-file comment marking the rows for review.

---

## Implementation order — Phase 5 brief sequence

1. **Foundation**: new `ConsentData` type, `og_consent` cookie helpers (`read`, `write`), Zustand store for consent state, SSR `readConsentForSsr` helper.
2. **SSR snippet**: `app/layout.tsx` server-side read + inline `<script>` emission. No banner changes yet.
3. **Banner rebuild**: new Drawer UX (twin CTAs + Customize), wired to `og_consent`. New banner replaces the old one in `(portal)/layout.tsx` and `(owner)/layout.tsx`.
4. **Footer link + reopen flow**: `companyNavigations.tsx` entry + Zustand flag + banner subscribe.
5. **Settings page integration**: `/owner/cookies` toggle rewires to `og_consent`; analytics toggle added.
6. **Card-size + language gating**: usePortalCardSize / useDashboardCardSize / language cookie writes check `og_consent.preference`.
7. **Cleanup**: delete dead `firebaseAnalytics.ts`, delete `PREFERENCE_COOKIE_NAME` alias, remove old `cookieConsent` field from `GlobalCookie` type.
8. **Translations + locale seed**: SQL file with all four locales.
9. **Docs cleanup**: this feature spec is updated post-shipping to `shipped` status; legal notification fires.

Brief 1 lands the type and helpers without changing user-visible behavior. Each subsequent brief is independently mergeable and testable. Briefs 1-3 are the load-bearing ones; the rest are layered on top.

---

## Cross-repo seams

- **No backend consent mirror.** The original spec mirrored `preference` to `AuthUserDTO.allowPreferenceCookies`. Brief C removed the field end-to-end (column, DTOs, entity, propagator code). Consent state is browser-local. The next-Mastermind handoff brief queues the cross-device consent sync work that may revisit this decision.
- **Backend translation seeds.** Brief 8 and Brief A seeded 24 keys in the `COOKIES` namespace; Brief C added 1 key in the `NAVIGATION` namespace. All four locales (EN / RS / RU / CNR) inline-appended to `0001-data-web-translations-*.sql`. Brief 7b deleted 7 legacy COOKIES keys; Brief C deleted 3 legacy COOKIES keys.
- **No backend changes to `/auth/firebase-sync` request shape** beyond removing the `allowPreferenceCookies` field. Wire payload is now `{ displayName?: string }` only.
- **Firestore Rules.** No changes. Consent doesn't touch any Firestore-backed surface.
- **Router worker.** No changes.

---

## Test plan

- **Unit (web).** `og_consent` read/write helpers. `readConsentForSsr` paths: cookie present, cookie absent (all-denied default). `gtag('consent', 'update', ...)` is called with the correct arguments on each banner CTA.
- **Component (web).** Banner default-state and customize-state render correctly. Twin CTAs each write the right values. Footer link reopens the banner with current state pre-populated. Settings page toggles bidirectionally synced with `og_consent`.
- **Integration (web).** SSR snippet emits the correct `gtag('consent', 'default', ...)` call for each cookie state. Cookie gating: card-size write no-ops when `preference === 'denied'`.
- **Manual.** Open in private window, see banner, click each CTA path, verify cookie state. Verify the footer link works in all three locales. Verify the settings toggle sync is bidirectional.

---

## Definition of done

- All 9 implementation steps shipped and merged to `dev`.
- Test coverage per Test plan above.
- `og_consent` cookie verifiable in browser devtools, with correct shape and attributes.
- SSR snippet visible in page source on first paint, before any other script.
- Banner UX matches the wireframes in this spec.
- Card-size and language cookies no-op when consent denies preference.
- Dead `firebaseAnalytics.ts` and `PREFERENCE_COOKIE_NAME` alias deleted.
- Translation seeds in for all four locales.
- Docs/QA notifies the legal-drafts chat that consent UX has changed.
- `state.md` updated: status `shipped`, GA4 v1 unblocked.
- `decisions.md` carries the closing entry summarizing the feature.

---

## Open items at draft time

None. All Phase 3 decisions (D1–D10, D5 and D6 expanded) are folded in. Ready for Phase 5 briefing.

---

## Platform adoption

| Platform | Status        | Notes                                                                 |
|----------|---------------|-----------------------------------------------------------------------|
| Web      | `shipped`     | This spec.                                                            |
| Mobile   | `not-started` | Separate Mastermind chat. Mobile has its own consent UX considerations (App Tracking Transparency on iOS, etc.) and a different SDK story; not folded in here. |
