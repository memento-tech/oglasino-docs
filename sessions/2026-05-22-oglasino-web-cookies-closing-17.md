# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-22
**Task:** Brief 3f — wrap createNavigation + migrate plain useRouter sites

## Implemented

- Replaced `src/i18n/navigation.ts` with a routing-locale-aware wrapper module. `Link`, `useRouter`, `usePathname` now source the compound routing locale from `useRoutingLocale()` instead of next-intl's `useLocale()` (which post-Brief-3d returns the bare formatter locale). `<Link href="/catalog/men">` on `/rs-sr/...` now emits `<a href="/rs-sr/catalog/men">`; `router.push('/admin/products')` navigates to `/rs-sr/admin/products`; `usePathname()` returns the unprefixed path so the five equality / `startsWith` predicates the audit Area 2 found broken (FavoriteButton refresh, ProductList refetch, ChatsWatcher cleanup, SearchInput portal-scope branch, FilterManager root gate) come back into alignment.
- Created `src/i18n/navigation-client.tsx` (`'use client'`) containing the three client primitives, hand-rolled over `next/link` and `next/navigation` (Shape A). `src/i18n/navigation.ts` re-exports those and provides sync `getPathname` / `redirect` helpers that take an explicit `locale` arg (no callers in repo today; kept for API parity).
- Migrated the six plain-`useRouter`-from-`next/navigation` sites the audit Area 7.8 enumerated to import `useRouter` from `@/src/i18n/navigation`. `router.push('/messages')` / `'/favorites'` / `'/notifications'` / `'/user/<id>'` calls now prefix to the user's actual compound locale instead of redirecting through next-intl middleware to default `rs-sr`. `FavoriteButton`'s `window.location.href = '/favorites'` was swapped for `router.push('/favorites')` per Brief Step 4.
- Verification: `npx tsc --noEmit` clean, `npm run lint` 0 errors / 175 warnings (Brief-3e-bis baseline unchanged), `npm test` 206/206 passing, `npm run format:check` clean.

## Files touched

- src/i18n/navigation.ts (rewrite: 1 → 19 lines)
- src/i18n/navigation-client.tsx (new, 88 lines)
- src/components/client/buttons/AuthChatButton.tsx (+1 / -1)
- src/components/client/buttons/AuthFavoritesButton.tsx (+1 / -1)
- src/components/client/buttons/AuthNotificationButton.tsx (+1 / -1)
- src/components/client/StartMessageButton.tsx (+1 / -1)
- src/components/client/UserDetails.tsx (+1 / -1)
- src/components/client/buttons/FavoriteButton.tsx (+2 / -3)

## Tests

- Ran: `npx tsc --noEmit` → clean
- Ran: `npm run lint` → 0 errors, 175 warnings (matches Brief-3e-bis baseline)
- Ran: `npm test` → 17 files / 206 tests passed
- Ran: `npm run format:check` → clean
- New tests added: none (the wrapper's contract is exercised through every existing consumer test; no unit-test-only surface).

Manual scenarios from the brief (1–10) were not executed in this session — code-on-disk only, awaiting Igor's manual smoke. Audit Area 2's static walk was reverified file-by-file as part of Step 3 (see "Implemented" above).

## Cleanup performed

- None needed. The wrapper is a like-for-like replacement at the import boundary; no commented-out code, no obsolete helpers, no debug logging introduced.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- The previous `src/i18n/navigation.ts` (6 lines, `createNavigation(routing)` re-export) is replaced. Deleted in this session; behavior captured in the new wrapper.
- `next-intl/navigation` is no longer imported anywhere in the repo (grep across `src/` + `app/` confirms zero remaining call sites). The dependency itself stays — next-intl is still load-bearing for `useTranslations`, the `<NextIntlClientProvider>`, the middleware, etc.

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one set flagged in "For Mastermind."
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): confirmed. The compound vs bare split survives intact: URL construction reads compound from `useRoutingLocale()` / `getRoutingLocale()` (the audit-blessed primitive), `Intl.*` consumers continue to read bare from `useLocale()` / `getLocale()`. No trust-boundary surface touched.

## Known gaps / TODOs

