# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-22
**Task:** Fix the `INVALID_MESSAGE: invalid language tag: "me-cnr"` error surfaced on every page that uses interpolated translations under `me-*` locales. Restore the strip from compound routing locale to bare language portion before reaching next-intl's message formatter / Intl APIs.

## Outcome

**Stopped before implementation.** Investigation confirms the brief's stop-gate condition: there is no evidence a strip ever existed at any next-intl boundary, the bug is structurally pre-existing rather than introduced by cookies-closing work, and the brief's proposed one-file fix (swap the locale passed to next-intl from compound `me-cnr` to bare `cnr`) would cascade-break ≥16 client `useLocale()` call sites plus ~22 server `getLocale()` call sites that all rely on the compound form for URL construction and `getTenantLocale` parsing. Per the brief's "If the strip was never there and the bug pre-exists" instruction, I stopped and am routing this to Mastermind.

## Implemented

- Nothing on disk. Investigation only.

## Files touched

- None.

## Tests

- Not run. No code changes.

## Brief vs reality

I read the brief and the code. Before starting work, I found:

1. **The strip was never in place at any next-intl boundary.**
   - Brief says: "Before this chat's work, the compound routing locale (`me-cnr`, `me-sr`, etc.) was stripped to its bare language portion (`cnr`, `sr`) somewhere before reaching `next-intl`'s message formatter / `Intl.*` APIs. That strip is either gone or no longer running at the right boundary. Restore it."
   - Code says:
     - `src/i18n/request.ts` has been identical since the pre-production commit `73ab239` (2026-04-30). It calls `getTenantLocale(locale)` to derive `tenant` and `oglasinoLocale` for the namespace loader (used as the `lang` URL parameter against the backend's `/public/translations?lang=…&namespace=…` route — request shape, not Intl), and returns `{ locale, messages }` where `locale` is the unchanged compound routing locale (e.g. `me-cnr`). The bare `oglasinoLocale` is never returned to next-intl.
     - `src/i18n/internalRequest.ts` is the namespace dispatcher — pure pass-through, no Intl boundary.
     - `app/[locale]/layout.tsx` has been passing the compound locale to `<NextIntlClientProvider locale={locale}>` since the pre-production commit; only modification since is commit `25d7569` ("Adding navigation progress bar") which did not touch the locale plumbing.
     - `app/layout.tsx` is the only place that already invokes `getTenantLocale(locale).locale` (the BCP-47-shaped `cnr-ME` form) — and that's used solely for the `<html lang="…">` attribute, never wired into next-intl.
     - `git log --all -p -S "oglasinoLocale"` and `git log --all -p -S "messageLocale|formatLocale|formatterLocale"` show no historical strip-style wiring at the next-intl boundary in any commit on any branch.
   - **`me-cnr` has been in `routing.locales` since the same pre-production commit `73ab239`** — Brief 3 only added `localeDetection: false` and `localeCookie: false`; the locale list itself was untouched. Brief 3 also did not touch `request.ts`, `internalRequest.ts`, or `app/[locale]/layout.tsx`. (Brief 3 files: `GlobalCookie.ts`, `oglasinoCookies.ts`, `getLocalizedPath.ts` (new), `useLanguage.ts` (new), `routing.ts`, `PortalConfigDialog.tsx`, `proxy.ts`, `SyncLanguageFromCookie.tsx` (new, since deleted in 3b), `AppInit.tsx` (reverted in 3b).)
   - Why this matters: the brief's stop gate is explicit — "If the strip was never there and the bug pre-exists: surface in 'Brief vs reality' and stop — Mastermind needs to revisit whether this is actually Item 4's responsibility." The bug is structural and pre-existing; it became newly observable because cookies-closing's middleware + cookie-wins logic now routes traffic to `me-cnr` URLs that previously were not visited in practice. The fix is real work, but it is not a "restore previous behavior" one-liner; it is a structural decision about how to separate routing-identity locale from Intl-formatting locale.
   - Recommended resolution: see "For Mastermind" below. The four architectural options span from a trivial-looking but breaking one-line swap to a full routing-locale rename.

2. **The brief's proposed one-line fix breaks the rest of the app.**
   - Brief says (Step 2): "Most likely shape: the next-intl request config passes the compound locale to `getMessages` and `getNow`, but should pass the bare locale (via `getTenantLocale(routingLocale).oglasinoLocale` or whatever the helper exposes for the bare-language portion). Both `getMessages` for translation files and the locale itself when next-intl sets up its `<NextIntlClientProvider>`-equivalent for SSR."
   - Code says: changing the `locale` returned from `getRequestConfig` (and equivalently the `locale` prop on `<NextIntlClientProvider>`) changes what `useLocale()` and `getLocale()` return everywhere downstream. There are **16 client-side `useLocale()` call sites** and an additional ~22 server-side `getLocale()` call sites; representative samples that would break:
     - `src/components/client/SessionGuard.tsx:35,43` — `router.replace(`/${locale}`)`. If `useLocale()` becomes `cnr`, the redirect goes to `/cnr` → 404.
     - `src/components/client/initializers/FilterManager.tsx:312` — `const newUrl = `/${locale}${pathSegment}` + …`. Same break.
     - `src/components/popups/dialogs/PortalConfigDialog.tsx:44–45` — `const tenantLocale = getTenantLocale(locale);` consumed throughout the dialog. `getTenantLocale("cnr")` returns `tenant="cnr", lang=undefined`, which breaks the language switcher's "is this lang active" comparison and the URL construction.
     - `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:57` and `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:47` — both rely on `getTenantLocale(locale).oglasinoLocale` for the SSR cross-tenant redirect computation.
     - `src/components/server/OglasinoBreadcrumbs.tsx`, `src/components/client/buttons/AuthUserProfileButton.tsx`, `src/lib/config/api.ts` (axios `X-Lang` header), `src/lib/service/nextCalls/*Service.ts` (4 files) — all depend on `getTenantLocale(locale)` extracting both `tenant` and `oglasinoLocale` from the compound form.
   - The brief's sketch reads like a one-file change. In reality the surface is wider, and the right framing is "separate routing-identity locale from Intl-formatting locale across the app," not "swap one return value."
   - Why this matters: this isn't a stylistic preference — the change as written would land a passing typecheck and a green test suite (the tests don't exercise `me-cnr` end-to-end), then break the cross-tenant SSR redirects, the language switcher, the URL-canonicalization in FilterManager, the SessionGuard fallback, and the `X-Lang` header on every backend call. The brief's "If the sketch contradicts the repo, follow the repo" license applies, but the contradiction is broad enough that I want Mastermind to pick the architectural shape before I act, rather than picking it for them.
   - Recommended resolution: see "For Mastermind" below.

