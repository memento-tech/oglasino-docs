# Audit — Product-Creation Entry Path & Missing User-Basic-Data Setup Gate

**Repo:** oglasino-expo
**Branch context:** brief named `new-expo-dev`; the working tree is on `main`. `git diff --name-only main new-expo-dev` returns **0 files** — the two branches have identical trees (`main` is one no-op commit, `0890f0d` "chore: initialize main branch", ahead). The audit reads the same code either way. No checkout performed (hard rule).
**Type:** Read-only audit. No code changed.
**Date:** 2026-06-06

---

## 1. The create-product trigger — every entry point

`NEW_PRODUCT_DIALOG` (`DialogId.NEW_PRODUCT_DIALOG = 'addNewProductDialog'`, `src/components/dialog/dialogRegistry.ts:6`) is opened from **four** call sites, not one:

1. **BottomBar "+" button** — `src/components/navigation/BottomBar.tsx:144`
   `onPress={() => openDialogSafe(() => openDialog(DialogId.NEW_PRODUCT_DIALOG))}`
   Gated by `openDialogSafe` (auth + hydration only — see §2).

2. **Dashboard sidebar "+" (PlusCircle)** — `src/components/dashboard/layout/DashboardSidebar.tsx:63`
   `<Pressable onPress={() => openDialog(DialogId.NEW_PRODUCT_DIALOG)}>`
   **No auth check, no setup gate.** Auth is implicit only because the dashboard lives behind the `(secured)` route guard.

3. **Free-zone page, hero CTA** — `app/(portal)/(public)/blog/free-zone.tsx:62`
   `!user ? openDialog(LOGIN_OPTIONS_DIALOG) : openDialog(NEW_PRODUCT_DIALOG)`
   Inline auth check only. No setup gate.

4. **Free-zone page, bottom CTA** — `app/(portal)/(public)/blog/free-zone.tsx:118`
   Identical inline auth check. No setup gate.

All four route through `DialogManager` (`src/components/dialog/DialogManager.tsx:48` maps `addNewProductDialog → AddUpdateProductDialog`).

**Conclusion:** the brief's claim that BottomBar is "the only entry point" is **false** — there are four. None of the four performs a `baseSite` / `regionAndCity` setup gate.

---

## 2. `openDialogSafe` — what it checks and what it does not

Defined locally in `src/components/navigation/BottomBar.tsx:36-45`:

```ts
const openDialogSafe = (onAuthenticated: () => void, loginDescription?: string) => {
  if (!_hasHydrated) return;                       // hydration gate
  if (!user) {                                     // auth gate
    openDialog(DialogId.LOGIN_OPTIONS_DIALOG, { dialogDescription: loginDescription });
  } else {
    onAuthenticated();                             // <-- runs callback unconditionally
  }
};
```

**Checks:** (a) auth-store hydration (`_hasHydrated`, `authStore`); (b) presence of `user`. If unauthenticated it opens `LOGIN_OPTIONS_DIALOG`.

**Does NOT check:** anything about the user's *content* — not `user.baseSite`, not `user.regionAndCity`, not ban/deletion state, not subscription. Once `user` is truthy it calls `onAuthenticated()` verbatim. It is purely an auth-presence + hydration guard, not a setup gate. It is also local to BottomBar; the other three entry points don't even use it.

---

## 3. Dialog registry — every `DialogId`, and the basic-data question

`src/components/dialog/dialogRegistry.ts` enumerates 24 ids:

`LOGIN_OPTIONS_DIALOG`, `REGISTER_DIALOG`, `LOGIN_DIALOG`, `SUGGEST_CATEGORY_DIALOG`, `NEW_PRODUCT_DIALOG`, `INFO_DIALOG`, `REPORT_DIALOG`, `PREVIEW_PRODUCT_DIALOG`, `DASHBOARD_PRODUCT_FUNCTIONS_DIALOG`, `VIEW_MESSAGE_IMAGES_DIALOG`, `PRODUCT_REVIEW_DIALOG`, `PRODUCT_REVIEW_NOT_ALLOWED_DIALOG`, `PRODUCT_ALREADY_REVIEWED_DIALOG`, `PRODUCT_SUCCESS_DIALOG`, `CARD_SELECTION_DIALOG`, `CURRENCY_SELECTION_DIALOG`, `FILTERS_DIALOG`, `APP_VERSION_CONFIGURATION_DIALOG`, `SOFT_PUSH_PERMISSION_DIALOG`, `CHAT_USER_FUNCTIONS_DIALOG`, `PORTAL_CONFIG_DIALOG`, `DELETE_ACCOUNT_CONFIRMATION_DIALOG`, `VERIFY_EMAIL_DIALOG`.

