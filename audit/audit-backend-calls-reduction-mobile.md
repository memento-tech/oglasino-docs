# Audit — backend-calls-reduction (mobile)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-31
**Mode:** read-only audit. No code changes.
**Scope:** the four items in `.agent/brief.md`. Every "fires N times / cached / not cached" claim below traces to a `file:line` on the current `new-expo-dev` tree. The 2026-05-23 baseline (`sessions/2026-05-23-oglasino-expo-expo-readiness-backend-calls-reduction-1.md`) was run on `dev` and predates the boot redesign (decisions.md 2026-05-29); it is treated as a hint and re-verified throughout. Items 2 and 4 in particular have changed since that baseline.

---

## 1. `getUserForId` caching

**Status:** LIVE — uncached.

**Evidence:**
- Definition: `src/lib/services/userService.ts:59-67`. `getUserForId(userId)` does a bare `BACKEND_API.get('/public/user?id=' + userId)` and returns `res.data`. There is no memoization, no AsyncStorage, no store read-through — every call hits the backend.
- Call sites (only two):
  - `app/(portal)/(public)/product/[...productData].tsx:115` — `const fetchedOwner = await getUserForId(res.ownerId);` inside the product-load `useEffect` (the `load()` async fn at lines 102-136).
  - `app/(portal)/(public)/user/[...userData].tsx:32` — `getUserForId(userId).then(...)` inside that screen's effect.

**Frequency:** once per screen mount, per screen.
- Product detail: one `GET /public/user?id=<ownerId>` per product-detail view. Re-opening the same product, or opening two products by the same seller, re-fetches the owner each time (no cross-mount cache).
- User profile: one `GET /public/user?id=<userId>` per user-profile view, same re-fetch-on-revisit behaviour.
- Not per-render (both calls live in effects/async load fns, not render bodies).

**Note — the chat user cache is a different function and a different scope (baseline confirmed).** There IS a user cache, but it does not cover `getUserForId`:
- `src/lib/store/userCache.ts` exposes `getCachedUser` / `setCachedUser` / `clearUserCache`. Consumers are chat-only: `src/lib/store/useChatListStore.ts:18,25` and `src/lib/store/useActiveChatStore.ts:33,40`, plus `clearUserCache` on logout in `src/lib/store/authStore.ts:24`.
- That cache wraps **`getUserForFirebaseUid`** (`userService.ts:89`, `GET /auth/firebase/{uid}`) — a different endpoint keyed by firebaseUid — not `getUserForId` (`/public/user?id=`, keyed by numeric id). The two product/profile call sites bypass the cache entirely. So the baseline's "user cache is chat-scope only" is **confirmed**; the only change since baseline is that the cache moved from an in-`useChatStore` map to a dedicated `userCache.ts` module.

---

## 2. Version check on pathname change

**Status:** GONE (the per-pathname / per-navigation version check no longer exists).

**Evidence:**
- `AppVersionConfigInit.tsx` is deleted. It is listed in the boot-redesign deleted set (decisions.md 2026-05-29: "Deleted from the old path: … `AppVersionConfigInit.tsx` …"). On disk, `AppVersionConfigInit` survives only in two comment references — `src/components/init/HardUpdateScreen.tsx:11` and `src/lib/store/softUpdateDismissal.ts:6` ("Extracted/ported verbatim from the old AppVersionConfigInit") — and in a same-named-but-unrelated `AppVersionConfigurationDialog.tsx`. No `*.tsx` component named `AppVersionConfigInit` exists.
- The version call itself, `getAppVersionConfig()` (`src/lib/services/appVersionConfigService.tsx:6`, `GET /public/app/version/${Platform.OS}?currentVersion=…`), now has exactly one production caller: `src/lib/store/bootStore.ts:227` inside `runVersionGate` (Gate 2). All other matches are that file's own `bootStore.test.ts`.
- No `usePathname()` consumer triggers a version check. The live `usePathname()` sites are `SearchInput.tsx:53`, `ProductBreadcrumb.tsx:14`, `FiltersDialog.tsx:36`, `catalog/[...categories].tsx:21`, `owner/_layout.tsx:20` — none call `getAppVersionConfig` or any version path.