I have not started the implementation. Please pass these to Mastermind before I continue.

## Cleanup performed

- None needed (no code changes this session).

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change.
- `state.md`: no change.
- `issues.md`: no change. (If Mastermind decides the bug is pre-existing and out of cookies-closing scope, an `issues.md` entry would be appropriate — text drafted below for Mastermind's review.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A this session — no code changes.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one observation about `app/layout.tsx`'s `getTenantLocale(locale).locale` use, flagged below.
- Part 6 (translations): N/A this session — no label changes, no new keys.
- Part 11 (trust boundaries): N/A this session — investigation surfaced no trust-boundary touch points. The Intl-locale change only affects display formatting, and the routing-identity locale is already server-derived.
- Part 5 (closure gate): no implicit config-file dependency from this session's outcome. A `decisions.md` entry would be appropriate once Mastermind picks an option; that draft is the work of whichever chat picks the option.

## Known gaps / TODOs

- The bug remains live on `me-*` locales. Until Mastermind picks an option, every visit to a `me-cnr` (and likely `me-sr`, `me-en`, `me-ru` — see Mastermind question Q1 below) URL that renders an interpolated translation throws the `INVALID_MESSAGE` error.
- Manual verification of the symptom on a running dev server was not performed this session. The brief was an implementation brief, not a reproduction brief; I trust Igor's report that the error is live on `me-cnr` and confirmed it by static analysis of the locale flow. If Mastermind wants reproduction evidence before deciding, that's a separate read-only session.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** Nothing. No code shipped.
  - **Considered and rejected:**
    - Implementing the brief's literal sketch (swap `request.ts`'s return locale to `oglasinoLocale`) — rejected because it cascades through 38 call sites of `useLocale()`/`getLocale()`. The fix would land green on tsc + lint + test (the test suite does not cover `me-*` end-to-end), then break the SessionGuard redirect, the FilterManager URL canonicalization, the cross-tenant SSR redirects on product and user pages, the language switcher in PortalConfigDialog, and the `X-Lang` axios header — a much wider blast radius than the brief implied.
    - Implementing the brief's alternate (`getMessageLocale` helper used at every next-intl boundary) — rejected for the same reason. The "every next-intl boundary" framing is structurally equivalent to changing the provider locale once; either way `useLocale()`/`getLocale()` flips its return shape and the same 38 consumers break.
    - Adding `Intl.Locale`-shaped overrides only on the formatter path (e.g., a wrapping `t()` that explicitly passes `formatLocale` per call) — rejected for this brief. The next-intl public API does not expose a per-call format-locale override; the only place locale is configured is the provider. Working around this would mean a custom translator hook that wraps `useTranslations` and re-parses ICU manually — heavy, untested, premature.
    - Proceeding with an audit-then-fix instead of stopping — rejected because the brief is explicit ("surface in 'Brief vs reality' and stop"). Stopping is the instructed action.
  - **Simplified or removed:** Nothing.

