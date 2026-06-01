# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-01
**Task:** AUDIT (read-only) — Where does web's update-product screen get region/city? (issues.md 2026-05-31 — "Mobile: update-product screen crashes — regionAndCity undefined on productDetails")

This was a read-only audit. No code changed, nothing staged.

---

## Verdict on Q3 (the one-liner)

**(a)** — Web's update-product screen shows the **authenticated user's** region/city, fetched from a **separate user endpoint** (`GET /auth/firebase/{firebaseUid}` → `UserInfoDTO`), **not** from the product-details response. The product details (`productDetails` / `ProductEditState`) are never read for the displayed region/city, so the fact that the owner product-details endpoint omits `regionAndCity` does not affect web — it sources the value elsewhere.

(Nuance for mobile, expanded in Q3 below: `ProductEditState` *declares* a `regionAndCity?` field, but the screen does not render from it. The rendered value comes from the user object.)

---

## Findings (1–6)

### 1. Web's update-product screen — the file(s)

- **`app/[locale]/owner/products/[productId]/page.tsx`** — `ProductDetailsPage`, the owner-dashboard "edit a product" screen. This is the web equivalent of mobile's `app/owner/dashboard/products/[productId].tsx` (`UpdateProductScreen`).
- Supporting component: **`src/components/owner/client/CitySelector.tsx`** — renders the region/city control (display-only here).

### 2. Where region/city is DISPLAYED

`CitySelector` is mounted twice — once in the mobile-width layout, once in the xl-width layout. Both render the region/city:

- Mobile-width layout — `page.tsx:394-401`:
  ```tsx
  <CitySelector
    selectedRegionAndCity={userInfo.regionAndCity}
    onChange={() => {}}
    regions={baseSite.regions}
    showOnSelector={false}
    required={true}
    disabled={true}
  />
  ```
- xl-width layout — `page.tsx:472-479`:
  ```tsx
  <CitySelector
    selectedRegionAndCity={userInfo.regionAndCity}
    onChange={() => {}}
    regions={baseSite.regions}
    showOnSelector={false}
    required={true}
    disabled={true}
  />
  ```

Inside `CitySelector`, the labels are read from the leaf `labelKey` and passed through next-intl `t(...)`:
- `CitySelector.tsx:80` — `` `${t(selectedRegionAndCity.region.labelKey)} / ${t(selectedRegionAndCity.city.labelKey)}` `` (the "Region / City" line)
- `CitySelector.tsx:71` — `setButtonLabel(t(selectedRegionAndCity.city.labelKey))` (button label = city)

### 3. Where the displayed VALUE comes from (the trace) → **(a)**

The value passed to `CitySelector` is **`userInfo.regionAndCity`**, never `productDetails.regionAndCity`.

`userInfo` is populated by a **separate user fetch**, not by the product-details call:

- `page.tsx:101` — `const userInfo = await getUserForFirebaseUid(user.firebaseUid);`
- `page.tsx:103` — `setUserInfo(userInfo);`
- `page.tsx:72` — `const [userInfo, setUserInfo] = useState<UserInfoDTO>();`

`getUserForFirebaseUid` hits a dedicated user endpoint and returns `UserInfoDTO`:
- `src/lib/service/reactCalls/userService.ts:77-93` — `GET /auth/firebase/{firebaseUid}` → `UserInfoDTO | null`.

The product-details fetch is a *different* call (`page.tsx:93` → `getDashboardProductDetails(productId)` → `productsSearchService.ts:57`, owner-scoped `GET .../products?productId=<id>`). Its result is stored in `productDetails` and `oldProductDetails` and is **not** the source of the rendered region/city.

So web reads the **user's** region/city from a **separate endpoint/field** — this is squarely option **(a)** (and, because the user fetch is a distinct endpoint, it also technically satisfies "(b) a different endpoint"; the controlling fact is that the *value is the user's*, not a per-product value).

**Important sub-finding for the mobile fix:** `ProductEditState` *does* declare a `regionAndCity?` slot as a "read-only display field" (`src/lib/types/product/ProductEditState.ts:34`, commented "rendered on the edit page; never sent on update"). **But the screen does not render from it** — `page.tsx` passes `userInfo.regionAndCity`, never `productDetails.regionAndCity`. Whether or not the owner product-details endpoint populates that slot, web does not depend on it. (Same pattern in the *create* flow: `BasicInfoProductDialog.tsx:273` passes `user.regionAndCity` from the auth store.)

### 4. Editable or read-only on web?

**Read-only / display-only.** Both `CitySelector` mounts are `disabled={true}` with `onChange={() => {}}` (no-op). Inside `CitySelector`, `disabled` suppresses the popover, typing, and selection (`CitySelector.tsx:59, 101, 110, 128-129, 153-155`). Region/city is immutable on the web update screen — consistent with the backend treating it as immutable post-create.

### 5. VERBATIM user type + region/city field shape (the key part for mobile)

The displayed value is `UserInfoDTO.regionAndCity`, of type `RegionAndCityDTO`. Verbatim:

