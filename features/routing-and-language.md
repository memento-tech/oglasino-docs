# Routing and Language

Permanent reference for the routing + language + `globalCookie` infrastructure in `oglasino-web` as it works post cookies-closing (2026-05-22). This is a reference doc, not a spec. For the decisions that produced this shape, see [decisions.md](../decisions.md) 2026-05-22 "Cookies-closing" entry and [features/cookies-closing.md](cookies-closing.md). For deferred theme work, see [`../.agent/handoffs/theme-path-a.md`](../.agent/handoffs/theme-path-a.md).

A new engineer joining the team should be able to understand the routing/language system from this doc alone — plus the briefs and audits in `.agent/` if they want history.

---

## 1. Overview

`globalCookie` is the browser-local JSON cookie that carries user preferences across page loads. Fields today:

- `lang` — explicit user language preference (`'sr' | 'en' | 'ru' | 'cnr'`).
- `portalCardSize`, `ownerCardSize`, `adminCardSize` — card-size preference per scope.
- `theme` — reserved (`'light' | 'dark'`); not yet written by any code path. See [`../.agent/handoffs/theme-path-a.md`](../.agent/handoffs/theme-path-a.md).

Storage shape: a single JSON string under the `globalCookie` cookie name, readable from both the client (`document.cookie`) and the edge (`request.cookies.get('globalCookie')` in `proxy.ts`).

Consent gating: every field is consent-gated on write. The cookie is cleared on consent decline. A self-healing migration on app init handles pre-consent-system cookies (clears the cookie when consent is not granted and `globalCookie` contains preference data).

---

## 2. Routing identity vs formatter identity

The system carries two distinct locale concepts. Knowing which to use where is the most common source of bugs.

**Routing locale** — compound `<tenant>-<lang>` (e.g., `rs-sr`, `me-cnr`). Ten values per `routing.locales`. Used for:

- URL construction
- Language switching
- `getTenantLocale` parsing
- `X-Lang` and `X-Base-Site` outbound headers
- Cross-tenant comparison decisions

**Formatter locale** — bare language (e.g., `sr`, `cnr`). Used by `next-intl`'s message formatter and by `Intl.*` APIs. Must be a valid BCP-47 tag.

### Primitives

| Use case            | Client                              | Server                              |
| ------------------- | ----------------------------------- | ----------------------------------- |
| Compound routing    | `useRoutingLocale()`                | `getRoutingLocale()`                |
| Bare formatter      | `useLocale()` (from next-intl)      | `getLocale()` (from next-intl)      |

### Rule of thumb

