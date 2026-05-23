# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-21
**Task:** Fix the portal home RSC re-render double-fetch in FilterManager (levers A + B).

## Implemented

- **Lever A (trailing-slash fix).** Normalized the `pathname === '/'` case so the constructed `newUrl` no longer carries a trailing slash on the no-filters portal home. The gate at `FilterManager.tsx:315` (originally `:315`, now `:316`) was firing on every refresh of `/rs-sr` because `newUrl` was `/rs-sr/` while `current` was `/rs-sr`. After the fix, `newUrl` is `/rs-sr` on the home, the gate evaluates to `false`, and no `router.replace` runs on a no-filter refresh.
- **Lever B (drop the queued router.refresh).** Deleted the `setTimeout(() => router.refresh(), 0)` line that produced a second RSC refetch immediately after every `router.replace`. In Next 16, `router.replace` to a new pathname already refetches the server tree; the `setTimeout` was a leftover from an older Next version where `router.replace` did not consistently refetch.
- **Other FilterManager-mounted routes scanned.** FilterManager mounts in three layouts (`(portal)/(public)/layout.tsx`, `owner/layout.tsx`, `admin/layout.tsx`) via `SelectableFilterManagerWrapper`. All other routes — `/owner/products`, `/admin/products`, `/admin/products/[userId]`, `/catalog/...` — produce a non-`/` `pathname` from next-intl's `usePathname`, so `newUrl` never picked up the trailing slash on those routes in the first place. Lever A is a no-op for them but the normalization is safe and route-agnostic. Lever B applies everywhere FilterManager fires legitimately (any real filter change), eliminating the structural double-fetch.

## Files touched

- `src/components/client/initializers/FilterManager.tsx` (+2 / -2)

(The diff is one comparison-target line replaced and the `setTimeout` line removed; the new `pathSegment` constant adds one line.)

## Tests