- **Recommended options for Mastermind to pick from.** All four address the same root cause: `me-cnr` is not a valid Unicode/BCP-47 locale identifier, so `IntlMessageFormat`'s `new Intl.Locale("me-cnr")` throws. The differences are blast radius and how cleanly each separates routing-identity from formatting-identity.

  - **Option A — Override only the `locale` prop on `<NextIntlClientProvider>` and the SSR `getRequestConfig` return, plus add a thin `useRoutingLocale()` hook that wraps next-intl's `useLocale()` with the compound-locale recovery.**
    Sketch: `request.ts` returns `locale: getTenantLocale(routingLocale).locale` (the BCP-47 form `cnr-ME`); layout passes the same to the provider. Introduce `src/i18n/useRoutingLocale.ts` that internally uses `useLocale()` (which now returns `cnr-ME`) and reconstructs the compound `me-cnr` form for callers that need URL construction. Migrate the 16 `useLocale()` client call sites to `useRoutingLocale()`. Server side, do the same wrap around `getLocale()`. This is the cleanest separation but it's an explicit, ~38-site mechanical migration.
    *Risk:* `cnr-ME` → `me-cnr` reconstruction needs a mapping table (essentially the inverse of `getTenantLocale`); needs to be locked-step with `routing.locales`. Achievable but mechanical.

  - **Option B — Same as A, but pass the bare `oglasinoLocale` (`cnr`) instead of the BCP-47 form (`cnr-ME`).**
    Sketch: identical to A except the formatter locale is `cnr` (no region). ICU plural rules and date/number formatting still resolve cleanly for `cnr`/`sr`/`en`/`ru`. Slightly cheaper because `routing.locales` already encodes the bare lang in its second segment — no separate mapping table needed. Same ~38-site migration cost.
    *Risk:* date and currency formatting will use region-neutral defaults instead of `ME`/`RS` regional conventions. Today the app does not appear to use region-dependent formatting heavily (the only direct `Intl.*` use is `Intl.RelativeTimeFormat` in `src/components/admin/es/timeFormat.ts`, which is region-agnostic), so this is likely a non-issue. Worth a 5-minute audit before committing to B over A.

  - **Option C — Rename `routing.locales` to BCP-47-compliant compound form: `cnr-ME`, `sr-RS`, `en-RS`, `ru-RS`, `sr-ME`, `en-ME`, `ru-ME`, `sr-RSMOTO`(?), etc.**
    Sketch: change the routing convention from `<tenant>-<language>` to `<language>-<region>`. URLs become `/cnr-ME/...`, `/sr-RS/...`. Routing identity is now BCP-47 by construction; no separation needed because the compound IS a valid Intl locale. `getTenantLocale` re-shapes to "extract tenant from the region portion" (`RS` → `rs`, `ME` → `me`, …).
    *Risk:* large user-visible URL change; every external link / bookmark / SEO entry breaks. Requires Cloudflare router worker rewrites for backward compatibility, plus a 301-redirect strategy. Out of proportion for this fix unless Mastermind has other reasons to move to BCP-47 URLs. Also breaks the `rsmoto` tenant (no region code maps to it) — would need a tenant-region map that can't be one-to-one.

  - **Option D — Keep the compound routing locale but configure next-intl to *not* validate the locale string against BCP-47 / `Intl.Locale`.**
    Sketch: next-intl v4 does not expose a "skip locale validation" knob. `intl-messageformat` (the underlying library) calls `new Intl.Locale(locale)` internally on first message-format construction; that's the throw point. Working around requires monkey-patching `Intl.Locale` (cross-cutting global side effect, fragile) or forking the library.
    *Risk:* high. Strongly do not recommend.

  - **My read:** Option B is the most likely correct choice. The migration is mechanical (an `useRoutingLocale()` hook plus a 38-site grep-and-replace, all behind one PR), it preserves the existing URL shape entirely, and it cleanly separates the two concepts that the code today conflates. Option A is a strict superset of B's complexity for no obvious win. Option C is a much bigger change that would only make sense if BCP-47 URLs are themselves desirable. Option D is a dead end.