- URL construction, header construction, cross-tenant comparison → compound (`useRoutingLocale` / `getRoutingLocale`).
- `Intl.*` API or `next-intl` message formatting → bare (next-intl's `useLocale` / `getLocale`).

Roughly 22 sites use compound, 4 sites use bare. The compound is the default; reach for bare only at the `Intl.*` / message-format boundary.

### Why the split exists

The compound routing locale `me-cnr` is not BCP-47. `Intl.Locale("me-cnr")` throws. Earlier in cookies-closing Item 4, `request.ts` passed the compound to next-intl's message formatter and crashed on every Montenegrin request. The fix derives the bare language inside `request.ts` for the formatter only; the compound stays as the routing identity everywhere else.

---

## 3. Navigation primitives

All app navigation goes through `@/src/i18n/navigation`. The exports are thin wrappers around `next/link` and `next/navigation` that source the compound routing locale via `useRoutingLocale()` / `useParams()`.

| Export        | Wraps                          | Behavior                                                                                |
| ------------- | ------------------------------ | --------------------------------------------------------------------------------------- |
| `<Link>`      | `next/link` `<Link>`           | `<Link href="/catalog/men">` on `/rs-sr/...` produces `<a href="/rs-sr/catalog/men">`. |
| `useRouter`   | `next/navigation` `useRouter`  | `.push('/admin/products')` from `/rs-sr/...` navigates to `/rs-sr/admin/products`.     |
| `usePathname` | `next/navigation` `usePathname`| Returns the URL with the compound prefix stripped (`/catalog/men`, not `/rs-sr/catalog/men`). |
| `redirect`    | `next/navigation` `redirect`   | Server-side redirect, locale-aware.                                                     |
| `getPathname` | `next-intl`'s `getPathname`    | Exported for symmetry with `createNavigation`. Currently unused — see `issues.md`.     |

### When NOT to use the wrapped primitives

- **SSR cross-tenant `redirect()` calls** that already build full compound URLs (product/user cross-tenant guards). Use plain `redirect()` from `next/navigation` so the URL is not double-prefixed.
- **Explicit-locale URL construction.** When you need to construct a URL for a different locale (e.g., the language switcher in PortalConfigDialog), pass the `{ locale }` override to the wrapper.

### What about `<a>` and `next/link`'s `Link`?

Plain anchors and `next/link`'s `Link` consume URLs directly without prefixing. Use them when you have a fully-qualified URL or external link. The wrapping is only for app-internal paths relative to the current locale.

---

## 4. The cookie-wins middleware (`proxy.ts`)

`proxy.ts` (Next.js 16 naming for what was `middleware.ts` in Next 15) is the sole authority on locale reconciliation between cookie and URL.

Behavior matrix:

| Cookie state                                | URL state                                   | Outcome                                                              |
| ------------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------- |
| Cookie lang ≠ URL lang AND tenant supports cookie lang | Any                                  | 307 redirect to URL with cookie's language.                          |
| Cookie lang ≠ URL lang AND tenant does NOT support cookie lang | Any                          | URL wins. Cookie is preserved as user's preference for next visit.   |
| Cookie absent (no preference data)          | Any                                         | URL wins. No redirect.                                               |
| Any                                         | Invalid `<tenant>-<lang>` combination       | 307 redirect to tenant's default language with same path.            |
| Any                                         | Malformed URL                               | Falls through to `defaultMiddleware` (next-intl). URL wins on error. |

Defensive parsing: malformed cookie JSON or missing fields fall through to URL-wins. No throws. Logging is silent.

Edge case worth knowing: when both the URL's lang is invalid for the tenant AND the cookie's lang is valid for that tenant, the user sees a two-hop redirect (invalid-lang branch → tenant default → cookie-wins → cookie lang). Each redirect is correct in isolation. Logged in `issues.md` as a possible future optimization.

---

## 5. Cross-tenant routing rules

The product and user pages have cross-tenant guards. They use different equality rules — by design.

| Surface     | Equality on        | Why                                                                                                 |
| ----------- | ------------------ | --------------------------------------------------------------------------------------------------- |
| **Products** | `baseSite.code`    | `rs` and `rsmoto` don't share product inventory despite sharing a domain. Code disambiguates.       |
| **Users**    | `baseSite.domain`  | A user belonging to `rs` or `rsmoto` is accessible from both (shared domain). Domain disambiguates. |
| **Catalog**  | No cross-tenant guard | Catalog structure differs per tenant. Visiting `/me-sr/catalog/cars` when `me` doesn't have `cars` is handled by the catalog page's own not-found path. |

When the equality fails, SSR redirects to the resource's home tenant. Both product and user redirects preserve cookie language when not supported by the target tenant.

---

## 6. PortalConfigDialog navigation rules

The dialog handles two distinct user actions with different rules.

**Language change** (same tenant):

- Preserves path. The leading `<tenant>-<lang>` segment is rewritten via `getLocalizedPath`; everything after is preserved.
- Writes `globalCookie.lang` (under consent gate). The cookie is the user's explicit language choice.

**Tenant change** (different tenant):

- `/product/*` and `/catalog/*` paths drop to tenant home (because product IDs and catalog slugs are tenant-scoped).
- Other paths preserve (e.g., `/owner/products` stays at `/owner/products`).
- Does NOT write `globalCookie.lang`. The new language is a computed fallback from `getLanguageCodeOrDefault`, not a user choice.

The shared `getLocalizedPath` helper handles language; an inline regex in PortalConfigDialog handles tenant.

---

## 7. Cookie lifecycle and consent

`globalCookie` is consent-gated end-to-end. The rules differ by direction:

**Write side** — every `globalCookie` field is consent-gated:

- `updateGlobalCookie('lang', lang)` checks `isPreferenceConsentGranted()` before writing.
- Card-size writes via `useCardSizeStore.setSize` follow the same pattern.
- Card-size and theme (when implemented) also follow this pattern.

**Read side** — `getGlobalCookie()` returns the parsed cookie as-is, no gate. The proxy reads the cookie directly via `request.cookies.get('globalCookie')`. The lack of a read-side gate is safe because of the next rule.

**Decline transition** — when consent transitions to denied, `globalCookie` is cleared entirely. Handled in `useConsentStore.setConsent`. After clearance, the read side has nothing to read.

**Migration** — self-healing check on app init. If consent is not granted and `globalCookie` exists with preference data, the cookie is cleared. Runs once per mount. Handles pre-consent-system users gracefully.

**In-memory-wins-on-grant** (card-size, theme when shipped) — when consent transitions to granted, captured in-memory state is written to cookie. Handled by `SyncCardSizeFromCookie` (component-level effect subscribing to consent state).

### Note for language specifically

Language is **cookie-wins**, not in-memory-wins. The cookie is the preference; the in-memory locale reflects the URL (which can be a forced fallback from a cross-tenant redirect, not a user choice). Capturing the URL on consent-grant would silently overwrite the user's actual preference.

Concretely:

- No `SyncLanguageFromCookie` component exists.
- `useLanguageStore` was collapsed during cookies-closing Brief 3b — it had no readers.
- The cookie is set only by `PortalConfigDialog`'s explicit language change.

If a future requirement makes capturing the URL → cookie at consent-grant desirable (it isn't today), it would be implemented at the consent-grant transition handler, not at the URL-read site.

---

## 8. `globalCookie` schema

```ts
type Lang = 'sr' | 'en' | 'ru' | 'cnr';

type GlobalCookie = {
  portalCardSize?: CardSelectionType;
  ownerCardSize?: CardSelectionType;
  adminCardSize?: CardSelectionType;
  lang?: Lang;
  theme?: 'light' | 'dark';  // Reserved; not yet written by any code path.
};
```

All fields are optional. Cookie absence is normal (first-visit users, denied-consent users).

---

## 9. Routing locales reference

The static source of truth is `routing.locales` in `src/i18n/routing.ts`. Ten values:

- `rs-sr`, `rs-en`, `rs-ru` — RS tenant, 3 languages.
- `rsmoto-sr`, `rsmoto-en`, `rsmoto-ru` — RSMOTO tenant, 3 languages.
- `me-sr`, `me-cnr`, `me-en`, `me-ru` — ME tenant, 4 languages (`cnr` only available here).

Defaults:

- `defaultLocale = 'rs-sr'`.

Derived constants:

- `ALLOWED_LANGS_PER_TENANT` in `proxy.ts` derives from `routing.locales` at module load. Used by the cookie-wins middleware to check whether a tenant supports a given language.

---

## 10. Things deliberately NOT in scope (and where they live)

- **Theme persistence** (Path A — `next-themes` replacement). Deferred to a future Mastermind chat. Full context in [`../.agent/handoffs/theme-path-a.md`](../.agent/handoffs/theme-path-a.md).
- **'system' theme option in UI.** Logged to [issues.md](../issues.md) as a separate UI improvement opportunity.
- **Card-size and theme reconciliation semantics.** Spec amendment captured in [features/cookies-closing.md](cookies-closing.md) decision #7.
- **Cross-device consent sync.** Deferred to the [`../.agent/handoffs/consent-mode-v2-followups.md`](../.agent/handoffs/consent-mode-v2-followups.md) handoff.