- None added. The brief listed nine plain-`useRouter` sites with the same teleport-to-rs-sr pattern that are not in audit Area 7.8's enumeration — those are flagged in "For Mastermind" rather than fixed here, since the brief's Step 4 scope was the audit's named list.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - `src/i18n/navigation-client.tsx` — three client primitives implemented over `next/link` / `next/navigation`. The complexity is the unavoidable shim that lets the rest of the codebase keep its plain-string href ergonomics while the compound-vs-bare split holds. Earns its place: 18 import sites depend on this surface.
    - `WrappedRouter` type and `WrappedNavigateOptions` / `WrappedPrefetchOptions` extensions — required because the wrapper accepts an optional `locale` override on `push` / `replace` / `prefetch` for caller-controlled routing (parity with next-intl's prior API surface so future callers don't need to re-learn).
    - `getPathname` / `redirect` exports in `navigation.ts` — kept solely for API completeness. Zero callers today, per the brief. Two short sync helpers; if Mastermind prefers, they can be dropped in a follow-up — flagging the choice here.
  - **Considered and rejected:**
    - **Shape B (wrap next-intl's primitives with explicit `locale` prop).** Tracing `node_modules/next-intl/dist/esm/development/navigation/shared/createSharedNavigationFns.js:35-46` and `…/react-client/createNavigation.js:42-49` confirmed that `<Link locale={...}>` and `useRouter().push(href, {locale})` do override `useLocale()` via `forcePrefix: locale != null` → `applyPathnamePrefix` → `getLocalePrefix → '/' + locale`. So Shape B is viable for those two primitives. Rejected because (a) `usePathname` has no override hook (no way to swap `useLocale()` inside `useBasePathname`), so a custom hook would be needed regardless; (b) coupling our prefix logic to next-intl's internal `applyPathnamePrefix` / `forcePrefix` semantics adds a moving part across minor versions for no functional gain; (c) the codebase has zero object-form hrefs, zero `pathnames` config, and `mode: 'always'` is locked — string concatenation captures the full behavior in five lines. The diff difference between Shape A and Shape B is ~10 lines; the reasoning-surface difference is larger.
    - **Option 4 (compound at provider) — already vetoed upstream (Brief 3 / 3a discussion).** Not reconsidered.
    - **Reaching into `next-intl`'s `createSharedNavigationFns` to pass a custom `getLocale`.** The function is not part of next-intl's public exports — only the `createNavigation` factory is. Using the private path would create a build-fragile coupling. Rejected.
    - **A `LocaleAwareLink` component that wraps next-intl's Link and re-passes `locale` explicitly at every call site.** Would have required touching all 16 absolute-href `<Link>` sites instead of one wrapper module. The whole point of the wrapper is that consumer code doesn't change. Rejected.
    - **Splitting `navigation.ts` into separate `.client.ts` / `.server.ts` files at the public level.** The current split (one client file, one barrel) achieves the same client / server boundary without splitting the public surface. The barrel re-exports the client primitives transparently. Server callers of `getPathname` / `redirect` get plain functions (no `'use client'` boundary), client callers of `Link` / `useRouter` / `usePathname` get the client component / hook chain. Same shape, less import-site noise.
  - **Simplified or removed:**
    - The old `navigation.ts` (one-liner `createNavigation(routing)` re-export) is gone. The codebase no longer imports from `next-intl/navigation` at all (verified by grep) — that surface is replaced by ours.
- **Brief vs reality discrepancies (none load-bearing, but flagged):**
  - The brief Step 4 lists `src/components/client/buttons/StartMessageButton.tsx`; the file is actually at `src/components/client/StartMessageButton.tsx` (no `buttons/` subfolder). Same file, different path — implemented against the on-disk location.
  - The brief Step 4 says "8 plain `useRouter` sites the audit identified in Area 7.8." Audit Area 7.8 explicitly enumerates six files (counting AuthChatButton, AuthFavoritesButton, AuthNotificationButton, StartMessageButton, UserDetails, FavoriteButton). Counting call sites within those files yields nine; counting files yields six. I migrated all six.
- **Part 4b adjacent observations:**
  - **`router.push` to unprefixed paths in nine additional files not in audit Area 7.8's enumeration.** Same teleport-to-`rs-sr` mechanism the audit describes; the audit's "e.g." in front of its file list implicitly accepted there are more. Each is mechanically migrate-able by switching the `useRouter` import to `@/src/i18n/navigation`. Severity: medium (silent locale teleport for non-`rs-sr` users). Not fixed in this brief because the brief's Step 4 scope was the audit's named list; flagging for a possible cleanup brief. The files and lines:
    - `src/messages/components/Messages.tsx:187` — `router.push(\`/user/\${peer.id}\`)`
    - `src/components/owner/follows/UserCard.tsx:56` — `router.push('/user/' + userInfo.id)`
    - `src/components/server/NotFound.tsx:24` — `router.push('/')`
    - `src/components/popups/dialogs/DashboardProductFunctionsDialog.tsx:68` — `router.push('/owner/products/' + productOverview.id)`
    - `src/components/client/buttons/AuthUserProfileButton.tsx:65, :73, :88, :120` — four unprefixed pushes (line 83 builds its own compound and is correct as-is)
    - `src/components/client/initializers/ForegroundPushInit.tsx:47` — `router.push(url)` where `url` derives from a notification payload; needs a content audit to confirm whether unprefixed paths can flow through it
    - `src/components/client/product/AdminProductCard.tsx:29` — `router.push(\`/admin/products/product/\${productOverview.id}\`)`
    - `src/components/client/product/PortalProductCard.tsx:30` — `router.push(getNormalizedProductUrl(id, name))` (returns unprefixed `/product/<id>/<slug>`)
    - `app/[locale]/owner/products/[productId]/page.tsx:630` — `router.replace('/owner/products')`
  - **`AuthNotificationButton.tsx:16` passes `router` into `useNotificationsStore(router)`, which forwards to `notificationManager.showMany(notifications, router)`, which calls `router.push('/notifications')` inside an `onClick` (`src/notifications/lib/notificationManager.ts:58`).** With the migration, the wrapped router flows all the way through and the inner `router.push('/notifications')` prefixes correctly. No code change needed at the notification-manager level. (Side note: `useNotificationsStore` and `notificationManager` accept `router: any` — the `: any` annotation is one of the 175 baseline warnings, not introduced here. Severity: low.)
  - **`OglasinoBreadcrumbs.tsx` continues to live under `components/server/` while using client-only hooks** (`useScreenBreakpoint`, `useTranslations`, `useDialogStore`, `useRoutingLocale`). Pre-existing inconsistency noted by audit surprise #4; not Brief 3f's scope.
  - **`UserRow.tsx:51` `<Link href={\`users/${user.id}\`}>` (relative)** continues to be accidentally-correct under our wrapper because relative hrefs (no leading `/`) are passed through unmodified, exactly as next-intl's `isLocalizableHref` did. UserRow.tsx:56 immediately below it uses the absolute form. The single-character call-site difference flagged in audit surprise #5 is still latent.
  - **`useLanguageStore.lang` field has zero readers** (audit Area 6). Out of Brief 3f's scope (collapse is Brief 3h per the brief), restated here so it doesn't get lost.
- **API completeness note on `getPathname` / `redirect`.** No call sites today. Mastermind may want to either (a) keep them as I've implemented them (sync, explicit `locale` arg) for any future server-side redirect helpers, or (b) drop them entirely and require any future caller to use `next/navigation`'s `redirect` directly with a manually-prefixed URL. I left them in per the brief's "keep the exports for API completeness" guidance.

## Brief vs reality

I read the brief and the code. Two discrepancies (already noted above), neither load-bearing enough to halt the work:

1. **StartMessageButton.tsx path.** Brief Step 4 says `src/components/client/buttons/StartMessageButton.tsx`; actual path is `src/components/client/StartMessageButton.tsx`. The file with the same identifier exists; the brief's path was one folder off. Implemented against the on-disk path.
2. **"8 plain useRouter sites."** Brief Step 4 says "the 8 plain `useRouter` sites the audit identified in Area 7.8." Audit Area 7.8 names six files (nine call sites if counting individual `router.push` lines). The discrepancy doesn't change what I migrated — the brief explicitly enumerates the six file paths under "Sites per audit," and I migrated those six. The broader sweep across the additional nine files exhibiting the same pattern is flagged for Mastermind as a Part 4b adjacent observation rather than absorbed into this brief.
