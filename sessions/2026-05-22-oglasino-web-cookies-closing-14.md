# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-22
**Task:** Fix the double-locale URL bug in the user-page cross-tenant watcher (Item 4, Brief 3e). Reproducer: at `/rs-cnr/user/8453`, expect redirect to `/rs-sr/user/8453`; actual is `/rs-sr/rs-cnr/user/8453`.

## Implemented

- Step 1 — Investigation only. No code changes this session.
- Read the user-page cross-tenant watcher added by Brief 3b at `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:50-62`. URL construction is `redirect(\`/${targetBaseSite.code}-${finalLocale}/user/${userDetails.id}\`)`. The variables — `targetBaseSite.code` (a plain string like `'rs'`), `finalLocale` (a plain language code like `'sr'`), `userDetails.id` (a number) — cannot, in any combination, emit a path containing two locale segments. No `pathname` is involved in this template.
- Compared against the product page at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:71-73`: `redirect(\`/${targetBaseSite.code}-${finalLocale}${getNormalizedProductUrl(productDetails.id, productDetails.name, false)}\`)`. `getNormalizedProductUrl(id, name, false)` returns `/product/{id}/{slug}` (verified at `src/lib/utils/utils.ts:116-123`). The product page redirect emits `/{code}-{lang}/product/{id}/{slug}`. The user page redirect emits `/{code}-{lang}/user/{id}`. **Same shape. No diff between them in URL construction.**
- Followed the brief's "common alternative" — checked whether middleware is producing the doubled locale on the redirect target. Read `proxy.ts` and the next-intl middleware bundle at `node_modules/next-intl/dist/esm/production/middleware/middleware.js` plus its utils (`getNormalizedPathname`, `getPathnameMatch`, `formatPathname`). next-intl's `getPathnameMatch` builds a regex from the literal entries in `routing.locales`. `rs-cnr` is not in `routing.locales` (`src/i18n/routing.ts:5-16` lists `rs-sr`, `rs-en`, `rs-ru`, `rsmoto-*`, `me-sr`, `me-cnr`, `me-en`, `me-ru`). When the incoming pathname `/rs-cnr/user/8453` is checked, no match is found, `getNormalizedPathname` returns the pathname unchanged, and the middleware redirects to `formatPathname(z, defaultLocalePrefix, search)` — which prepends `/rs-sr` to `/rs-cnr/user/8453`. The result is exactly `/rs-sr/rs-cnr/user/8453`. That is the observed bug, and it is produced by next-intl, not by the user-page `redirect()` call.
- Confirmed `proxy.ts` does not catch this case before next-intl runs: its `LOCALE_PREFIX` regex (`/^\/([^/-]+)-([^/]+)(\/.*)?$/`) parses `/rs-cnr/...` as `tenant='rs'`, `urlLang='cnr'`. It only redirects when the cookie's language differs from the URL's language **and** the cookie's language is in `ALLOWED_LANGS_PER_TENANT[tenant]`. It does not redirect when the URL's own `urlLang` is invalid for the tenant. So invalid tenant-lang combinations entered manually (or returned by any future code path) fall through to next-intl, which then prepends the default locale.
- Verified the four sanity scenarios from the brief by code reading (no execution this session, since I am not changing code):
  - Scenario 1 (cookie preservation, `/me-cnr/product/<rs-owned>` → `/rs-sr/product/<id>/<slug>`): unchanged from Brief 3b. The route is `/me-cnr/...` which is a valid locale; the product page renders, detects mismatch, builds `/rs-sr/product/...`, and redirects. Cookie `lang='cnr'` is preserved because `proxy.ts` on the redirect target finds `cnr` not in rs's allowed langs and falls through.
  - Scenario 2 (`/me-cnr/user/<rs-user>` → `/rs-sr/user/<id>`): the user-page is reached because `/me-cnr/` is a valid locale. `oglasinoLocale='cnr'`, rs's `allowedLanguages` (sr/en/ru, mirroring `routing.locales`) does not include cnr, so `finalLocale='sr'`. `redirect('/rs-sr/user/<id>')`. The target locale IS valid; next-intl does not mangle it. **This scenario already works with the Brief 3b code as-is.**
  - Scenario 3 (`/rs-sr/user/<me-user>` → `/me-sr/user/<id>`): `/rs-sr/` valid. user-page renders. Domain mismatch (rs vs me). `oglasinoLocale='sr'`, me allows sr, `finalLocale='sr'`. `redirect('/me-sr/user/<id>')`. Target valid. Works.
  - Scenario 4 (`/rs-en/user/<rsmoto-user>` → no redirect): `/rs-en/` valid. user-page renders. `baseSite.domain === user.baseSiteOverview.domain` (rs and rsmoto share a domain). No redirect. Works.
- **Conclusion.** The brief's hypothesis ("the bug is in the URL Brief 3b's user-page watcher builds") does not survive contact with the code. The user-page redirect URL is built correctly; the bug surfaces only when the user navigates to an invalid tenant-lang URL (`/rs-cnr/...`) directly, in which case the user-page never executes — next-intl middleware redirects first, prepending the default locale to the entire path including the invalid segment. Per the brief's instruction ("If the bug is not where Mastermind expects (URL construction), surface in 'Brief vs reality' and stop"), I did not make any code changes.

## Files touched

- None.

## Tests

- Ran: none. No code changed; the Brief 3d/3b baseline is already clean (per session summary `2026-05-22-oglasino-web-cookies-closing-13.md` and `-11.md` respectively).
- Result: N/A.
- New tests added: none.

## Cleanup performed

- None needed. No code edited, no files created.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed. No edits, no debug logging, no TODOs added.
- Part 4a (simplicity): see structured evidence in "For Mastermind." Three "considered and rejected" entries dominate this session because the appropriate action was to challenge the brief, not write code.
- Part 4b (adjacent observations): one observation noted below.
- Part 11 (trust boundaries): confirmed. No new trust-relevant code introduced; the existing Brief 3b user-page guard's trust-boundary analysis from `2026-05-22-oglasino-web-cookies-closing-11.md` still holds. Tenant routing is decided server-side from URL + cookie + backend `baseSiteOverview`, never from client-supplied request fields.
- Part 5 (closure gate): no implicit config-file dependency. The drafted brief follow-up below is a brief-text suggestion for Mastermind, not a config-file edit.

## Brief vs reality

I read the brief and the code. Before starting work, I found:

1. **The brief assumes the bug is in the user-page redirect URL construction. It is not.**
   - Brief says: "The bug is in the URL Brief 3b's user-page watcher builds. Either: The path it passes to `redirect()` is relative-shaped..., Or the watcher is concatenating the new locale onto `pathname`..., Or `redirect()` is being called with a URL that already includes a locale segment."
   - Code says (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx:61`):
     ```ts
     redirect(`/${targetBaseSite.code}-${finalLocale}/user/${userDetails.id}`);
     ```
     `targetBaseSite.code` is a plain string (`'rs'`, etc.); `finalLocale` is a plain language code (`'sr'`, etc.); `userDetails.id` is a number. `pathname` is not referenced. The result string for the brief's scenario is `/rs-sr/user/8453` — a single, valid locale segment. The product page (`product/[productId]/[productName]/page.tsx:71-73`) has the same shape and the same correctness properties.
   - Why this matters: implementing a "fix" in the user-page redirect URL would either be a no-op or break a currently-correct call site. The behavior the brief describes as the user-page's output (`/rs-sr/rs-cnr/user/8453`) cannot be emitted by this template.

2. **The double-locale URL is produced by next-intl middleware on a direct visit to `/rs-cnr/...`, not by the user-page.**
   - Brief's reproducer (paraphrased): "Visit `/me-cnr/user/8453`. Manually change URL to `/rs-cnr/user/8453`. Expected `/rs-sr/user/8453`, actual `/rs-sr/rs-cnr/user/8453`."
   - Code says: at `/rs-cnr/user/8453`, `proxy.ts:33-56` only redirects when the cookie's language differs from the URL's language and is in the tenant's allowed set; an invalid `urlLang` for the tenant is not caught. The request falls through to `defaultMiddleware(request)` (next-intl). next-intl's `getPathnameMatch` (`node_modules/next-intl/dist/esm/production/middleware/utils.js` function `a`) builds its regex strictly from entries in `routing.locales`. `rs-cnr` is not in that list (`src/i18n/routing.ts:5-16`), so no match. `getNormalizedPathname` returns the pathname unchanged. The middleware then issues a redirect to `formatPathname(unchanged-pathname, defaultLocalePrefix, search)` — i.e. `/rs-sr` + `/rs-cnr/user/8453` = `/rs-sr/rs-cnr/user/8453`. The user-page code never executes for this request.
   - Why this matters: the fix lives upstream of the user-page. Specifically, `proxy.ts` needs a new branch that detects "URL parses as `tenant-urlLang` but `urlLang` is not in `ALLOWED_LANGS_PER_TENANT[tenant]`" and redirects to a valid combination (most natural: `getLocalizedPath(pathname, tenantDefaultLang)`, where `tenantDefaultLang` can be derived from the existing static mirror by picking, say, the first entry — or by adding a parallel `DEFAULT_LANG_PER_TENANT` constant). That redirect arrives before next-intl runs and short-circuits the doubling.
   - Recommended resolution: re-target this brief at `proxy.ts`. The user-page (Brief 3b) is not the right surface. The brief explicitly anticipated this outcome under "Common alternative" and instructed me to surface and stop, which I have done.

3. **Brief's verification Scenario 2 (`/me-cnr/user/<rs-user>` → `/rs-sr/user/<id>`) is already passing under Brief 3b.**
   - This is not strictly a "bug in the brief," but worth flagging so Mastermind doesn't re-test a passing case as if it were the bug. The reproducer's "Visit `/me-cnr/user/8453`" first step is the prelude that establishes context (presumably a session/cookie state); the bug surfaces only on the second step (the manual change to `/rs-cnr/`).

I have not started any implementation. Please pass these findings to Mastermind before the next brief.

## Known gaps / TODOs

- None. This session was investigation-only by design.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** Nothing. No code edits.
  - **Considered and rejected:**
    - Editing `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` to switch the redirect URL builder. Rejected — the existing builder is already correct. Changing it would either be cosmetic or introduce a regression.
    - Adding a `pathnameWithoutLocale`-based redirect target in the user-page (per the brief's Step 2 suggestion). Rejected — the user-page does not use `pathname` in its redirect target today. Introducing it would be a step backwards (the current pattern is "build from typed parts," which is safer than "manipulate the request URL"). Also, `pathnameWithoutLocale` (`src/lib/utils/utils.ts:10-13`) strips the first segment unconditionally, including for paths that have no locale prefix — using it on an absolute target string in `redirect()` would mangle paths.
    - Editing `proxy.ts` to handle the invalid-tenant-lang case. Rejected for this session — that is a different surface than the brief targets, the brief instructs me to surface and stop rather than expand scope on my own initiative, and the proxy fix needs Mastermind to decide whether the invalid combination should redirect to the tenant's default lang (my recommendation) or 404 (an alternative).
    - Adding a route guard inside the user-page that validates `${tenant}-${finalLocale}` against `routing.locales` before passing to `redirect()`. Rejected — defense in depth in a place that already builds valid combinations is just additional code with no behavior change for any currently-reachable input. The invalid-input case is not reachable through the user-page; it is reached directly from the URL bar.
  - **Simplified or removed:** Nothing.

- **Recommended next brief.** Brief 3e-bis (or 3e renamed) targeting `proxy.ts`. Sketch of the fix:

  ```ts
  // After parsing tenant/urlLang from LOCALE_PREFIX, but before the cookie check:
  if (ALLOWED_LANGS_PER_TENANT[tenant] && !ALLOWED_LANGS_PER_TENANT[tenant].has(urlLang)) {
    // urlLang is not valid for this tenant. Redirect to a valid combination.
    // Default choice: the tenant's first allowed lang (= routing.locales-derived default).
    const fallbackLang = [...ALLOWED_LANGS_PER_TENANT[tenant]][0];
    const redirectUrl = new URL(request.url);
    redirectUrl.pathname = getLocalizedPath(pathname, fallbackLang as Lang);
    return NextResponse.redirect(redirectUrl);
  }
  ```

  Open question for Mastermind: which lang to pick as the fallback. Options:
  - First entry of `ALLOWED_LANGS_PER_TENANT[tenant]` (Set iteration order = insertion order = `routing.locales` order, so for `rs` this is `sr`). Simple, no new state.
  - A `DEFAULT_LANG_PER_TENANT` constant alongside the existing mirror (`{ rs: 'sr', rsmoto: 'sr', me: 'sr' }` or `me: 'cnr'` depending on what Mastermind wants for me).
  - Look up the tenant's `defaultLanguage` from a runtime backend call. Rejected — proxy.ts cannot make a backend call without breaking the edge-fast property.

  My recommendation: option 1 (first entry of the Set). The current `ALLOWED_LANGS_PER_TENANT` derivation in `proxy.ts:16-24` iterates `routing.locales` in order, so the first lang for each tenant is the listed-first lang in `routing.ts` — which for all three tenants today is the intended default per the routing convention. No new constant needed.

- **Part 4b adjacent observation — silent fall-through on invalid tenant-lang URLs is itself a UX bug.** Even before this brief, manually typing `/rs-cnr/user/8453` produces `/rs-sr/rs-cnr/user/8453`, which 404s (no route matches `/user/8453` inside the `/rs-cnr/...` literal segment). A user pasting an URL with an unsupported tenant-lang combination — for example a link shared from a different tenant — gets a 404 instead of a graceful redirect to the closest valid URL. Fixing the proxy as suggested above also fixes this broader UX issue. Severity: medium (user-visible, but rare in practice). File: `proxy.ts`. Not fixed this session — out of scope.

- **Part 4b adjacent observation — the brief's hint about `pathnameWithoutLocale` is a red herring.** The helper exists (`src/lib/utils/utils.ts:10`) and is used at `app/[locale]/(portal)/(protected)/layout.tsx:12` for a different purpose (favicon path detection). It is **not** applicable to the user-page redirect, which never touches `pathname`. Future briefs should not point at this helper for this kind of "build a redirect URL" work — `pathname` manipulation is the wrong tool when the target URL is built from typed inputs (`targetBaseSite.code`, language code, id). Severity: low (brief-text quality, not code). Suggested for the brief drafter rather than a code change.

- **Drafted config-file text.** None.

Brief 3e complete.