- **Questions for Mastermind:**
  - **Q1.** Does the error fire on all `me-*` locales (`me-sr`, `me-en`, `me-ru`, `me-cnr`), or only on `me-cnr`? `me-sr` parses as "Mende language, Serbian variant" in BCP-47 grammar — both are valid 2/3-letter subtags, so `Intl.Locale("me-sr")` may or may not throw depending on the JS runtime's strictness. Worth Igor confirming via a quick visit to `/me-sr/product/...` and `/me-en/product/...`. The fix is the same regardless, but knowing the symptom surface tightens the regression-check plan.
  - **Q2.** Is the cookies-closing chat the right home for this fix, given the bug is structurally pre-existing and only became observable because cookies-closing's middleware now routes traffic to previously-unreachable `me-*` URLs? Two reasonable Mastermind verdicts:
    - "Yes, in scope" — cookies-closing made it observable, cookies-closing fixes it. Item 4's brief originally framed the work as language preference cookies; this is a load-bearing piece of language plumbing that gates everything else.
    - "No, separate brief" — pre-existing bugs get their own feature spec / brief, not folded into the originating chat. There is a precedent for both in the project (see the `2026-05-17` bug-chat closeout where cross-cutting infra bugs got separate sessions, vs. the in-line product-validation cleanup bundled into product-validation Brief 4).
  - **Q3.** Should the fix scope include the `app/layout.tsx`-side `getTenantLocale(locale).locale` for `<html lang="...">`? Today line 56 uses `htmlLang.split('-')[0]`, which produces `cnr` from `cnr-ME` — already valid HTML `lang`. So this site is fine. Worth noting in the fix brief that the root layout's HTML attribute is unaffected and does not need migrating.

- **Part 4b adjacent observations.**
  - **`app/layout.tsx:37` already does the strip-and-shape that `request.ts` arguably should.** It calls `getTenantLocale(locale).locale` to get `cnr-ME` for the HTML `lang` attribute. This is the established pattern for "I need a valid Intl identifier from the routing locale." `request.ts` could borrow the same pattern. Severity: low. Out of scope for an investigation-only session; noted here so whoever writes the fix brief knows the precedent exists.
  - **`getPathname` from `src/i18n/navigation.ts` is still exported and unused** (carried from Brief 3's session summary). Severity: low. Out of scope. Not actioned this session.
  - **`useLanguageStore.lang` still has no readers** (carried from Brief 3b's session summary). Severity: low. Out of scope. Not actioned this session. Worth folding into whichever brief actions the language-locale separation, since the `useLanguageStore` shape might naturally subsume the "routing locale ↔ format locale" mapping if Option A or B is chosen.

- **Drafted config-file text.** None yet, pending Mastermind's option choice. If the verdict is "out of cookies-closing scope, log to backlog," the suggested `issues.md` entry text is:

  > **2026-05-22 — `INVALID_MESSAGE: invalid language tag` on `me-*` locales (medium).** Visiting any URL under `me-cnr` (and possibly other `me-*` locales — needs Igor confirmation) that renders an interpolated translation throws `INVALID_MESSAGE: invalid language tag: "me-cnr"`. Root cause: `me-cnr` is not a valid BCP-47 / Unicode locale identifier; `IntlMessageFormat`'s `new Intl.Locale(locale)` rejects it. Pre-existing structurally; became observable on 2026-05-22 because cookies-closing's cookie-wins middleware now routes traffic to `me-*` URLs that were previously unreachable. Surfaced in `Rating.tsx:78`. Fix scope: separate routing-identity locale (compound `me-cnr`) from formatting-identity locale (bare `cnr` or BCP-47 `cnr-ME`); requires migrating ~38 call sites of `useLocale()`/`getLocale()` to a wrapper hook. See `oglasino-web/.agent/2026-05-22-oglasino-web-cookies-closing-12.md` for the four architectural options.

Brief 3c stopped before implementation per brief's stop-gate. Awaiting Mastermind verdict.
