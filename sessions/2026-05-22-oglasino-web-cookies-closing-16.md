# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-22
**Task:** Audit brief — language routing surface (post-Brief-3d). Read-only inventory across `i18n/navigation.ts`, all next-intl navigation consumers, plain `<a>`/`next/link` usage, `proxy.ts`, PortalConfigDialog's tenant switcher, `useLanguageStore`, and other Brief-3d ripples. Output `.agent/audit-language-routing.md`.

## Implemented

- Read `src/i18n/navigation.ts`, `src/i18n/routing.ts`, `src/i18n/request.ts`, `src/i18n/getRoutingLocale.ts`, `src/i18n/useRoutingLocale.ts`, `src/i18n/getLocalizedPath.ts`, `src/i18n/internalRequest.ts`, and the relevant next-intl v4.12.0 source (`navigation/react-client/createNavigation.js`, `navigation/shared/createSharedNavigationFns.js`, `navigation/react-client/useBasePathname.js`, `middleware/middleware.js`, `routing/config.js`, `shared/utils.js`) to trace `<Link>` / `useRouter` / `usePathname` mechanics post-Brief-3d.
- Grepped every consumer of `@/src/i18n/navigation` (18 import sites: 1 self, 17 consumers — 16 `<Link>`, 1 `useRouter`, 7 `usePathname`, 0 `redirect`, 0 `getPathname`). Inventoried each call site's href shape and disposition.
- Audited plain `<a>` (one occurrence, hash-only) and `next/link` (three occurrences) usage.
- Walked `proxy.ts` end-to-end and traced the headline `/rs-sr/sr/catalog/men` mangling step by step (client `<Link>` emits `/sr/catalog/men` → proxy regex misses → next-intl middleware redirects to `defaultLocale` prefix → 404).
- Inspected `PortalConfigDialog.navigate` and the cross-tenant guard in `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` to capture the silent-snap-back behavior on tenant change.
- Confirmed `useLanguageStore.lang` has zero readers; sole call site is `PortalConfigDialog.setLang`.
- Verified `X-Lang` / `X-Base-Site` headers in `api.ts` and all server-side service files still consume the compound via `useRoutingLocale()` / `getRoutingLocale()`. Verified all `` /${locale} `` URL-interpolation sites (11 of them) source the compound correctly.
- Surfaced `usePathname()` from next-intl as a silent regression vector: five predicate branches (FilterManager home-gate, FavoriteButton favorites-refresh, ProductList favorites-reload, ChatsWatcher temp-state lifecycle, SearchInput portal-scope) now hold the wrong invariant.
- Wrote `.agent/audit-language-routing.md` with all 7 areas + 11 surprises and adjacent observations.

## Files touched

- `.agent/audit-language-routing.md` (new file, +320 / -0)
- `.agent/2026-05-22-oglasino-web-cookies-closing-16.md` (this file, new)
- `.agent/last-session.md` (overwritten with this file's content per Part 5)

No source-code changes. Read-only audit per brief.

## Tests

- Not run. Audit is read-only; no code changed. `npm run lint`, `npx tsc --noEmit`, and `npm test` were not invoked because Part 4 cleanliness applies to "touched paths" and no source path was touched.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change — no new bug entries authored. The audit document is the deliverable; Mastermind will derive `issues.md` entries from it if needed.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed. No source changes.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): see "For Mastermind" — adjacent observations are inside the audit document itself (the "Surprises and adjacent observations" section) rather than duplicated here, because the brief asked for them there.
- Part 6 (translations): N/A this session — no translation work.
- Other parts touched: Part 3 hard rules (no commits, no pushes, no cross-repo, no four-config writes) — confirmed.

## Known gaps / TODOs

- The audit does not propose fixes (brief: "Do not propose fixes"). The "what's broken and where" inventory is exhaustive to the limits of static analysis; the "what to do about it" comes from Mastermind.
- The audit does not include runtime reproduction (brief: "Do not start a dev server. Static investigation only"). The trace through next-intl source confirms the URL-generation mechanic mechanically; if Mastermind wants a live capture of the request chain (proxy ⟶ middleware ⟶ 307 ⟶ redirected request), that's a separate runtime task.
- `ProductBreadcrumbs.tsx:3` imports `usePathname` from `@/src/i18n/navigation` but the import is not visibly used in the file's rendered output (per grep). Flagged in the audit as a leftover-import suspect; not confirmed by full read.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing. Audit-only session.
  - Considered and rejected: I considered running the app to reproduce the bug with browser-bar URLs in hand. Rejected — brief explicitly says "Do not start a dev server. Static investigation only." The next-intl source trace produces equivalent mechanical certainty.
  - Simplified or removed: nothing. Read-only session.

- **Brief vs reality:** none. The brief's central hypothesis (next-intl `<Link>` produces double-prefixed URLs after Brief 3d) is confirmed by the next-intl source. The brief's expected behavior of the `proxy.ts` new branch (it does *not* fire on the bad URLs) is confirmed by branch-tracing. No discrepancies between brief and disk worth stopping for.

- **Audit document location:** `.agent/audit-language-routing.md`. Structure follows the brief's outline (7 areas + a closing "Surprises and adjacent observations"). 11 items in the closing section.

- **Most load-bearing finding:** the proxy.ts new invalid-locale branch (Brief 3e-bis) cannot reach the bug. The malformed URL is generated client-side by next-intl's `<Link>` from a bare `useLocale()`; the proxy sees only the already-emitted `/sr/catalog/men` (which its regex doesn't match) or the already-redirected `/rs-sr/sr/catalog/men` (which its regex treats as a valid `rs-sr` combo with extra junk). Any fix has to address Link/router URL generation directly, not the proxy.

- **Second load-bearing finding:** `usePathname()` from `@/src/i18n/navigation` is also broken post-Brief-3d and silently breaks five separate predicate branches in the codebase, two of which have explicit decisions.md guarantees against them (FilterManager home gate from 2026-05-21; ChatsWatcher temp-state lifecycle from 2026-05-20). The Link/router fix and the `usePathname` fix may be the same surgical change (replace next-intl's bare-locale-keyed prefix logic with a wrapper that uses the compound from `useRoutingLocale()`), or they may need separate treatment. That decision is Mastermind's.

- **Adjacent finding from Area 7.8, out of audit scope:** plain `useRouter` from `next/navigation` is used in eight buttons (`AuthChatButton`, `AuthFavoritesButton`, `AuthNotificationButton`, `StartMessageButton`, `UserDetails`, `FavoriteButton` via `window.location.href`, two more in `UserDetails`) with unprefixed paths like `router.push('/messages')`. These have been silently teleporting non-default-locale users to `rs-sr` since well before Brief 3d. Not introduced by Brief 3d; flagged for completeness because the cleanup brief that fixes Brief 3d's regression may want to fold this in.

- **Possible Brief 3f scope expansion:** `useLanguageStore.lang` has zero readers. The store reduces to a side-effecting helper (the cookie write inside `setLang`). The Brief 3f collapse can be a one-line inline of `updateGlobalCookie('lang', lang)` at the PortalConfigDialog call site (consent-gating preserved) plus deletion of `src/lib/store/useLanguage.ts`. Mastermind's call on whether to scope the collapse into the next routing-fix brief or keep it separate.

- **No config-file drafts** produced this session.