**Frequency:** once per boot sequence (Gate 2 runs once as the boot machine advances `runConfigGate → runVersionGate → runBaseSiteGate`). It does **not** fire on navigation, and it does **not** fire on app foreground.

**Note — what replaced it:** the boot-redesign `bootStore` gate machine (decisions.md 2026-05-29). Gate 2 (`runVersionGate`, `bootStore.ts:225-252`) is the sole version check; on a `forceUpdate` it routes to `toUpdateRequired` (HardUpdateScreen), on `optionalUpdate` it sets `softUpdate` (SoftUpdateModal), otherwise it advances. The only foreground-resume revalidation that still exists is `ForegroundRevalidationInit.tsx`, and it calls `syncUserToBackend` (auth re-check), **not** the version endpoint — and only after a 5-minute background threshold (`:9,:26`) and only for a logged-in user (`:28-32`). So there is no per-pathname and no per-foreground version check anywhere.

---

## 3. `ensureUserInFirestore` skip-for-returning-users

**Status:** LIVE — the doc-creation **write** is skipped for returning users, but a Firestore **read** still fires on every login.

**Evidence:**
- Definition: `src/lib/services/authService.ts:55-104`. Flow:
  1. `:59` `const snap = await retryWithDelay(() => getDoc(ref));` — a Firestore `getDoc` on `users/{uid}` (up to 3 attempts via `retryWithDelay`, `:106-120`).
  2. `:61` `if (snap?.exists()) return;` — **early return for returning users**: the photo upload, `setDoc(users/{uid})`, and `setDoc(userchats/{uid})` (`:63-100`) are all skipped when the doc already exists.
- Caller: `src/lib/services/authService.ts:143`, inside `buildUserSession`, which runs on every auth entry point — `loginUserFirebase` (`:153`), `registerUserFirebase` (`:161`), and (further down the file) the Google sign-in path. `buildUserSession` is `await ensureUserInFirestore(...)` then `syncUserToBackend(...)`.

**Frequency:** once per login/register/Google-sign-in (one `getDoc` read every time). For a returning user the read happens, `exists()` is true, and the function returns without any write. The expensive path (photo download/upload + two `setDoc` writes) runs only on the first-ever sign-in for that uid.

**Note:** the baseline's claim ("fires for every login → wasted Firestore read") is **still accurate for the read** — there is no skip on the `getDoc` itself (no local "already ensured" flag short-circuits the call before Firestore). What the current code does have is the `exists()` write-skip guard at `:61`, so returning users incur one read and zero writes rather than read + writes. I could not bind the `exists()` guard to the user-deletion Briefs 1–4 specifically: the brief's listed user-deletion edits to this file were the `deleteCurrentUser` helper and the C-6 listener guard, not this function, and the whole branch is uncommitted (issues.md 2026-05-31 "large uncommitted change set"), so `git blame` cannot attribute it. Reporting the as-built state only: write-skip present, read-skip absent.

---

## 4. Active base sites fetched per request

**Status:** NOT REPRODUCED on `new-expo-dev` — no base-site network call fires per request; the per-request path reads cached in-memory state, and the legacy list-fetch is now dead code.

**Evidence — the per-request path is a store read, not a fetch:**
- The only per-request hook is the axios request interceptor at `src/lib/config/api.ts:41-52`. It does:
  - `:44` `const { selectedBaseSite, language } = useBootStore.getState();`
  - `:45` `if (selectedBaseSite) config.headers.set('X-Base-Site', selectedBaseSite.code);`
  - `:46` `if (language) config.headers.set('X-Lang', language.code);`
  This is a synchronous in-memory Zustand read. No HTTP, no AsyncStorage read, on the request path. It sets the `X-Base-Site` / `X-Lang` headers from already-resolved boot state.