**There is NO `USER_BASIC_DATA_SELECTOR` equivalent.** Nothing in the registry collects baseSite + regionAndCity as a setup step. (`VIEW_MESSAGE_IMAGES_DIALOG` is registered in the enum but commented out in the `DialogManager` map at line 57 — unrelated.)

---

## 4. `NEW_PRODUCT_DIALOG` open with null `baseSite` / null `regionAndCity` — where it fails

Component: `AddUpdateProductDialog` (`src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx`).

Boot/browsing base site `selectedBaseSite` comes from `bootStore` (always present post-boot, behind the boot `RequireBaseSite` gate). `user` is present (entry-point auth gate). The first effect resolves which base site the wizard uses:

```ts
useEffect(() => {
  if (validBaseSite) return;
  if (!user || !selectedBaseSite) return;        // line 81 — guards user, NOT user.baseSite
  const userBaseSite = user.baseSite;            // line 82 — null if user has no baseSite

  if (
    userBaseSite.code === selectedBaseSite.code ||   // line 85  <-- FAILS HERE
    userBaseSite.domain === selectedBaseSite.domain
  ) {
```

**Failing line (quoted):** `src/components/dialog/dialogs/product-creation/AddUpdateProductDialog.tsx:85`
`userBaseSite.code === selectedBaseSite.code ||`

If `user.baseSite` is `null`/`undefined`, line 82 binds `userBaseSite = null` and line 85 dereferences `.code` on null → `TypeError: Cannot read property 'code' of null`, thrown from inside a `useEffect` → React error / red-screen crash. `productData` is never set (it depends on `validBaseSite`, line 94-111), so `if (!productData) return <></>` at line 184 never even gets a chance to short-circuit gracefully — the throw happens first, after the empty first render.

**`regionAndCity` null does NOT crash.** The downstream step `BasicInfoProductDialog` reads it defensively:
- `src/components/.../BasicInfoProductDialog.tsx:58` — `useAuthStore((s) => s.user?.regionAndCity)` (optional chaining).
- line 261 — `{userRegionAndCity?.region && userRegionAndCity.city && (<CitySelector ... disabled />)}` — render-guarded; absent region/city just renders nothing.

So the crash is **specifically a null `user.baseSite`**, in the parent `AddUpdateProductDialog` effect, before any field is collected. A user with a baseSite but null regionAndCity opens and fills the wizard without a client crash (see §7 for what happens on submit).

---

## 5. Existing mobile surface to set `baseSite` / `regionAndCity`

- **`regionAndCity`: YES, partial.** `app/owner/user.tsx` (the owner profile/settings screen) renders `CitySelector` (line 269), seeds it from `details.regionAndCity` (line 75), and saves via `updateUser({ ..., regionAndCity: selectedRegionAndCity })` (`app/owner/user.tsx:178-186` → `updateUser` in `src/lib/services/userService.ts:30` → `POST` through `UpdateUserDTO`). So an *existing* user can edit region/city from settings.