`src/lib/types/user/UserInfoDTO.ts`:
```ts
export interface UserInfoDTO {
  ...
  regionAndCity: RegionAndCityDTO;
  ...
}
```

`src/lib/types/catalog/RegionAndCityDTO.ts` (entire file, verbatim):
```ts
export type KeyLabelPair = {
  id: number;
  labelKey: string;
};

export type RegionAndCityDTO = {
  region: KeyLabelPair | null;
  city: KeyLabelPair | null;
};
```

So the exact shape mobile should match:
- `regionAndCity.region` → `{ id: number; labelKey: string } | null`
- `regionAndCity.city` → `{ id: number; labelKey: string } | null`
- **The leaf used for display is `.labelKey`** (translated via `t(...)`), **not** `.key` or `.label`. Both `region` and `city` are individually **nullable**, and `regionAndCity` itself can be absent — `CitySelector` guards with `selectedRegionAndCity?.region && selectedRegionAndCity.city` before dereferencing (`CitySelector.tsx:70, 79, 88`).

The auth-store user object exposes the identical shape: `AuthUserDTO.regionAndCity: RegionAndCityDTO` (`src/lib/types/user/AuthUserDTO.ts:10`). So whether mobile reads its current user from a `UserInfoDTO`-equivalent or an auth-store user, the field is `regionAndCity.region.labelKey` / `regionAndCity.city.labelKey`, both optional/nullable.

### 6. Adjacent — does web guard imageKeys before mapping?

**Yes — web guards it.** Web never maps over `imageKeys` directly on the screen; it derives `imagesData` in the service with a nullish guard:
- `productsSearchService.ts:67` — `const imagesData = ((data.imageKeys ?? []) as string[]).map((key) => ({ key }));`

The `?? []` means an absent/undefined `imageKeys` yields an empty `imagesData`, not a crash. The screen then reads `productDetails.imagesData` (`page.tsx:350, 108, 187`), which is always at least `[]`. So web's analogue of mobile's unguarded `imageKeys.map` is the `?? []`-guarded derivation in the service layer — mobile can mirror that guard.

---

## Files touched

- None (read-only audit).

## Tests

- Not run (no code change). `npm run lint` / `npx tsc --noEmit` / `npm test` N/A this session.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change required from web. (The 2026-05-31 mobile crash entry is owned by a mobile chat; this audit feeds that fix but does not itself amend the log. Optional note for Mastermind below.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code written.
- Part 4a (simplicity): N/A — read-only audit, no abstractions added. See structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one observation flagged in "For Mastermind" (the unused `ProductEditState.regionAndCity` slot).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundaries) — confirmed: region/city is read-only on the web update screen and immutable post-create, consistent with the backend omitting it from the owner update DTO.

## Known gaps / TODOs

- This audit reads web only. It does not assert what the owner product-details endpoint *actually emits* for `regionAndCity` (that's a backend read) — but the answer is moot for web, which sources the value from the user.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **The answer mobile needs (Q3 = a):** web shows the **current user's** region/city, not a per-product value. Mobile's crash (`productDetails.regionAndCity.region.labelKey` undefined) is because the owner product-details endpoint deliberately omits `regionAndCity` (immutable post-create) — mobile is dereferencing the wrong source. The web-matching fix is to read region/city from the **user** (`user.regionAndCity.region.labelKey` / `.city.labelKey`), with `region`/`city`/`regionAndCity` all treated as nullable/optional, exactly as `CitySelector` guards. Display-only; do not make it editable.

- **Exact field names for mobile to match (Q5):** `regionAndCity.region.labelKey` and `regionAndCity.city.labelKey` — leaf is **`labelKey`** (not `.key`/`.label`), each of `region`/`city` is `{ id, labelKey } | null`, and `regionAndCity` itself may be absent. Mobile should verify its own user store exposes this same shape before wiring.

- **imageKeys (Q6):** web guards with `(data.imageKeys ?? [])` at `productsSearchService.ts:67`. Mobile's unguarded `imageKeys.map` can mirror this `?? []` guard.

- **Adjacent observation (Part 4b):**
  - **`ProductEditState.regionAndCity` is a declared-but-unrendered field.** `src/lib/types/product/ProductEditState.ts:34` declares `regionAndCity?: RegionAndCityDTO` with a comment claiming it is "rendered on the edit page," but the edit page renders `userInfo.regionAndCity`, never `productDetails.regionAndCity`. Severity: **low** (cosmetic / misleading comment; the field is dead weight on the view-model and the comment is inaccurate). I did not change it — out of scope for a read-only audit. Mastermind may want to route a one-line cleanup (drop the field + comment, or correct the comment) to a future web chat.

- **Optional issues.md note (web side):** if useful, the 2026-05-31 mobile crash entry could carry a one-line cross-reference that "web sources region/city from the user (`UserInfoDTO.regionAndCity` / `AuthUserDTO.regionAndCity`, leaf `labelKey`), display-only — mobile should match." Draft text is above; I did not write to issues.md (Docs/QA is the sole writer).
