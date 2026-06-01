# Session — FIX: update-product screen crash (`regionAndCity` undefined)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (no switch, no commit, no push — staged on disk only)
**Date:** 2026-06-01
**Slug:** update-product-region-crash (order 2)
**Type:** implementation. Follows the read-only audit `-1` (2026-05-31) + `.agent/audit-update-product-region-crash.md`.

---

## Task (one sentence)

Stop the update-product screen crashing on `productDetails.regionAndCity` (which the backend deliberately omits) by sourcing region/city from the authenticated user — display-only, guarded for all three nullable levels — and harden the adjacent `imageKeys.map`, matching web.

---

## Step 0 — shape verification (brief gate)

**Confirmed: mobile's user store exposes `regionAndCity` in the same shape web uses. Proceeded with the fix.**

- Auth store: `useAuthStore` exposes `user: AuthUserDTO | null` — `src/lib/store/authStore.ts:39`.
- `AuthUserDTO.regionAndCity: RegionAndCityDTO` — `src/lib/types/user/AuthUserDTO.ts:10`.
- `RegionAndCityDTO = { region: KeyLabelPair | null; city: KeyLabelPair | null }`; `KeyLabelPair = { id: number; labelKey: string }` — `src/lib/types/catalog/RegionAndCityDTO.ts:1-8`. Leaf is `labelKey` (NOT `.key`/`.label`), matching web.
- Three nullable levels available, exactly as the brief specified: `user` may be `null` (so `regionAndCity` may be absent), and `region`/`city` are each `| null`.
- `CitySelector` accepts `selectedRegionAndCity: RegionAndCityDTO | undefined` and already guards `selectedRegionAndCity?.city` before dereferencing `.labelKey` — `src/components/basic/CitySelector.tsx:14,40-41`. So passing `user?.regionAndCity` (possibly `undefined`) is type-safe and render-safe.

---

## What I did (the four scoped changes)

In `app/owner/dashboard/products/[productId].tsx`:

1. **Region/city now sourced from the user, not `productDetails`.** Added `const userRegionAndCity = useAuthStore((s) => s.user?.regionAndCity);` (with the `useAuthStore` import). The render block previously dereferenced `productDetails.regionAndCity.region.labelKey` / `.city.labelKey` raw — the crash. Backend's `UpdateProductRequestDTO` omits region/city (immutable post-create), so that field is always `undefined`. Now reads the user's own region/city.
2. **All three nullable levels guarded.** The label `<Text>` renders only under `userRegionAndCity?.region && userRegionAndCity.city`; when absent it renders nothing (matching web's CitySelector, which guards before rendering the label — no placeholder). `selectedRegionAndCity={userRegionAndCity}` is passed straight through (the component already null-guards internally).
3. **Display-only kept.** `onChange={() => {}}`, `disabled={true}`, `showOnSelector={false}` unchanged. Region/city is not editable and is not in the update payload (the `toUpdateWirePayload` allow-list already excludes it) — untouched.
4. **`imageKeys.map` hardened.** `productDetails.imageKeys.map(...)` → `(productDetails.imageKeys ?? []).map(...)`, mirroring web's `(data.imageKeys ?? [])` pattern. `imageKeys?: string[] | undefined` → an absent value now yields `[]` instead of a TypeError.

No backend change. No new endpoint/fetch. No update-payload change. Region/city not made editable.

---

## Brief vs reality

Nothing to challenge. The brief's root-cause analysis matched the code exactly (confirmed in the `-1` audit): the backend's `UpdateProductRequestDTO` omits region/city by design, and the screen read them raw. Step 0 confirmed the user object carries the field in the web shape, leaf `labelKey`, all three levels nullable — so the prescribed fix is implementable as written.

---

## Verification

- `npx tsc --noEmit` — clean (exit 0).
- `npx eslint "app/owner/dashboard/products/[productId].tsx"` — 1 warning, **pre-existing** (`react-hooks/exhaustive-deps` on the `fetchProduct` useEffect's `tDash` dep — not touched by this session). 0 errors. **No new warnings/errors introduced; lint baseline held.**
- `npm test` (vitest) — 26 files, 334 tests, all passing. No test directly covers this screen; the suite confirms no regression.
- Dependencies unchanged → `expo-doctor` not required.

---

## Cleanup performed

None needed. The change is additive/substitutive within one file: one import added (`useAuthStore`, used), one selector added (used), one render block swapped, one `?? []` guard. No commented-out code, no console logging, no TODO/FIXME, no unused symbols introduced. (The larger uncommitted diff in this file is pre-existing product-validation chat-A work, not this session's.)

---

## Obsoleted by this session

The raw `productDetails.regionAndCity.region/.city.labelKey` deref and the unguarded `productDetails.imageKeys.map` are gone. No prior file or doc superseded.

---

## Conventions check

- **Part 4 (cleanliness):** clean — no debug logging, no dead code, no unused imports/vars, no unmatched TODO/FIXME. tsc/lint/test green for the touched path; baseline held.
- **Part 4a (simplicity):** minimal change — reused the existing `CitySelector` (which already null-guards), the existing `useAuthStore` selector pattern, and web's `?? []` idiom. No new component, helper, or abstraction. Label guard is a single inline `&&`.
- **Part 4b (adjacent observations):** the `-1` audit's two adjacent items are now both resolved here (inner-null `region`/`city` and the `imageKeys.map` raw-optional deref). Nothing new surfaced worth logging.
- **Stack reminders:** the screen reuses the web endpoint `GET /secure/products`; region/city now read from the same user object web uses (`AuthUserDTO.regionAndCity`), same leaf `labelKey`. No mobile-specific route, no new translation keys (display reuses existing `COMMON_SYSTEM` labelKey lookups). Behavioral parity with web on this surface.

---

## Config-file impact

No edit to any of the four config files (`conventions.md`, `decisions.md`, `state.md`, `issues.md`). 

**Closure gate:** no implicit config-file dependency. This is a bug fix, not a feature adoption — it does not adopt or retire any `state.md` Expo-backlog row. The bug is tracked in `issues.md` (2026-05-31 entry, `open`); updating its status to `fixed` is Docs/QA's to make, not mine. See "For Mastermind."

---

## For Mastermind

1. **Fix landed (mobile-only).** `issues.md` 2026-05-31 "Mobile: update-product screen crashes — `regionAndCity` undefined on `productDetails`" can be flipped to `fixed`. Root cause confirmed as designed-omission, not a backend regression: the backend's `UpdateProductRequestDTO` deliberately omits region/city (immutable post-create), and the mobile screen was reading them from the wrong source. Fixed by sourcing from the authenticated user (web's pattern), display-only, guarded for all three nullable levels; the adjacent `imageKeys.map` hardened in the same pass.
2. **Suggested `issues.md` resolution note (Docs/QA to write):** "Resolved (2026-06-01, `oglasino-expo-update-product-region-crash-2`, `new-expo-dev`). Mobile-only bug — the screen read immutable region/city from `productDetails`, which the backend's `UpdateProductRequestDTO` omits by design. Now sourced from `useAuthStore().user.regionAndCity` (matching web), display-only, guarded for absent `regionAndCity` / null `region` / null `city`; renders nothing when absent. The adjacent `productDetails.imageKeys.map` hardened with `?? []`. tsc/lint/test green; lint baseline held. No backend change."
3. **Ψ note:** this surface still rides the product-validation on-device smoke (Ψ) dependency — the fix should be confirmed on a real device against a product whose owner has, and has not, a region/city set.