- Ran: `npx tsc --noEmit` — clean (no output).
- Ran: `npm run lint` — 0 errors, 175 pre-existing warnings. No new warnings introduced; the two `react-hooks/exhaustive-deps` warnings on `FilterManager.tsx:223` and `:319` are pre-existing on the dependency arrays I did not touch.
- Ran: `npm test -- --run` — **206 passed (206)** across 17 test files. No regressions.
- Manual verification (engineer drove dev server, Igor drove backend access log):
  - **Hard refresh of `/rs-sr`:** **2** `POST /api/public/product/search` (was 4). Matches brief criterion 1.
  - **Normal refresh of `/rs-sr`:** **1** `POST /api/public/product/search` (was 3). Matches brief criterion 2.
  - **Browser console:** clean on refresh. No routing warnings. Confirmed by Igor.
  - **Filter interaction still works:** confirmed by Igor that applying a filter on `/rs-sr` updates the URL and re-renders the product list as before. Lever A's normalization does not break real filter changes. Matches brief criterion 4.
  - **Hard refresh of `/rs-sr/owner/products`:** 1 `POST /api/secure/products` (route's owner-scoped endpoint, not the public search endpoint). No double-POST shape. Confirms the trailing-slash false-positive never affected non-home FilterManager routes and Lever A introduced no regression. Matches brief criterion 5 in spirit (no regression on at least one other FilterManager-mounted route).
- No new tests added: the behavior being fixed is a route-level RSC re-render count that has no existing unit-test surface in this repo, and the manual verification fully exercised both regression-risk routes.

## Cleanup performed

- The `setTimeout(() => router.refresh(), 0)` line was removed in Lever B (also covered as cleanup because the redundancy was structural — that line had been a no-op-equivalent for the actual filter-change scenarios since the Next 15 upgrade, contributing only to the double-fetch on every legitimate trigger).
- No commented-out code, no debug logging, no TODOs added.

## Config-file impact

- `conventions.md`: no change
- `decisions.md`: no change
- `state.md`: no change
- `issues.md`: no change

## Obsoleted by this session

- The two-call-site mapping in `.agent/2026-05-21-oglasino-web-portal-multi-search-investigation-2.md` (the reconciliation tables for hard refresh and normal refresh) — the "post-mount RSC re-renders #2 and #3 of `Home`" rows are now structurally unreachable on the portal home; the investigation summary remains a correct historical record of the pre-fix behavior. Not deleted (it is read-only audit material, kept for context).
- Adjacent observation #5 from `.agent/2026-05-21-oglasino-web-portal-multi-search-investigation-1.md` ("`router.replace + setTimeout(0) → router.refresh` double-fetches in Next 16") — closed by Lever B. Surfaced as an observation in the audit; resolved structurally in this session.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports, no debug logging, no TODOs.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): confirmed — two adjacent observations surfaced; see "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 3 (engineer agent hard rules — stayed on `dev`, no commits, no pushes, no cross-repo edits, no writes to the four config files). Part 9 (stack reference — confirmed next-intl's `usePathname` strips the locale, validating Lever A's coverage analysis).

## Known gaps / TODOs

- None. Both levers applied, scan complete, tests + manual verification green.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** one new constant `pathSegment` inside the SYNC TO URL effect (`FilterManager.tsx:311`). It earns its place because the conditional needs to be applied at the construction site (per the brief's preferred approach — keeping the canonical URL form free of trailing slashes), and inlining the ternary in the template literal would have made the line harder to read. One line of localized state, no abstraction layer introduced.
- **Considered and rejected:**
  - A `stripTrailingSlash` helper used on both sides of the `!==` comparison. Rejected: it would have left the constructed `newUrl` carrying a trailing slash on the home (so `router.replace('/rs-sr/')` would still run on legitimate filter changes from the home), which is a worse canonical-URL form than the no-trailing-slash version. The brief explicitly listed this approach and the chosen approach as equally valid; I picked the one that produces the cleaner URL.
  - Building a generic URL-canonicalization helper to handle future trailing-slash cases anywhere in the app. Rejected: only one call site has this problem (FilterManager's home-route construction), and a helper for one caller is premature.
  - Wrapping the SYNC TO URL effect's URL construction in a `useMemo` to avoid rebuilding `newUrl` on every effect run. Rejected: the construction is shallow string work, the effect already has a guard (`if (newUrl !== current)`), and the prior audit established the dep churn (not the URL build cost) is the firing trigger.
- **Simplified or removed:**
  - Deleted the `setTimeout(() => router.refresh(), 0)` line — one fewer scheduled callback per filter-change trigger, one fewer RSC refetch, structurally simpler.
  - The `if (newUrl !== current)` block went from two statements (replace + setTimeout) to one (replace).

### Adjacent observations surfaced during the session (Part 4b)

1. **`isAllowedPath` is recreated on every render and not memoized.** `FilterManager.tsx:52-70`. Severity: low (cosmetic). It is referenced inside both `useEffect` dependency arrays (warned by lint as `react-hooks/exhaustive-deps`), but the existing code intentionally omits it from the deps array. The current pattern works because the function closes over `pathname` (already in deps) and uses no other dynamic state. This is one of the two pre-existing lint warnings on the file. Out of scope; flagging as cosmetic.
2. **Pre-existing typo in the destructured store variable name.** `FilterManager.tsx:40` — `selectedModerationStates: selectedModeratioStates` (the local alias drops the `n`). Used at `:307-308` and in the deps array at `:328` consistently, so it works, but the alias name is a typo. Severity: low (cosmetic). Out of scope.

### Verdict on the brief's expectation about other routes

The brief asked me to confirm whether `/admin/products`, `/admin/products/[userId]`, `/owner/products`, etc. had the same trailing-slash false-positive shape. **They do not.** The trailing-slash bug was unique to the home route because only the home produces `pathname === '/'` from next-intl's locale-stripping `usePathname`. All other FilterManager-mounted routes have a non-`/` `pathname` (e.g., `/owner/products`), so `newUrl` never picked up the bad trailing slash on those routes. The owner-products hard-refresh check confirmed: 1 endpoint call, no FilterManager-triggered refetch, no regression.

Lever A's normalization is route-agnostic and safe (the ternary just falls through to the `pathname` value for any non-`/` case), and Lever B applies to every legitimate FilterManager trigger anywhere it mounts. No follow-up brief is needed for other routes.

### Does the random-seed jerk get fixed incidentally?

The brief asked me to note in "For Mastermind" whether the random-seed jerk surfaces as fixed-by-side-effect. **Likely yes for hard refresh; harder to call for normal refresh.** Before this fix:
- Hard refresh produced 3 `Home` SSR renders (#1, #2 from `router.replace`, #3 from `setTimeout(router.refresh)`). Each render generated a fresh random seed → up to 3 visually-distinct product orderings, of which the user saw the last two swap in fast succession (= the visible jerk).
- After this fix, hard refresh produces 1 `Home` SSR render (the initial). No further re-renders → no second random seed → no swap.

That covers the jerk on the canonical home URL. But Mastermind's separate-brief framing of the random-seed jerk anticipated a single re-render still happening on real filter changes — and after this fix, real filter changes still cause one re-render (the `router.replace` from FilterManager when the URL actually changes). If a user lands on the home and a real filter change triggers ONE re-render, the random seed differs between the SSR call and the post-replace call, and the user sees one product-order swap. So the seed-key fix is still warranted as a separate brief; this session does not obsolete it. It just makes the symptom much rarer on the canonical home (only when an actual filter change happens).

### Drafted config-file text

- None drafted this session. No `conventions.md`, `decisions.md`, `state.md`, or `issues.md` edits required. The fix is a localized bug fix; no convention or architectural decision was introduced or revised, no state flip is needed (the `Active feature` table does not list this work as a tracked feature — it's a follow-up surfaced by the two prior portal-multi-search investigations), and no `issues.md` entry was promised by the brief.

### Questions / next steps

- The random-seed-driven jerk on legitimate filter changes is still the right next brief (its symptom is now much rarer but not eliminated).
- The `UseTokenRefresh` duplicate firebase-sync + push-token POSTs (out of scope per the brief) remain visible in both verification log captures — same shape as before. Worth confirming the brief sequence for that is queued.
- The `/favorites` icon flicker (also out of scope) is unaffected by this fix.
