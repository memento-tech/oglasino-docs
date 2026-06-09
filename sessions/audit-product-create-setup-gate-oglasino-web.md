# Audit — Product-Create Setup Gate (`USER_BASIC_DATA_SELECTOR_DIALOG`)

**Repo:** oglasino-web
**Branch:** dev
**Type:** Read-only audit (Phase 2). No code changes.
**Date:** 2026-06-06
**Purpose:** Document the web setup-gate behavior so mobile can mirror the *behavior*, not the web mechanism. Mobile's create flow has no `baseSite`/`regionAndCity` gate and is failing because of it.

---

## 1. `AuthAddNewProductButton` — the gate

**File:** `src/components/client/buttons/AuthAddNewProductButton.tsx`

Confirmed. The gate is exactly as the brief states.

```tsx
function openNewProductDialog() {
  if (!user) return;                                    // line 23

  if (!user.baseSite || !user.regionAndCity) {          // line 25 — the gate condition
    openDialog(DialogId.USER_BASIC_DATA_SELECTOR_DIALOG, {
      dialogTitle: 'base.site.select.required.title',
      shouldOpenDialog: true,
    });
    return;
  }
  openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG);        // line 32 — gate passed
}
```

- Gate condition: `!user.baseSite || !user.regionAndCity` (`AuthAddNewProductButton.tsx:25`).
- If gated → opens `USER_BASIC_DATA_SELECTOR_DIALOG` with `{ dialogTitle: 'base.site.select.required.title', shouldOpenDialog: true }` (`:26–29`).
- If passed → opens `CREATE_NEW_PRODUCT_DIALOG` directly (`:32`).
- `user` comes from `useAuthStore()` (`:19`); the gate reads the client-side hydrated user object.

`user.baseSite` is a `BaseSiteDTO` and `user.regionAndCity` is a `RegionAndCityDTO`, both on `AuthUserDTO` (`src/lib/types/user/AuthUserDTO.ts`). The dialog is registered at `DialogManager.tsx:63` (`userBasicDataSelectorDialog → UserBasicDataSelectorDialog`).

---

## 2. `USER_BASIC_DATA_SELECTOR_DIALOG` — the component

**File:** `src/components/popups/dialogs/UserBasicDataSelectorDialog.tsx`

### What it collects, and in what order

Two things, in this order:

1. **Base site** — a single choice from a button list (`:135–165`). The list is filtered to `site.domain === site.code` (`:143`), i.e. only top-level/canonical base sites are selectable.
2. **Region + city** — collected together via the shared `CitySelector` (`:167–176`), producing a `RegionAndCityDTO` (`{ region, city }`). The region list shown is the set of regions belonging to the selected base site; selecting a base site loads its regions and clears any previously chosen region/city (`:150–154`).

The region/city selector only renders once `selectedRegions` is populated, i.e. after a base site has been chosen (`:167`). So the flow is strictly base-site-first, then region/city.

### Precondition check — `canUserAssignLocation()`

On mount, the dialog calls `canUserAssignLocation()` (`:61`, `userService.ts:52`) → `POST /secure/user/update/region-city/validate-update`, returns `true` iff HTTP 200.

- If allowed → `validationFailed = false`, fetch the base-site list and render the selectors (`:62–63`).
- If not allowed → show `tValidation('base.site.select.validation.fail')`, hide the selectors and the update button (`:65–66`, `:133`, `:188`).

For the create-gate path the gated user has no location yet, so this normally returns `true`. The check exists because the same dialog also serves the "change my existing location" use case (where changing may be restricted).

### What persists each selection — store action / service / endpoint

There is a single persist call for both selections, made in `onUpdate()` (`:74–98`):

- `assignUserLocation(selectedRegionAndCity)` (`:84`, `userService.ts:36`) → **`POST /secure/user/update/region-city`** with the `RegionAndCityDTO` body. Returns `true` on 2xx, `false` otherwise.
- **The base site is *not* sent in this request.** Only `{ region, city }` crosses the wire. The server infers/derives the base site from the chosen region/city (a region belongs to exactly one base site) and/or from the authenticated identity. The base-site picker in the UI is what scopes which regions are offered; it is not itself persisted as a separate field.

Before persisting, a structural guard requires both a selected base site and a selected region/city, else it shows `tValidation('form.incomplete')` and returns (`:77–80`).

