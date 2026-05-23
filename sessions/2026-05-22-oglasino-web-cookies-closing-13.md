# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-22
**Task:** Fix the `INVALID_MESSAGE: invalid language tag: "me-cnr"` error. Separate routing locale from formatter locale per Mastermind's Option B — formatter locale becomes the bare language portion; routing locale stays compound for URL construction.

## Implemented

- Introduced `useRoutingLocale()` (client) and `getRoutingLocale()` (server) primitives. Client reads from `useParams().locale` (the `[locale]` URL segment). Server reads from the `x-next-intl-locale` request header (set by next-intl's middleware from the URL, falls back to `routing.defaultLocale`). Same shape as before — both return the compound routing locale (`'me-cnr'`, `'rs-sr'`, etc.).
- Changed the formatter locale at the next-intl boundary. `src/i18n/request.ts` now derives `oglasinoLocale` from the compound and returns it as the config `locale` (bare: `'sr' | 'en' | 'ru' | 'cnr'`). `app/[locale]/layout.tsx` derives the bare locale from `params.locale` and passes it to `<NextIntlClientProvider locale={oglasinoLocale}>`. `setRequestLocale(locale)` keeps the URL-shaped compound so next-intl's request scope (used for static rendering and the URL ↔ identity link) is unaffected.
- Migrated 13 client `useLocale()` consumers and 9 server `getLocale()` consumers (covering all 22 callers that derive URL paths, parse via `getTenantLocale`, build cross-tenant redirects, or compose backend `X-Lang` / `X-Base-Site` headers) to the new primitives. Left 4 sites on next-intl's `useLocale()` / `getLocale()` — they consume the formatter locale (`Intl.DateTimeFormat`, `Intl.RelativeTimeFormat`, `toLocaleString`) and need a valid Unicode locale identifier, not the compound.
- `AccountStateDialogsInit.tsx` is the only consumer that needed both: it uses the routing locale for `router.replace(\`/${routingLocale}\`)` on ban-dialog dismissal and the formatter locale for `toLocaleDateString` on the deletion-date dialog. Split into two variables (`routingLocale` and `formatterLocale`). Tightened the `formatDeletionDate` Montenegrin fallback from `locale === 'me' || locale === 'cnr'` to `locale === 'cnr'` — under the new convention the bare locale is `'cnr'`, never `'me'`, so the `'me'` branch was dead.

## Files touched

New:
- `src/i18n/useRoutingLocale.ts` (+11 / -0)
- `src/i18n/getRoutingLocale.ts` (+13 / -0)

Modified — next-intl boundary:
- `src/i18n/request.ts` (+8 / -4)
- `app/[locale]/layout.tsx` (+4 / -1)

Modified — client consumers migrated to `useRoutingLocale()`:
- `src/components/owner/client/BackToPortalButton.tsx` (+2 / -2)
- `src/components/server/OglasinoBreadcrumbs.tsx` (+3 / -2)
- `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx` (+3 / -2)
- `src/components/popups/dialogs/DashboardProductFunctionsDialog.tsx` (+3 / -2)
- `src/components/popups/components/UploadedProductDialog.tsx` (+3 / -2)
- `src/components/popups/dialogs/PortalConfigDialog.tsx` (+4 / -3)
- `src/components/client/UserDetails.tsx` (+3 / -2)
- `src/components/client/ProductFunctions.tsx` (+3 / -2)
- `src/components/client/SessionGuard.tsx` (+2 / -2)
- `src/components/client/buttons/AuthUserProfileButton.tsx` (+3 / -2)
- `src/components/client/initializers/AccountStateDialogsInit.tsx` (+10 / -5)
- `src/components/client/initializers/AppInit.tsx` (+2 / -2)
- `src/components/client/initializers/FilterManager.tsx` (+3 / -2)

Modified — server consumers migrated to `getRoutingLocale()`:
- `src/lib/service/nextCalls/userService.ts` (+2 / -2)
- `src/lib/service/nextCalls/productsSearchService.ts` (+3 / -3)
- `src/lib/service/nextCalls/favoritesService.ts` (+3 / -3)
- `src/lib/service/nextCalls/metadataSnapshotService.ts` (+2 / -2)
- `app/[locale]/(portal)/(protected)/layout.tsx` (+2 / -2)
- `app/[locale]/(portal)/(public)/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/privacy/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/about/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/terms/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/pricing/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` (+3 / -3)
- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` (+3 / -3)
- `app/actions/getBaseSiteServer.ts` (+2 / -2)

Not touched (per brief Step 4): `app/layout.tsx` — `<html lang={...}>` already uses `getTenantLocale(locale).locale.split('-')[0]` from the `x-next-intl-locale` header, which is the URL-derived compound — unaffected by the new convention. Deliberately left as-is.

Left on next-intl's `useLocale()` (formatter, bare) — these pass the locale to `Intl.*`:
- `src/components/admin/chats/Chat.tsx` — `toLocaleString(locale)`
- `src/components/admin/es/EsReindexPanel.tsx` — `toLocaleString`, `Intl.RelativeTimeFormat`
- `src/components/admin/es/EsClusterStatePanel.tsx` — same
- `src/components/client/initializers/AccountStateDialogsInit.tsx` — formatter only (the routing-locale path uses `useRoutingLocale()`)

## Tests

- Ran: `npx tsc --noEmit`
- Result: clean (exit 0)
- Ran: `npm run lint`
- Result: 0 errors, 175 warnings (no change in count vs. pre-session baseline — verified my new files contribute zero warnings)
- Ran: `npm test`
- Result: 17 files, 206 tests passing
- Ran: `npm run format:check`
- Result: clean
- No new tests added: the migration is mechanical and covered by the existing locale-derivation tests; the bug itself is observable only on `me-cnr` URL visits, which the test suite does not exercise end-to-end (acknowledged in Brief 3c).

## Cleanup performed

- The conflation of routing locale and formatter locale is now structurally impossible to recur. Every call site goes through one of two named primitives whose return shape and meaning are documented at the source. A future engineer reading `useRoutingLocale()` or `getRoutingLocale()` cannot mistake one for the other.
- Removed the dead `'me'` branch in `AccountStateDialogsInit.tsx`'s `formatDeletionDate` Montenegrin fallback. Under the new convention `useLocale()` returns `'cnr'`, never `'me'` — the `'me'` arm was unreachable.
- Removed the stale `// ✅ Use dynamic loader instead of hardcoding all namespaces` checkmark-prefix comment from `src/i18n/request.ts` (replaced with the WHY comment for the new bare-formatter-locale return).
- No commented-out code, no unused imports, no ad-hoc debug logging, no new TODO/FIXME entries.

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change. The closing decision (Option B chosen, ~38-site migration shipped) is Mastermind's draft at chat close; not engineer-drafted text.
- `state.md`: no change. The cookies-closing feature is still in-progress on Item 4; a status flip belongs at feature close, not per-brief.
- `issues.md`: no change. The `INVALID_MESSAGE` bug had no `issues.md` entry to flip — Brief 3c routed the entry as Mastermind-discretion text and Mastermind chose to action the bug directly via this brief instead of logging it. The medium-severity `formatDeletionDate` dead `'me'` branch was fixed in passing (in-scope cleanup); no separate entry needed.

## Obsoleted by this session

- `useLocale()` as a routing-locale source in client components. After this session, every site that uses `useLocale()` directly intends the formatter locale (and there are exactly 4 such sites, all consuming `Intl.*` APIs). New routing-locale consumers go through `useRoutingLocale()`.
- `getLocale()` as a routing-locale source in server code. No remaining server consumer reads it for routing identity; the only `getLocale()` references left in the codebase are inside the new primitive files' header comments documenting the contract.
- The `'me'` Montenegrin branch in `AccountStateDialogsInit.tsx`'s `formatDeletionDate` — removed (was unreachable under both old and new conventions; the old convention saw `'me-cnr'`, the new sees `'cnr'`, neither saw bare `'me'`).
- The `// ✅` checkmark-prefix style comment in `src/i18n/request.ts` — replaced with substantive WHY comment.

## Conventions check

- Part 4 (cleanliness): confirmed. All edits leave no commented-out code, dead branches, or stale imports; the `'me'` branch and the checkmark comment are deleted; no new TODO/FIXME entries.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): three flagged below.
- Part 6 (translations): N/A this session — no key changes.
- Part 11 (trust boundaries): confirmed. No trust boundary touched. Both the routing locale (URL-derived, server-mediated) and the formatter locale (config-derived) are display-side; nothing in this brief routes a client-supplied value into a moderation/authorization/state-transition decision.
- Part 5 (closure gate): no implicit config-file dependency from this session. Drafts listed in "For Mastermind" are optional.