- **`baseSite`: NO. There is no surface anywhere.** `UpdateUserDTO` (`src/lib/types/user/UpdateUserDTO.ts`) has **no `baseSite` field** — it carries `regionAndCity?` but not baseSite. No dialog, profile, or settings flow lets a user set/select their own `baseSite`. (`src/components/init/BaseSiteSelector.tsx` is the *boot-time browsing* base-site picker — props `isMaintenance` / `isOffline`, wired into `app/_layout.tsx` — and writes the `bootStore` browsing site, **not** the user's profile `baseSite`. It is not a user-setup surface.)

**Plainly:** mobile can edit region/city after the fact, has no baseSite-setup surface at all, and — critically — has **no gate** that forces either to exist before opening the create wizard.

---

## 6. `useAuthStore.user` shape — `baseSite` / `regionAndCity`

`user` is an `AuthUserDTO` (`src/lib/store/authStore.ts:112,145` — `setUser`; populated from `POST /auth/firebase-sync`, `src/lib/services/authService.ts:144-152`).

`src/lib/types/user/AuthUserDTO.ts`:
- line 9 — `baseSite: BaseSiteDTO;` — **typed non-nullable.**
- line 10 — `regionAndCity: RegionAndCityDTO;` — **typed non-nullable.**

**Value for a freshly-registered, setup-incomplete user:** the TypeScript type asserts both are always present, so the compiler never forces a null-guard (which is exactly why line 82/85 in §4 has none). Whatever `/auth/firebase-sync` actually returns for a brand-new `users` row (`wasRegister: true`) is the runtime truth, and it is **not verifiable from this repo** — it's the backend's emission. The reported crash is only explicable if the backend emits `baseSite: null` (and/or `regionAndCity: null`) for such a user, i.e. the runtime value **contradicts the declared mobile type.** This is the central seam (see Seams). The mobile code trusts the type and pays for it at `AddUpdateProductDialog.tsx:85`.

---

## 7. Trust boundary (Part 11) — is the setup requirement load-bearing or UX-only?

Create path: `AddUpdateProductDialog` → `BasicInfoProductDialog` → `productService.saveProduct` → `createProduct` → `POST /secure/products/create` (`src/lib/services/productService.ts:88,103-105`).

Key fact: the wire payload is built by an **allow-list narrowing** `toCreateWirePayload`, which deliberately **omits `regionAndCity` and `free`** (`src/lib/services/productService.ts:75-80` comment; `src/lib/types/product/CreateProductWireDTO.ts:9`; `src/lib/services/productWirePayload.ts`). The client never sends baseSite or regionAndCity — both are **server-derived from the authenticated user** (web parity; this is the Part 11 design — see also `BasicInfoProductDialog.tsx:55-58` "immutable and server-derived").

Therefore:
- There is **no client gate** today that enforces "setup must exist before create" — the four entry points open the wizard unconditionally past auth.
- The actual requirement (a product needs an owner baseSite + region/city) can only be enforced **server-side**, since those values are read from the authenticated user on the backend, not sent by mobile.
- **Verdict:** the client gate is **UX-only** (and currently absent/broken — it crashes instead of gating). The **server is the load-bearing boundary** per conventions Part 8 ("server is the single source of truth") and Part 11 ("derived from the authenticated identity"). Confirming the backend's exact behavior for a setup-incomplete user (reject with which code/status, vs. create a location-less product) requires the **backend audit** — it is out of mobile scope and not verifiable from this repo. Flagged as a seam.

A web-parity gate that opens a basic-data collector before the wizard would be a **UX** improvement (and the fix for the §4 crash), not a security control.

---

## Seams — each place mobile assumes something about backend or web

1. **`AuthUserDTO.baseSite` non-null.** Mobile assumes `/auth/firebase-sync` always returns a `baseSite`; the reported crash implies the backend can emit `null` for a setup-incomplete user, contradicting the declared type (`AuthUserDTO.ts:9`).
2. **`AuthUserDTO.regionAndCity` non-null.** Same assumption for region/city (`AuthUserDTO.ts:10`); downstream code (`BasicInfoProductDialog.tsx:58,261`) actually treats it as optional, so the type and the usage already disagree internally.
3. **Server derives baseSite + regionAndCity on create.** Mobile assumes the backend reads the owner's stored baseSite/region from the authenticated identity and intentionally never sends them (`productWirePayload.ts`, `CreateProductWireDTO.ts:9`).
4. **Server enforces the setup precondition.** Mobile assumes the backend rejects (or otherwise handles) a create from a user with no baseSite/region — there is no client enforcement, and mobile cannot verify this from its own repo.
5. **No web-parity setup gate exists on mobile.** Mobile assumes (incorrectly) that reaching the create button implies completed setup; web reportedly opens `USER_BASIC_DATA_SELECTOR_DIALOG` first, mobile has no equivalent (`dialogRegistry.ts`).
6. **`selectedBaseSite` (bootStore) is always present.** The wizard assumes the boot/browsing base site is set (boot `RequireBaseSite` gate), which holds — but it then conflates that browsing site with the *user's profile* baseSite at `AddUpdateProductDialog.tsx:82-91`.
7. **Web's create wizard shape is the contract.** Several comments assert "web parity" (immutable server-derived region, `setCurrentStep(Math.max(0, step-1))` mirror) — mobile assumes web's step model and field-derivation rules without a local source of truth.