**Evidence — where base sites actually get fetched (none per-request):**
- `fetchBaseSites()` (`src/lib/init/baseSitesService.ts:20`, `GET /public/baseSite/details`) — the legacy full-list fetch the baseline flagged. It now has **zero callers** (grep for `fetchBaseSites` returns only its own definition and log lines). Dead code post-boot-redesign.
- `fetchBaseSiteOverviews()` (`baseSitesService.ts:40`, `GET /public/baseSite/overviews`) — called from `bootStore.ts:285` in `runBaseSiteGate`, **only on the no-stored-site path** (first run / cleared storage), and from `bootStore.ts:365-369` `ensureBaseSiteOverviews`, which is on-demand for the base-site picker dialog (`PortalConfigDialog.tsx:51`) and guarded by `if (get().baseSiteOverviews.length > 0) return;` (`bootStore.ts:366`).
- `fetchBaseSiteByCode()` (`baseSitesService.ts:48`, `GET /public/baseSite/{code}`) — called only from `bootStore.ts:313` `pickBaseSite`, i.e. when the user taps a site in the picker.

**Evidence — the base site IS cached:**
- Persisted: `setStoredBaseSite` writes the full DTO to AsyncStorage key `base_site` (`baseSitesService.ts:7-18`).
- In-memory: `bootStore` holds `selectedBaseSite` (set once at boot — `runBaseSiteGate` reads `getStoredBaseSite()` at `bootStore.ts:270` and sets the slot at `:279`, or `pickBaseSite` sets it at `:327`). The interceptor reads this slot. Same model as language and config (all boot-redesign slots).

**Frequency:** base-site network calls fire **per cold boot at most** (one `getStoredBaseSite` AsyncStorage read; plus one `overviews` fetch only when there is no stored site, plus one `byCode` fetch when the user actively picks). Zero base-site fetches per backend request, per navigation, or per warm foreground.

**Note:** the issues.md 2026-05-31 entry and this brief's item 4 ("active base sites fetched on every request, uncached") describe pre-boot-redesign behaviour (the `fetchBaseSites` full-list path) that no longer runs on `new-expo-dev`. On the current branch the symptom is not present in code: the interceptor reads a cached store slot, and the old per-fetch list call is unreferenced. If Igor observed per-request base-site traffic recently, the most likely explanations are (a) an older build/branch, or (b) seeing the `X-Base-Site` request header (set on every call from cache) mistaken for a base-site fetch — not an actual `GET /public/baseSite/*` per request. There is no code path on `new-expo-dev` that fetches base sites per request. Flagged for Mastermind below.

---

## Summary (one line per item)

| # | Item | Status | Evidence | Frequency |
|---|---|---|---|---|
| 1 | `getUserForId` caching | LIVE — uncached | `userService.ts:59`; callers `product/[...productData].tsx:115`, `user/[...userData].tsx:32`; chat cache (`userCache.ts`) wraps `getUserForFirebaseUid` only | 1 backend call per screen mount; re-fetches on every revisit |
| 2 | Version check on pathname change | GONE | `AppVersionConfigInit.tsx` deleted (decisions.md 2026-05-29); `getAppVersionConfig` sole caller `bootStore.ts:227` (Gate 2); no `usePathname` site checks version | once per boot; not on navigation, not on foreground |
| 3 | `ensureUserInFirestore` skip | PARTIAL — write skipped, read not | `authService.ts:59` getDoc, `:61` `exists()` early-return; caller `:143` `buildUserSession` | 1 Firestore read per login; writes only on first-ever sign-in |
| 4 | Active base sites per request | NOT REPRODUCED | interceptor reads cached store (`api.ts:44-46`); `fetchBaseSites` (`baseSitesService.ts:20`) has zero callers; site cached in AsyncStorage `base_site` + bootStore slot | 0 fetches per request; overviews/byCode only at cold boot / picker |