### `dialogTitle` and `shouldOpenDialog` params

- **`dialogTitle`** (string, required): a `DIALOG`-namespace translation key. The dialog renders it as the modal title via `tDialog(dialogTitle)` (`:112`). The create-gate caller passes `'base.site.select.required.title'`. It exists so the same dialog can be reused with a context-appropriate heading (e.g. a "change your location" entry point could pass a different key).
- **`shouldOpenDialog`** (boolean, default `false`): the post-success auto-advance flag. When `true`, after a successful persist the dialog opens `CREATE_NEW_PRODUCT_DIALOG`; when `false`/omitted, it just closes. See §2 post-success below.

The **only** call site that opens this dialog is `AuthAddNewProductButton`, and it always passes `shouldOpenDialog: true` (verified by repo-wide grep — no other `openDialog(USER_BASIC_DATA_SELECTOR_DIALOG, …)` exists). So in the current codebase the dialog is *exclusively* the create-product setup gate; the `shouldOpenDialog: false` branch is latent (reserved for a future settings/standalone-edit entry point).

### Post-success behavior — exact trace

In `onUpdate()` (`:82–97`), on a successful `assignUserLocation`:

1. `await refreshUser()` (`:85`) — re-fetch authoritative user (see §4).
2. If `shouldOpenDialog` → `openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)` (`:87–89`). **This auto-advances straight into the create-product dialog.**
3. `finally { setLoading(false) }` (`:95–97`).

It does **not** explicitly call `onClose()` on success. The create-product dialog is opened over/replacing it via the dialog store; the setup dialog is superseded by the newly opened dialog. (On failure it sets `errorMessage` to `tErrors('review.system.1')` and stays open — `:90–92`, `:93–94`.)

So: **after successful setup the user is auto-advanced into product creation in the same interaction** — they do not have to click "add product" a second time.

---

## 3. Base-site and region/city sub-flows — components and data sources

### Base-site selection
- **Component:** inline button list inside `UserBasicDataSelectorDialog` (`:135–165`).
- **Data source:** `getAllBaseSitesOverviews()` — note there are **two** implementations:
  - The dialog imports the **server action** `getAllBaseSitesOverviews` from `app/actions/getBaseSiteServer.ts` (`:3`), a `cache()`-wrapped server action → `GET /public/baseSite/overviews` (1-day Next Data Cache, tags `base-site` / `base-site:overviews`). Returns `BaseSiteOverviewDTO[]`.
  - (A second client-side `getAllBaseSitesOverviews` exists in `src/lib/service/reactCalls/baseSiteService.ts` → same endpoint; not the one used here.)
- List is filtered client-side to `site.domain === site.code` (canonical sites only) and labels render via `tIntro(site.labelKey)` with the flag image (`:143–161`).

### Region + city selection
- **Component:** `CitySelector` (`src/components/owner/client/CitySelector.tsx`), shared with the rest of the app.
- **Data source for regions:** `getBaseSiteRegions(baseSiteOverview.code)` (`:101`, `baseSiteService.ts:22`) → **`GET /public/baseSite/regions/{code}`**, returns `RegionDTO[]` (each region carries its `cities`).
- `CitySelector` renders region → city as a searchable command list and emits the chosen `{ region, city }` as a `RegionAndCityDTO` (`CitySelector.tsx:156–160`). City/region labels render via translation keys (`labelKey`), searchable with `normalizeSerbian`.

### Pre-fill for existing-location users
If the user already has `baseSite` + `regionAndCity`, an effect pre-seeds the selectors from the current user (`:51–57`). Irrelevant for the create-gate (gated users have neither), relevant for the future change-location use case.

---

## 4. How `user.baseSite` / `user.regionAndCity` get refreshed client-side

**Mechanism: server re-fetch via `refreshUser()`, not optimistic write.**

`UserBasicDataSelectorDialog.onUpdate` calls `refreshUser()` from `useAuthStore` after a successful persist (`:85`).

`refreshUser()` (`src/lib/store/useAuthStore.ts:266–294`):
1. Reads `auth.currentUser` (Firebase).
2. `syncUserToBackend(firebaseUser)` → **`POST /auth/firebase-sync`** (`authService.ts:syncUserToBackend`), which returns the full authoritative `AuthUserDTO` — including the now-updated `baseSite` and `regionAndCity`.
3. `set({ user: backendUser })` — replaces the whole user object in the Zustand store.