## Known gaps / TODOs

- None. The brief's full scope is implemented; all four automated checks green; the bug is structurally impossible to recur. Manual verification on a running dev server is the next gate (Igor's runbook, brief's "Manual" section).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - `useRoutingLocale()` — earns its place because Option B explicitly requires separating routing identity from formatter identity, and the alternative (calling `useParams()` directly at every of 13 client call sites) creates 13 places to remember "by the way, the URL segment is the compound, not what `useLocale()` returns." A named wrapper is the one place to encode that decision.
    - `getRoutingLocale()` — same justification for the 9 server call sites. The body is non-trivial (read header, fall back to default, type-narrow against `routing.locales`); inlining would duplicate ~6 lines × 9 sites.
    - The bare-locale derivation in `request.ts` (one call to `getTenantLocale`, return `oglasinoLocale` as `locale`) — load-bearing for fixing the bug.
  - **Considered and rejected:**
    - **Option A** (formatter locale = BCP-47 `cnr-ME` instead of bare `cnr`) — rejected. Bare is sufficient for Intl resolution on `cnr`/`sr`/`en`/`ru`, requires no inverse-mapping table, and matches the existing `oglasinoLocale` field that `getTenantLocale` already exposes.
    - **Option C** (rename `routing.locales` to BCP-47 compound `cnr-ME` etc.) — rejected. Out of proportion: changes every URL, every SEO entry, every external link. Mastermind already vetoed.
    - **Option D** (monkey-patch `Intl.Locale` or fork `intl-messageformat`) — rejected. Mastermind already vetoed; fragile.
    - **Per-page parameter threading** (server-side: pages pass `params.locale` to every function that needs it instead of a `getRoutingLocale()` helper) — rejected for the 4 `nextCalls/*Service.ts` files: they're called from multiple call sites (page render, generateMetadata, the cross-tenant SSR redirect paths), threading the parameter through every caller would force a wider refactor. Helper is cleaner. The pages themselves could use either pattern; I chose helper consistency.
    - **A unified `useLocaleSplit() → { routingLocale, formatterLocale }` hook** — rejected. The two values have different consumers; bundling them as a tuple forces every caller to discard one. Two named primitives are cheaper to call and clearer to read.
  - **Simplified or removed:**
    - The implicit conflation of routing-identity and formatter-identity through one `useLocale()` / `getLocale()` return value, replaced with two named primitives whose contracts are stated at the source.
    - Dead `'me'` branch in `AccountStateDialogsInit.tsx#formatDeletionDate` (one line).
    - Stale checkmark-style comment in `src/i18n/request.ts` (one line, replaced with WHY comment).

