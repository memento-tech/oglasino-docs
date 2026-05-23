# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-18
**Task:** Badge Cache Invalidation Fix (Issue C residual) + Post-Deletion Refresh — diagnose why the `ScheduledForDeletionBadge` persists after restoration despite the new `revalidateUserCache` action, fix it, and add `router.refresh()` to the post-deletion dialog's `onClose`.

## Investigation findings (Step 0)

**Confirmed root cause: hypothesis (c) — wrong `revalidateTag` shape for the installed Next.js version.** The other three hypotheses are contradicted.

### Step 0.1 — Tag string match (hypothesis (b) ❌ contradicted)

- Action: `app/actions/revalidateUserCache.ts:11` (pre-fix) — `revalidateTag(\`user:${userId}\`, 'default')`.
- Consumer: `src/lib/service/nextCalls/userService.ts:18` — `tags: ['user', \`user:${userId}\`]`.
- Both sides use identical template literals; no separator, casing, or whitespace mismatch. `userId` is a `number` in both call sites, coerced the same way.

### Step 0.2 — `userId` read at close time (hypothesis (a) ❌ contradicted)

- `src/components/client/initializers/AccountStateDialogsInit.tsx:107-114` (pre-fix) reads `useAuthStore.getState().user?.id` **inside the `onClose` closure**, executed only when the dialog is dismissed.
- Restoration fires after a successful sign-in interceptor flip; by the time the user (or 10s auto-dismiss) closes the dialog, the auth store has the restored user populated.
- The handler `await`s `revalidateUserCache(userId)` before `router.refresh()`.
- A temporary `console.log` was not needed — the symptom is reproducible even when `userId` is observably present (badge sticks regardless of how the dialog closes).

### Step 0.3 — Next.js version and `revalidateTag` signature (hypothesis (c) ✅ confirmed)

- `package.json` declares `"next": "^16.2.6"`; `node_modules/next/package.json` confirms 16.2.6 installed.
- `node_modules/next/dist/server/web/spec-extension/revalidate.d.ts`:
  ```ts
  revalidateTag(tag: string, profile: string | CacheLifeConfig): undefined;
  ```
  The second argument is now a **cacheLife profile name**, not the Next-15 `'default' | 'tags'` switch the brief assumed.
- `node_modules/next/dist/server/web/spec-extension/revalidate.js:42-49` shows the deprecation path:
  ```js
  function revalidateTag(tag, profile) {
    if (!profile) console.warn('"revalidateTag" without the second argument is now deprecated...');
    return revalidate([tag], `revalidateTag ${tag}`, profile);
  }
  ```
- `node_modules/next/dist/server/config-shared.js:136-141` defines the default `cacheLife` profiles. The `'default'` profile resolves to `{ stale: undefined, revalidate: 900, expire: INFINITE_CACHE }`.
- `node_modules/next/dist/server/revalidation-utils.js:100-138` shows the per-handler call: with a string profile, `durations = { expire: cacheLife.expire }` is passed to `incrementalCache.revalidateTag`. The inline source comment is explicit:
  ```js
  // If profile is not found and not 'max', durations will be undefined
  // which will trigger immediate expiration in the cache handler
  ```
- Net effect of `revalidateTag(tag, 'default')` in Next 16: tells the cache handler the entry is fresh until `INFINITE_CACHE` — the literal opposite of immediate invalidation. The tag is queued but the cache is told to keep the stale entry. Hence the badge sticks until the original 5-minute `revalidate: 300` window elapses naturally and is unrelated to this action firing.
- Confirming the `pathWasRevalidated` flag: in `revalidate.js:204-208`, the flag is set only when `!profile || cacheLife?.expire === 0`. With `'default'` profile and `expire: INFINITE_CACHE`, neither branch fires — so the response also doesn't carry the path-revalidated signal the client uses to invalidate its router cache.
- The Next-16 Server-Action API for immediate, on-demand tag invalidation is `updateTag(tag)` (`revalidate.js:50-69`): it passes `undefined` profile (triggers immediate expiration in the handler, no deprecation warning) and marks the path as revalidated.

### Step 0.4 — Diagnostic side-experiments (not needed)

The Next 16 source is unambiguous: `'default'` profile is documented as the long-cache profile for `"use cache"` data, never as an immediate-purge value for fetch-tag caches. No `revalidatePath` belt-and-suspenders or hardcoded-tag isolation was required to confirm the diagnosis. The on-tree comment at `app/api/revalidate/route.ts:80-82` ("'default' is fine for traditional fetch().next.tags caches") is wrong — flagged in "For Mastermind" because that route handler has the same bug for any caller relying on it for immediate invalidation.

### Decision