There is **no** optimistic store write and no Firestore involvement for this field. The location lives in Postgres (backend), persisted by `POST /secure/user/update/region-city`, and the client re-reads it through the same firebase-sync path used for all user hydration. After `refreshUser()` resolves, `useAuthStore().user.baseSite` and `.regionAndCity` are populated, so a second `AuthAddNewProductButton` click passes the gate at `:25`.

(In the normal flow the user never has to click twice, because `shouldOpenDialog` auto-advances — see §2. The refresh is what makes the *gate* itself pass on any subsequent evaluation.)

---

## 5. Trust boundary (conventions Part 11)

**What the web side establishes:**

The product-create request **does not carry location**. `NewProductRequestDTO` (`src/lib/types/product/NewProductRequestDTO.ts`) contains: `name`, `description`, `price`, `currency`, `topCategory`, `subCategory`, `finalCategory`, `filters`, `imageKeys`, `imagesData`. There is **no `baseSite`, `region`, or `city` field.** `createNewProduct` (`productService.ts:152–198`) POSTs this shape to `/secure/products/create` unchanged.

Therefore the product's base-site/region is **derived server-side from the authenticated user**, not supplied by the client at create time. The client-side gate in `AuthAddNewProductButton` is **UX only** — it stops a user from reaching a create form they can't complete; it is not the trust boundary.

**What this audit cannot confirm (backend-owned):**

Whether the backend *rejects* `POST /secure/products/create` when the authenticated user has no `baseSite`/`regionAndCity` set, and if so the error `code`, is **not determinable from this repo**. The web client has no handling for a specific "location-not-set" create error code — `createNewProduct` only branches on generic parseable validation statuses (400/422) via `parseProductErrorsForStatus`. If such a code exists it is not consumed by name on the web.

> **For the seam analysis:** this is the open question for the backend audit — does `/secure/products/create` enforce location-eligibility server-side (and with which code), or does it silently create a product with a null/partial location, trusting the client gate? Per Part 11 the server must be the trust boundary; the web gate alone is not sufficient. Mobile inherits the same exposure: if mobile skips the gate, the server behavior on a location-less create is what actually governs correctness.

---

## What mobile must replicate (platform-neutral behavior contract)

Mobile does not need the web dialog mechanism, the Next server action, the Zustand store, or the `shouldOpenDialog` plumbing. It needs to honor this behavior contract:

1. **Gate condition.** Before entering the create-product flow, check the authenticated user's profile. If **either** the user's base site **or** the user's region/city is missing, do not open the create flow — open a location-setup step first. If both are present, proceed directly to create.

2. **Eligibility precheck (optional but matches web).** Before showing the location picker, the user may be checked for assign-eligibility via `POST /secure/user/update/region-city/validate-update` (200 = allowed). For a brand-new user with no location this passes; the check primarily matters for the change-location case. If not allowed, surface the "cannot set location" message instead of the picker.

3. **What to collect, in order.**
   - First a **base site** (single choice) — from `GET /public/baseSite/overviews` (`BaseSiteOverviewDTO[]`; web shows only entries where `domain === code`).
   - Then a **region + city** — regions for the chosen base site come from `GET /public/baseSite/regions/{baseSiteCode}` (`RegionDTO[]`, cities nested). The user picks one region and one city, yielding `{ region: {id,labelKey}, city: {id,labelKey} }`.
   - Changing the base site must reset the region/city selection.

4. **What to persist.** A single call: `POST /secure/user/update/region-city` with body `RegionAndCityDTO` = `{ region, city }`. **The base site is not sent** — the server derives it from the region/city (and identity). Treat 2xx as success.

5. **Refresh after persist.** Re-fetch the authoritative user from the backend (web uses the `/auth/firebase-sync` hydration path) so the in-memory user now carries `baseSite` + `regionAndCity`. Do not optimistically fake the location locally; read it back from the server.

6. **Post-success behavior.** On successful persist, **auto-advance directly into the create-product flow** in the same interaction — do not make the user re-tap "add product". On failure, keep the setup step open and show an error.

7. **Trust note.** The product-create request carries no location; the server derives it from the user. The gate is UX. Mobile must not rely on the gate as a security/correctness boundary — the backend's behavior on a location-less create (reject vs. silently create) is the real boundary and is the open question for the backend audit (§5).
