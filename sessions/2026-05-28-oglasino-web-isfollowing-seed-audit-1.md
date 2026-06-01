# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** Audit `UserInfoDTO.isFollowingCurrent` seed: non-authoritative because `getUserForId` uses `skipAuth: true`. READ-ONLY; recommend fix shape.

## Implemented

- Read-only audit of the `isFollowingCurrent` data path, the `iamActive` recompute precedent, `FollowUserButton`'s seed behavior, the three candidate fixes from `issues.md` 2026-05-17, and the missing post-follow revalidation. No code changes.

## Files touched

- (none)

## Files read

- `src/lib/service/nextCalls/userService.ts` (current `getUserForId` lives at lines 7–32; the cited 13–19 range is the inner `FETCH_BACKEND_API.get` call)
- `src/lib/config/fetchApi.ts` (the `skipAuth` semantics are documented at lines 15–20 and implemented at lines 36–45)
- `src/components/client/UserDetails.tsx` (full file; `iamActive` recompute at 55–61, two `FollowUserButton` consumer sites at 122–125 and 136–139)
- `src/components/client/buttons/FollowUserButton.tsx` (the seed-from-prop initial state at line 28)
- `src/lib/service/reactCalls/followService.ts` (`markFollowUser`, lines 4–19)
- `src/lib/types/user/UserInfoDTO.ts` (viewer-dependent fields `iamActive`, `isFollowingCurrent`)
- `src/lib/service/reactCalls/userService.ts` (`getUserFollows`, `/secure/user/follows`; for follows-store discovery)
- `src/components/owner/follows/UserCard.tsx`, `app/[locale]/owner/follows/page.tsx` (sibling consumer of `markFollowUser`; row-removal pattern post-unfollow)
- `app/actions/revalidateUserCache.ts` (the existing `updateTag('user:${userId}')` Server Action — only invoked from `DeleteAccountConfirmationDialog` and `AccountStateDialogsInit`)
- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`, `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` (RSC call sites of `getUserForId`)
- Repo-wide grep: `markFollowUser`, `isFollowingCurrent`, `useFollow*`, `FollowsStore`, `revalidateTag`, `updateTag`

## Audit

### Section 1 — The data path

`getUserForId` (`src/lib/service/nextCalls/userService.ts:7-32`) is an RSC-side fetch wrapper that calls the backend at `/public/user?id=<id>` with three notable options:

- `headers: { 'X-Base-Site': tenant, 'X-Lang': lang }`
- `skipAuth: true` (line 15)
- `next: { revalidate: 300, tags: ['user', 'user:${userId}'] }` (lines 16–19)

The route uses the public catalog endpoint `/public/user`. Two RSC call sites consume it:

- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:32` (inside `generateMetadata`) and `:49` (inside `UserPage` render) — `/user/<id>` route.
- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:91` — the product detail page fetches the owner.

`skipAuth` semantics (`src/lib/config/fetchApi.ts:15-20, 36-45`): the inline doc-comment states `skipAuth: true` deliberately skips the `cookies()` call AND the Firebase Bearer header. The implementation (lines 36–45) gates the `cookies()` read on `!skipAuth && typeof window === 'undefined'`; without that, `firebaseToken` and `cookieHeader` are both unset, and the resulting `fetch` carries neither `Cookie:` nor `Authorization: Bearer ...`. So the backend cannot identify the viewer on this path. The doc-comment's rationale is the dynamic-rendering / data-cache concern: any `cookies()` read in a Server Component opts the route into dynamic rendering and disables the Data Cache.

Result: `UserInfoDTO.isFollowingCurrent` and `UserInfoDTO.iamActive` (`src/lib/types/user/UserInfoDTO.ts:13,15`) are both viewer-dependent but populated by an endpoint that has no viewer identity. Whatever value the backend defaults to is what flows through. Compounded by the 5-minute Data Cache `revalidate`, the value is also shared across viewers — any one viewer's request stores the value into the per-`userId` cache entry for the next ~5 minutes.

### Section 2 — The `iamActive` precedent

`src/components/client/UserDetails.tsx:55-61`:

```tsx
const iamActive = useMemo(() => {
  if (user) {
    return user.id === userDetails.id;
  }
  return false;
}, [user, userDetails.id]);
```

`user` comes from `useAuthStore()` (line 48). The pattern: ignore the seed entirely, recompute client-side against the authenticated user in Zustand. The `useMemo` deps are `[user, userDetails.id]`, so the value re-evaluates when auth hydrates or when the viewed user changes. `userDetails.iamActive` itself is never read in the file.

This is the closest existing template for `isFollowingCurrent`. The difference: `iamActive` is computable from a single boolean comparison against one ID the client already has in its Zustand store. `isFollowingCurrent` requires knowing the viewer's full follow set, which the client does not currently have anywhere.

### Section 3 — `FollowUserButton` seed behavior

`src/components/client/buttons/FollowUserButton.tsx:28`:

```tsx
const [following, setFollowing] = useState(isFollowing);
```

The prop is used **only** as the initial state of `useState`. There is no `useEffect` that resyncs `following` if `isFollowing` changes (the destructured `isFollowing` is read once at mount). There is no post-mount correction: no re-fetch, no store subscription, no listener. The two `FollowUserButton` consumer call sites are both in `UserDetails.tsx` (122–125 inside the `<h1>` branch when `isOnUserPage`, and 136–139 inside the `<div>` branch when not) and both pass `userDetails.isFollowingCurrent` directly.

After the user clicks, `markFollowUser(userId)` is called; on success the local `following` state is replaced by the server-returned `result.following` (line 40) — so subsequent renders inside the same mount reflect the real state. The bug is the initial click: if the seed is wrong, the user clicks the wrong icon and the server happily toggles.

No follows-store exists anywhere in the web codebase. Repo-wide grep for `useFollow*`, `FollowsStore`, `followStore`, `useFollowing*` returns zero matches. The only follow-related client data is `getUserFollows` (`src/lib/service/reactCalls/userService.ts:114`), a paginated POST `/secure/user/follows` used exclusively by the `/owner/follows` page to render the list (`app/[locale]/owner/follows/page.tsx`). It is not used as a global "do I follow X?" lookup; pagination + on-demand load is the wrong shape for a per-product-page check.

### Section 4 — The three candidate fixes

**A — Split `getUserForId` into public-skipAuth + authenticated paths.**

Feasibility: web-only mechanical split if the backend already exposes a matching `/secure/user?id=<id>` (or equivalent) that includes `isFollowingCurrent`. The repo's existing `/public/*` vs `/secure/*` split (e.g., `/secure/user/update`, `/secure/follow/<id>`, `/secure/user/follows`) is the precedent. Backend dependency unknown from this audit's scope — needs Mastermind confirmation with backend. If the backend endpoint exists, the split is small: `getUserForId` keeps the public-skipAuth path returning a DTO with viewer-dependent fields omitted (or marked `null`); a new `getUserForIdAuthed` runs from the client side via `BACKEND_API` after auth hydrates.

Blast radius: both RSC call sites (`user/[userId]/page.tsx`, `product/[productId]/[productName]/page.tsx`) must decide how to pass viewer-dependent state down to client components. Either (a) the RSC continues to use the public path for SSR-cacheable data and the client hydrates `isFollowingCurrent` separately via a small client call (a B/C hybrid in effect), or (b) the RSC switches to the authenticated path on these two routes, which forfeits the Data Cache hit per the `fetchApi.ts` doc-comment's rationale.

Cache implications: option (b) above defeats the 5-minute `revalidate` for any signed-in viewer — every page render becomes dynamic. For high-traffic public pages (user profile, product detail) that is a meaningful regression unless paired with a per-`(userId, viewerId)` cache key, which Next's Data Cache does not naturally provide.

**B — Drop `isFollowingCurrent` from the public endpoint; seed via a separate small auth-required call.**

Feasibility: requires backend change (remove or null the field from `/public/user`'s response — purely an API-shape cleanup, no behavior change). Web side then needs a new small endpoint such as `GET /secure/follow/status?userId=<id>` returning `{ following: boolean }`. The button (or a wrapping component) fetches it post-mount.

Blast radius: small. `FollowUserButton` gains an internal effect that fetches the status once on mount when `user` is authenticated; while pending, it shows nothing or a neutral state. UserDetails stops passing the seed. The button becomes self-sufficient — the change is contained to one component plus one new service function.

Cache implications: clean. The public endpoint stays fully cacheable; the per-button auth call is small and naturally per-viewer.

Trade-off vs A: B introduces an extra round-trip per FollowUserButton mount but keeps the public RSC fully cacheable. A keeps the data colocated in `UserInfoDTO` but is harder to make cacheable.

**C — Compute `isFollowingCurrent` client-side post-mount via a follows-store (mirrors `iamActive`).**

Feasibility: requires (i) a new Zustand `useFollowsStore` carrying the viewer's full follow set as a `Set<number>`, (ii) a new endpoint `GET /secure/user/follow-ids` returning the IDs (the existing `/secure/user/follows` is paginated and returns full DTOs — wrong shape), (iii) an initializer that loads the store on auth hydration, (iv) a Zustand subscription in `FollowUserButton` (or the surrounding `UserDetails`) for the lookup. Also requires keeping the store in sync after `markFollowUser` and on cross-tab events.

Blast radius: a new piece of cross-cutting state. Useful beyond this bug (any future "do I follow X?" surface gets it for free), but introduces a new fetch on every authenticated page load and a non-trivial sync surface.

Cache implications: best of the three. The public endpoint stays cacheable; the per-viewer follow set is one auth-only fetch, reusable across the whole session. Survives the 5-minute Data Cache staleness because the field is computed at runtime.

**Recommended fix shape:** B.

Reasoning: B has the smallest blast radius, keeps the `/public/user` endpoint cleanly viewer-independent (a long-term win for Data Cache hits), and avoids both A's cache regression and C's new cross-cutting state. C is structurally cleaner if a second "do I follow X?" surface lands, but no such surface exists today and Part 4a says abstractions earn their introduction. A's "split paths" framing pushes the problem onto the RSC layer where the Data Cache concern is sharpest. B contains the change to two components (`FollowUserButton` + one new service function) plus a backend DTO field removal — the lightest end-state.

Caveat: B presumes Mastermind agrees the round-trip cost is acceptable (one extra `GET` per FollowUserButton mount for authenticated users). If that proves contentious, C is the principled fallback.

### Section 5 — The missing revalidation

Confirmed. `markFollowUser` (`src/lib/service/reactCalls/followService.ts:4-19`) POSTs `/secure/follow/<userId>` and returns `{ ok, following }`. The function does not call `revalidateTag`, `updateTag`, `revalidateUserCache`, `router.refresh`, or any cache invalidation. Its two callers (`FollowUserButton.tsx:38`, `UserCard.tsx:26`) also do not invalidate. The only existing post-action cache buster for the `user:${userId}` tag is `app/actions/revalidateUserCache.ts:21`, which is invoked by `DeleteAccountConfirmationDialog` (account deletion) and `AccountStateDialogsInit` (account restoration). No follow path touches it.

Consequence: even if a viewer's seed is correct on first load (e.g., they follow X for the first time, server returns `following: true`, button renders correctly), navigating away from `/user/X` and back within 5 minutes shows a stale `UserInfoDTO` — the cached one from before the follow action. If the seed was already correct, the staleness silently re-introduces the bug.

**Does the staleness fix belong in the same brief?**

It depends on the chosen fix shape:

- **A or B:** yes, same brief. With A, the cached `/public/user` payload still carries (or omits) `isFollowingCurrent`, so any cached `UserInfoDTO` consumed alongside still needs invalidation if it contains viewer-dependent state. With B, the public payload becomes truly viewer-independent and the cache staleness is moot for `isFollowingCurrent` specifically — but the existing `user:${userId}` tag still needs a `markFollowUser`-triggered bust for any *other* viewer-dependent state that the public DTO might re-acquire later (and to make the `iamActive` recompute pattern more robust to data-shape drift). Cheap to add as a one-line Server Action call from `markFollowUser`'s success branch.
- **C:** no, separate brief or none at all. With a client-side follows-store, the cached `UserInfoDTO` no longer carries the viewer-dependent field and the staleness is irrelevant.

Net recommendation: bundle staleness into the same brief if B is chosen — it is two-to-three lines (`await revalidateUserCache(userId)` in `markFollowUser`'s success branch, mirroring `DeleteAccountConfirmationDialog`'s use). Defer if C is chosen.

## Tests

- Ran: (none — read-only audit)
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change drafted by this session; Mastermind decision on A/B/C and on bundling vs separating the staleness fix is the only config-file write this audit implies, and is drafted as a question in "For Mastermind"
- state.md: no change drafted by this session; if/when the fix brief is queued, an Active-features entry would land then
- issues.md: no change drafted by this session — the existing 2026-05-17 entry remains the canonical record. When the fix ships, Docs/QA flips that entry to `fixed` based on the eventual session summary

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code touched
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): two observations recorded in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 11 (trust boundaries) — informs the analysis (the backend cannot derive the viewer's identity on a `skipAuth: true` call, so any viewer-dependent field on that endpoint is structurally non-authoritative). No new trust-boundary violation introduced.

## Known gaps / TODOs

- The audit does not enumerate backend endpoints. Whether a `/secure/user?id=<id>` mirror of `/public/user` exists (which would make Fix A nearly trivial), and whether the backend would accept removing `isFollowingCurrent` from `/public/user` (Fix B), are explicit questions for Mastermind to pass to the backend engineer. Brief out-of-scope per the brief's "Out of scope" section.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit
  - Considered and rejected: nothing — no new abstractions, helpers, configs, or patterns proposed
  - Simplified or removed: nothing — no existing code touched

- **Recommended fix shape: B** (drop `isFollowingCurrent` from `/public/user`, seed the button via a small `GET /secure/follow/status?userId=<id>` call). Smallest blast radius, keeps the public endpoint cacheable, no new cross-cutting state. C is the principled fallback if you decide the round-trip cost is unacceptable.

- **Cross-repo question (for the backend engineer agent):**
  1. Does `/secure/user?id=<id>` (or an equivalent authenticated variant of `/public/user`) exist today? If yes, Fix A becomes a small web-only change; if no, Fix A requires backend work.
  2. Would the backend accept removing `isFollowingCurrent` from `UserInfoDTO` on the `/public/user` response (or returning it as `null`) and exposing a small `GET /secure/follow/status?userId=<id>` endpoint? This is Fix B's backend dependency.

- **Staleness fix scoping:** bundle with the main fix if you choose A or B (one-line `revalidateUserCache(userId)` call from `markFollowUser`'s success branch); defer if you choose C.

- **Drift note on issues.md 2026-05-17 entry line numbers (very low severity, no action needed):** the entry cites `userService.ts:13-19`, `UserDetails.tsx:53-59` (iamActive recompute), and `UserDetails.tsx:119` (single FollowUserButton consumer). Current code is at 7–32 (full function) / 13–19 (inner fetch call), 55–61, and 122–125 + 136–139 (two consumer sites — `UserInfoBlock` is rendered in both an `<h1>` and a `<div>` branch). The bug shape is unchanged; only the line numbers drifted slightly. The "single line 119" framing also undercounts: there are two consumer sites, both seeded from the same `userDetails.isFollowingCurrent` value, so the fix shape doesn't change but it is two touchpoints, not one.

- **Adjacent observation 1 (low):** `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:91` invokes `getUserForId` and assigns the result to `const owner: UserInfoDTO = ...`, but `getUserForId`'s signature returns `UserInfoDTO | null`. Line 93 then accesses `owner.iamActive` without a null check. If the public user fetch ever returns `null` (backend down, deleted owner, etc.), this is a runtime `TypeError`. Mirrors the pattern that was fixed in the `2026-05-22 — Product page getBaseSiteServer() null-guard latent issue` entry. Out of scope for this audit; flag for Mastermind to either fold into the fix brief (because both touch the same file and same data fetch) or open separately. File: `oglasino-web/app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:91-95`.

- **Adjacent observation 2 (low):** `FollowUserButton.tsx`'s `<button>` element (lines 71–98) wraps two click handlers that are placed on the inner `<UserMinus>` / `<UserPlus>` SVG components rather than on the `<button>` itself. The `<button>` has no `onClick`, no `type="button"`, and no `aria-label` (the tooltip is on the wrapping `WithTooltip`). Keyboard activation (Enter / Space on the focused button) does nothing. Accessibility regression, not the audited bug. Out of scope; flag for Mastermind to triage as its own polish item or fold into the fix brief if the engineer is already editing the file. File: `oglasino-web/src/components/client/buttons/FollowUserButton.tsx:71-98`.