- **Part 4b adjacent observations.**
  - **`app/[locale]/(portal)/(protected)/layout.tsx` calls `pathnameWithoutLocale(locale)` instead of `pathnameWithoutLocale(pathname)`.** Severity: low. `pathnameWithoutLocale` expects a pathname; the layout passes a locale string (e.g. `'me-cnr'`). On a locale string, `split('/').filter(Boolean)` yields `['me-cnr']`, then `slice(1).join('/')` is `''`, then `'/' + ''` is `'/'` — which never matches `/favicon.ico`. The check is dead code as written. The layout's intent was probably "if the request is for `/favicon.ico`, skip auth," but the check never fires. Out of scope (separate bug-fixer brief — affects favicon under protected routes, not blocking).
  - **`app/[locale]/(portal)/(public)/privacy/page.tsx:14` has a `// TODO Create locale privacy.md`.** Pre-existing (not added by this session). Severity: low. The Markdown viewer hardcodes `privacy-policy.en.md` from GitHub — same content in every locale until per-locale Markdown files are created. Out of scope for this brief; preserved as-is during the migration.
  - **`generatePrivacyPageMetadata.ts` and `generateTermsPageMetadata.ts` pass `locale` (now the routing locale: `'me-cnr'`) as `inLanguage` in the JSON-LD structured data.** Severity: low. JSON-LD `inLanguage` expects BCP-47; `me-cnr` is not BCP-47. SEO consumers either ignore the field, fall back, or accept it leniently — but the value is technically wrong. Same issue would have existed before this brief (just with the same wrong value); the migration doesn't worsen it. Worth a one-line fix in those metadata generators (`getTenantLocale(locale).locale` to get BCP-47 `cnr-ME`) when next touched. Out of scope.
  - **The `getPathname` export from `src/i18n/navigation.ts` is still exported and unused** (carried from Brief 3 / Brief 3b / Brief 3c session summaries). Low. Not actioned this session.
  - **`useLanguageStore.lang` still has no readers** (carried from Brief 3b / Brief 3c). Low. Mastermind queued this as a separate brief (cookies-closing follow-up Obs 1 per Brief 3d's "Out of scope" list).

- **Drafted config-file text.**
  - No mandatory drafts. The closing entry to `decisions.md` (Option B chosen and shipped) is Mastermind's text to draft at chat close, not engineer-drafted. No `issues.md` entry to draft — the bug is fixed, not deferred.

Brief 3d complete.