Hypothesis (c) confirmed. Fix is to swap `revalidateTag(\`user:${userId}\`, 'default')` for `updateTag(\`user:${userId}\`)` — same import path (`next/cache`), Server-Action-context-only (which the action is), Next-16-correct semantics.

## Implemented

1. **Step 1 — cache-bust fix.** `app/actions/revalidateUserCache.ts` swapped from `revalidateTag(tag, 'default')` to `updateTag(tag)`. Expanded the file-level comment to document the Next-16 mechanism so a future reader sees why the `'default'` profile was wrong (cacheLife shape, INFINITE_CACHE expire, path-revalidated flag), why `updateTag` is correct (immediate expiration + path flag from a Server Action), and what threat-model decision is intentional (no auth gate — mirrors `/api/revalidate`).

2. **Step 2 — post-deletion `router.refresh()`.** Added a custom `onClose` to the post-deletion `openDialog(INFO_DIALOG, ...)` call in `src/components/client/initializers/AccountStateDialogsInit.tsx`. Shape mirrors the ban-notice handler: `useDialogStore.getState().closeDialog()` followed by `router.refresh()`. No `revalidateUserCache` call here (the user is signed out by this point, per the brief; their profile-page third-party visibility is the B10 backlog item, not this brief's scope). `router` added to the deps array — `tButtons` and `tDash` were already there; this is a minor extension. Inline comment explains the no-cache-bust decision and points at B10.

3. **Step 1 call-site comment.** Updated the restoration `useEffect` comment in `AccountStateDialogsInit.tsx` to name the failure mode explicitly — that `revalidateTag(tag, 'default')` was the Next-16 cause of post-restoration badge persistence and that `updateTag` replaces it. Future readers tracing the badge symptom find the breadcrumb here and the mechanism note in the action file.

## Files touched

- `app/actions/revalidateUserCache.ts` (+13 / −2) — `revalidateTag(tag, 'default')` → `updateTag(tag)`, comment expanded.
- `src/components/client/initializers/AccountStateDialogsInit.tsx` (+11 / −2) — post-deletion custom `onClose` (Step 2), restoration `useEffect` comment expanded to reference the `updateTag` mechanism (Step 1 call-site documentation), `router` added to post-deletion deps array.

## Tests

- **Baseline (session start):**
  - `npx tsc --noEmit`: clean (0 errors)
  - `npm run lint`: 208 warnings, 0 errors
  - `npm test`: 154/154 passed
- **End of session:**
  - `npx tsc --noEmit`: clean (0 errors)
  - `npm run lint`: 208 warnings, 0 errors (unchanged)
  - `npm test`: 154/154 passed
- **New tests:** none (per brief — no tests added).

## Cleanup performed

- None needed. No temporary `console.log` was added during Step 0 — the diagnosis was derivable from reading the installed Next 16 source directly (the test in `revalidate.js:204-208`, the cacheLife defaults in `config-shared.js:136-141`, the source comment in `revalidation-utils.js:127-128`) without a runtime log. Brief flagged this as a "cheapest way to falsify (a)" but (a) was contradicted from code reading alone; running the repro added no signal because the symptom is unconditional on `userId` presence.
- No commented-out code, dead imports, console.log, or TODO/FIXME introduced. `revalidateTag` import in `revalidateUserCache.ts` replaced with `updateTag` (one-for-one; no orphan).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

(All four config files: explicitly no change. The two adjacent observations in "For Mastermind" — `/api/revalidate` and `cacheActions.ts` carrying the same wrong-`'default'`-profile bug, and the stale comment in `/api/revalidate/route.ts:80-82` — are drafts for Mastermind to triage, not config-file edits owned by this session.)

## Obsoleted by this session

- The previous `dialog-system-fix-1` session's `revalidateTag(tag, 'default')` shape in `app/actions/revalidateUserCache.ts` is dead. Deleted in this session and replaced with `updateTag(tag)`.
- The brief's hypotheses (a), (b), (d) are dead — the on-tree code now reflects the (c)-confirmed root cause documented above.
- The previous session's omission of `router.refresh()` from the post-deletion dialog's close handler is fixed in this session — the dialog now triggers a re-render so the (now signed-out) home page replaces the (signed-in-with-products) view that was visible behind the dialog.
- Nothing else was made stale. The restoration `onClose`, ban-notice `onClose`, single-slot store reactive pattern, and the `revalidateUserCache` action's shape all remain correct.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, dead imports, console.log, or TODO/FIXME without a session-summary entry. Touched-path verifications all green at baseline values.
- Part 4a (simplicity): confirmed. One-line API swap (`revalidateTag` → `updateTag`); no new abstractions, no extra plumbing. The expanded comment is the load-bearing change — it documents a non-obvious Next-16 semantic shift that the next engineer would otherwise have to rediscover from source. The post-deletion `onClose` reuses the existing `dialogProps`-spread seam in `DialogManager.tsx:69`, same pattern as the ban-notice handler.
- Part 4b (adjacent observations): three observations flagged in "For Mastermind" below — the same wrong-`'default'`-profile bug in `/api/revalidate/route.ts` and `app/actions/cacheActions.ts`, plus the stale comment block in the API route. All three are out-of-scope per the brief's hard rules ("Do not touch other files in the auth/dialog area beyond what Steps 1 and 2 require") and flagged for Mastermind triage.
- Part 5 (session summary): this summary written to both `.agent/2026-05-18-oglasino-web-badge-cache-fix-1.md` and `.agent/last-session.md` per rule. First session for slug `badge-cache-fix` in this repo, so `<n> = 1`.
- Other parts touched: none. Part 11 (trust boundaries) N/A — the action's threat model is unchanged from the prior session (public-data tag bust, no identity-derived authorization).

## Known gaps / TODOs

- Translation key `buttons.account.deleted.acknowledge.label` still missing per Task 6 (separate session, unchanged from prior summary). The post-deletion dialog now closes with `router.refresh()`, but the button label itself may still render as a raw key until Task 6 lands. Expected behaviour, not a regression.
- Backend → web cache invalidation (B10) still open — when a third-party user views the just-deleted or just-restored user's profile from their own session, they keep the stale `UserInfoDTO` until the 5-minute `revalidate` window elapses naturally. This session's fix only addresses the restored-user's own view + an other-tab same-user view (because both share the Next.js Data Cache layer). The cross-viewer gap is the backend's responsibility via `POST /api/revalidate` and is a separate Backend brief per the prior session's For-Mastermind item 2.

## For Mastermind

1. **Same wrong-`'default'`-profile bug in `app/api/revalidate/route.ts:80-83` and `app/actions/cacheActions.ts:28,39` (severity: medium, project-wide cache infra).** Both call sites use `revalidateTag(tag, 'default')` and `revalidateTag(tag, 'default')` for general-purpose cache eviction. By the Next 16 mechanics documented above, these calls do not immediately invalidate fetch-tag caches — they tell the handler the entries are fresh until `INFINITE_CACHE`. Every backend-triggered cache bust (translations, base-site, config, product, user) and every admin-panel cache eviction has the same latent no-op behaviour. The on-tree comment at `/api/revalidate/route.ts:80-82` reads:
   ```ts
   // Next.js 16: revalidateTag requires a cache-life profile as second arg.
   // 'default' is fine for traditional fetch().next.tags caches.
   ```
   The first line is correct; the second is wrong — `'default'` is exactly the profile that does NOT immediately invalidate. Fix is the same swap done in this session: `revalidateTag(tag, 'default')` → either `revalidateTag(tag, 'max')` (the deprecation-warning-suggested profile, but with `expire: 60 * 60 * 24 * 365` is *also* not immediate — only `revalidateTag(tag)` no-arg or `updateTag(tag)` is) → or for the Server-Action path (`cacheActions.ts`), `updateTag(tag)`; for the Route Handler path (`/api/revalidate/route.ts`, which cannot use `updateTag` — it throws outside Server Actions), bare `revalidateTag(tag)` (eats a deprecation warning but works) OR a switch to `revalidatePath(path, 'page')` (already used for the path side of that endpoint). Suggested routing: `issues.md` with severity `medium`, single entry covering both files since the root cause and fix shape are identical, or fold into the cross-repo Backend brief from the prior session's For-Mastermind item 2 since that brief will exercise `/api/revalidate` end-to-end and the gap will be observable there.

2. **`updateTag` is Server-Action-only.** The Next 16 `updateTag` throws `E872` if called from a Route Handler or any non-Server-Action context (`revalidate.js:50-58`). When `/api/revalidate/route.ts` is fixed (item 1 above), the right call there is bare `revalidateTag(tag)` (accepts the deprecation warning) or `revalidatePath` — not `updateTag`. Worth a sentence in the next brief touching that route handler so the Backend engineer doesn't mirror this session's swap mechanically.

3. **No config-file dependency from this session.** Closure-gate check: nothing here requires Docs/QA to apply edits to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`. Item 1 above is a draft text candidate for `issues.md` if Mastermind agrees but is not owned by this session — Mastermind to triage routing alongside the Backend B10 brief from the prior session.

4. **No spec drift this session.** The user-deletion spec sections §14.4 / §14.5 / §14.10 already flagged in the prior session's For-Mastermind item 3 (with stale `sessionStorage` references and an incorrect "user is already on the home page" line in §14.10) remain to be reconciled by Docs/QA when next touched. This session's changes — `updateTag` swap and post-deletion `router.refresh()` — slot into the same call sites, so the spec lag is the same shape, just one increment further behind.
