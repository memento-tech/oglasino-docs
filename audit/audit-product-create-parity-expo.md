# Audit — Expo Product Creation Dialog (read-only)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Feature slug:** product-create-parity
**Type:** Phase 2 audit. READ-ONLY. No code changed.
**Date:** 2026-06-01
**Scope:** the product CREATE wizard only. The update/edit screen (`app/owner/dashboard/products/[productId].tsx`, `UpdateProductScreen`) is excluded by the brief and was not audited.

All findings are from the code on `new-expo-dev`. Where the brief states "web does X," that was treated as a question, not an answer.

---

## 1. Entry point and component tree

**What opens the dialog.** A single dialog id, `NEW_PRODUCT_DIALOG = 'addNewProductDialog'` (`src/components/dialog/dialogRegistry.ts:6`), is opened from three call sites:

- `src/components/navigation/BottomBar.tsx:143-147` — the floating blue "+" (`Plus`) button: `onPress={() => openDialogSafe(() => openDialog(DialogId.NEW_PRODUCT_DIALOG))}`.
- `src/components/dashboard/layout/DashboardSidebar.tsx:63` — the `PlusCircle` icon: `openDialog(DialogId.NEW_PRODUCT_DIALOG)` (opened directly, no wrapper).
- `app/(portal)/(public)/blog/free-zone.tsx:62` and `:118` — the free-zone CTAs: `!user ? openDialog(LOGIN_OPTIONS_DIALOG) : openDialog(NEW_PRODUCT_DIALOG)`.

**Pre-entry gate.** There is **no** pre-entry gate that checks the user has a base-site and region/city before opening the create flow. The only gate is an **auth** gate, and only on the BottomBar path:

```ts
// BottomBar.tsx:36-45
const openDialogSafe = (onAuthenticated, loginDescription?) => {
  if (!_hasHydrated) return;
  if (!user) {
    openDialog(DialogId.LOGIN_OPTIONS_DIALOG, { dialogDescription: loginDescription });
  } else {
    onAuthenticated();
  }
};
```

The DashboardSidebar and free-zone paths perform no base-site/region check at all (free-zone does its own inline `!user` auth check). There is **no** "you must set a region/city on your profile first" gate anywhere — the wizard itself collects region/city instead (see §4).

A base-site reconciliation happens *inside* the dialog after it opens (`AddUpdateProductDialog.tsx:70-83`): it picks `selectedBaseSite` if it matches the user's `user.baseSite` by code or domain, otherwise falls back to `user.baseSite`. This does not block opening; it only resolves which base-site the wizard uses.

**Component tree (mount order):**

```
DialogManager (src/components/dialog/DialogManager.tsx)
  → Modal → KeyboardAvoidingView → DialogWrapper (components/DialogWrapper.tsx)
      → ScrollView
        → AddUpdateProductDialog (dialogs/product-creation/AddUpdateProductDialog.tsx)   ← the wizard wrapper
            → (conditional) DialogTitleDescription (components/DialogTitleDescription.tsx)
            → (conditional) progress bar (inline View)
            → ScrollView
              → steps[currentStep].component, one of:
                  • ImageSelectionProductDialog (dialogs/product-creation/ImageSelectionProductDialog.tsx)
                  • BasicInfoProductDialog       (dialogs/product-creation/BasicInfoProductDialog.tsx)
                  • MetaDataProductDialog        (dialogs/product-creation/MetaDataProductDialog.tsx)
                      → MetaDataProduct (src/components/MetaDataProduct.tsx)
                  • UploadedProductDialog        (dialogs/product-creation/UploadedProductDialog.tsx)
```

Note the wizard wrapper file is named `AddUpdateProductDialog` but is registered **only** as the create dialog (`DialogManager.tsx:47` `addNewProductDialog: AddUpdateProductDialog`); the actual update flow is the separate `UpdateProductScreen` and does not mount this component. The name is a misnomer (flagged in "Drift / issues noticed").

---

## 2. Steps, in order

The wizard is **0-indexed** in state (`currentStep`, `AddUpdateProductDialog.tsx:32`, range 0–3) but each step also carries a **1-indexed** `no` field (`steps[].no`, values 1–4) used for the progress percentage and for the step-jump mapping. The four steps in mount order (`AddUpdateProductDialog.tsx:123-172`):

### Step index 0 (`no: 1`) — `ImageSelectionProductDialog`
- **Fields:** images only, via `ImagesImport` (`ImageSelectionProductDialog.tsx:48-54`), `maxNumberOfImages={5}`. Images are held in `productData.imagesData`.
- **Validation before advancing:** client-side only — `validateImagesData(productData)` (`productValidator.ts:91-120`): checks MIME (`image/jpeg|png|webp`), size (≤ 5 MB), and duplicate URIs. **No minimum count** — if `imagesData` is empty/undefined it returns `''` (a pass) and the step advances. No server call.
- **Can it be skipped / advance with zero images?** Yes. With zero images `validateImagesData` returns `''` and `onNextStep()` fires (`ImageSelectionProductDialog.tsx:33-42`).

### Step index 1 (`no: 2`) — `BasicInfoProductDialog`
- **Fields:**
  - **Category** — `CategorySelector` (top/sub/final), `required` (`BasicInfoProductDialog.tsx:170-178`).
  - **Name** — `Input`, `required`, `maxNumOfChars={80}`, with an AI-suggestion icon that appears once a name is typed (`:187-205`).
  - **Description** — `Textarea`, `required`, `maxNumOfChars={2000}` (`:208-217`).
  - **Price + Currency** — rendered **only** when `productData.topCategory && !productData.topCategory.freeZone` (`:223-253`). Price `Input` `required` `isNumericOnly`; currency `Select` over `selectedBaseSite.allowedCurrencies`.
  - **Region/City** — `CitySelector`, `required` (`:255-259`). See §4.
- **Validation before advancing (`onNextInternal`, `:97-135`):** both.
  1. Client structural: `validateProduct(productData, false)` (`productValidator.ts:25-89`) — name/description length, category presence, conditional price. Images not re-checked here (`validateImages=false`). If any of `allFieldsRequired / nameErrorKey / descriptionErrorKey / categoryErrorKey / priceErrorKey` is set, it returns without advancing.
  2. Server pre-validate: `classifyPreValidate(name, description)` (`preValidateOutcome.ts:43-63` → `productService.preValidateProduct` → `POST /secure/products/pre-validate`). `clean` → advance; `validation` → render inline name/description errors + optional `__system` rate-limit; `error` (transport/5xx/unparseable) → `notify.warning(...)` and advance anyway (pre-validate is treated as a UX optimization, not a boundary).
- **Skippable?** No — structural + pre-validate gate it. Region/city is `required` in the picker UI but its absence is **not** checked by `validateProduct` (see §4 / drift).

### Step index 2 (`no: 3`) — `MetaDataProductDialog`
- **Fields:** filters, via `MetaDataProduct` (`src/components/MetaDataProduct.tsx`). Available filters = `selectedBaseSite.catalog.topFilters` plus any category-level `filters` from top/sub/final category, minus `age` (`MetaDataProduct.tsx:33-49`). Renders RANGE / DATE / MULTI_OPTION / SINGLE_OPTION selectors (`:113-157`).
- **Validation before advancing:** **none.** The "forward" button calls `onNextStep` directly (`MetaDataProductDialog.tsx:45-47`). No client validation, no server call, no reCAPTCHA.
- **Skippable?** Effectively yes — nothing blocks advancing.

### Step index 3 (`no: 4`) — `UploadedProductDialog` (terminal)
- **Fields:** none — this is the upload/result step. On mount it auto-fires the create (`useEffect` → `uploadProductInternal`, `:155-157`). See §5–§7.
- **Validation:** server-side (the create call). No "next" — terminal controls are success ("View link" / "Finish") or failure ("Go back and fix" / "Exit").

**Progress bar** renders on every step except the last: `currentStep !== steps.length - 1` (`AddUpdateProductDialog.tsx:180`), width `(steps[currentStep].no / steps.length) * 100`.

---

## 3. The dialog title across steps

Yes, the wizard renders a title — `DialogTitleDescription` with `title={tDialog('new.product.title')}` — but it is gated to step index 0 only:

```tsx
// AddUpdateProductDialog.tsx:179
{currentStep === 0 && <DialogTitleDescription title={tDialog('new.product.title')} />}
```

- **Step index 0** (Image step): title **shown**.
- **Step index 1** (BasicInfo), **2** (Meta), **3** (Upload): title **not shown**.

So the title appears on exactly one step (the first, image step) and is absent on all three subsequent steps. `DialogWrapper` does not render a title of its own — it only provides the backdrop, the scroll container, and a close "X" (`DialogWrapper.tsx`), so there is no second title source. (Web, per the brief, shows the title on every step except the terminal one; mobile shows it on only the first — see Web-drift checks.)

---

## 4. Region and city

**Mobile asks for region/city in the wizard.** On the BasicInfo step it renders:

```tsx
// BasicInfoProductDialog.tsx:255-259
<CitySelector
  selectedRegionAndCity={productData.regionAndCity}
  onChange={(regionAndCity) => onChange({ regionAndCity })}
  required
/>
```

- **Pickable, not defaulted.** `CitySelector` (`src/components/basic/CitySelector.tsx`) is a full interactive picker: a modal `SectionList` of regions → cities sourced from `selectedBaseSite.regions` (`:47-65`), with search. The picked value is **editable** (the user taps to choose, and can "remove selected"); it is not display-only and not disabled (mobile passes no `disabled` prop, so it defaults to `false`).
- **Initial value is empty.** `productData.regionAndCity` is initialized to `undefined` (`AddUpdateProductDialog.tsx:97`). It does **not** default from the user's saved profile (`user.regionAndCity` is never read in the create flow).
- **`required` in the UI**, but note: the absence of a region/city is **not** enforced by `validateProduct` (`productValidator.ts` checks name/description/category/price/images only — no `regionAndCity` branch), so the `required` flag drives only the red-border styling in `CitySelector` (`:119`), not a hard block on advancing.
- **It is NOT sent in the create payload.** `toCreateWirePayload` (`productWirePayload.ts:15-30`) and the `CreateProductWireDTO` `Pick` (`CreateProductWireDTO.ts:13-25`) both exclude `regionAndCity` by allow-list. The comment at `productService.ts:77-81` explicitly notes `regionAndCity` is one of the fields that "cannot reach the wire."

Net: mobile **collects and requires** region/city in the wizard, then **drops it** — it never reaches the server. (This is the divergence flagged in `issues.md` 2026-06-01 item; see Web-drift checks and Drift/issues.)

---

## 5. The create request payload (trust boundary — Part 11)

- **Endpoint / method:** `POST /secure/products/create` (`productService.ts:109`, `createProduct`).
- **Builder:** `toCreateWirePayload(product, imageKeys)` (`productWirePayload.ts:15-30`), called from `uploadProduct` (`productService.ts:90`) after the image-upload step computes `imageKeys`. The caller's `productData` is never mutated; the payload is a fresh allow-listed object.
- **Body shape (`CreateProductWireDTO`):**

| Field | Sent? | Source / classification |
|---|---|---|
| `name` | yes | (a) user-entered (BasicInfo) |
| `description` | yes | (a) user-entered (BasicInfo; may be AI-suggested, still user-confirmed) |
| `price` | yes (string) | (a) user-entered (BasicInfo; only meaningful for non-free-zone categories) |
| `currency` | yes — **full `CurrencyDTO`** | (a) user-selected; pre-defaulted to `allowedCurrencies[0]` |
| `topCategory` | yes — **full `CategoryDTO`** | (a) user-selected (CategorySelector) |
| `subCategory` | yes — **full `CategoryDTO`** | (a) user-selected |
| `finalCategory` | yes — **full `CategoryDTO`** | (a) user-selected |
| `filters` | yes — **full `SelectedFilterDTO[]`** | (a) user-selected + (b) client pre-seeded defaults (delivery/condition/availability) |
| `imageKeys` | yes (`string[]`) | (b) derived client-side — R2 keys returned by the upload step |
| `regionAndCity` | **NO** | collected in UI but excluded from the wire (server is expected to derive from the authenticated user) |
| `free` | **NO** | excluded by allow-list (server-derived from category) |
| `imagesData` | **NO** | client-only; replaced by `imageKeys` |

- **Full objects vs IDs:** mobile sends **full DTOs** for `currency`, `topCategory`, `subCategory`, `finalCategory`, and each `filters` entry (`{ filter: FilterDTO, options: FilterOptionDTO[] }` or `{ filter, selectedRangeValue }`) — not bare IDs.
- **Trust-boundary note:** the allow-list narrowing is deliberate (`CreateProductWireDTO` is a `Pick`, structurally excluding `free`/`regionAndCity`/`imagesData`). No client-supplied "before" values exist on the create wire (there is no `oldName`/`oldDescription` on create; those live only on the update wire).

---

## 6. Success and failure handling

All terminal handling is in `UploadedProductDialog.tsx`. The create call auto-fires on mount (`:155-157`); the result is a discriminated `CreateOutcome` (`:47-55`).

- **Success (`kind: 'success'`, `:197-232`):** no automatic navigation. Shows a success title (`new.product.success.title` with `productName`), a copy-to-clipboard public product URL, and two buttons: **"View link"** → `viewProduct()` closes the dialog and `router.push(productRoute)` (in-app navigation to the product), and **"Finish"** → `onClose()`. On success it calls `triggerDashboardReload()` (`:128`, `useDashboardProductsStore`) so the owner product list refetches when next shown. No toast on success.
- **Per-field server errors (`kind: 'validation'`, `:248-276`):** built from `parseServiceError(error)` + `findSystemError(parsed.errors)` (`:141-143`). The parsing helper is **`parseServiceError`** (`src/lib/utils/parseServiceError.ts`). Errors render as a bulleted list of `tError(e.translationKey)`; a **"Go back and fix"** button jumps to `resolveTargetStep(outcome.byField)` (1-indexed step from `productStepMapping.ts`), and **"Exit"** closes. (The earlier BasicInfo pre-validate step surfaces name/description field errors inline via the same `parseServiceError`/`classifyPreValidate` path.)
- **Rate-limit (429):**
  - On **create**: a 429 (or any thrown error whose body parses to a `field: null` `__system` entry) routes to the `validation` outcome and renders the system error in the bullet list (`:258-262`).
  - On **pre-validate** (BasicInfo step): `classifyPreValidate` routes the 429 to `type: 'validation'` with a `systemError`; `BasicInfoProductDialog.startRateLimitBackoff` (`:72-81`) sets `rateLimited`, disables the "Next" button, renders the `__system` message under the button, and clears after `RATE_LIMIT_BACKOFF_MS = 5000` ms.
- **Image-upload failure (`kind: 'upload'`, `:234-246`):** an `UploadError` thrown by the upload phase is caught separately (`:132-138`); shows one localized toast via `buildUploadErrorTitle(error, tError)` (unless `CANCELLED`) plus a generic failure surface with a single **"back"** button. Not folded into the field-error list.
- **Generic failure (`kind: 'error'`, `:278-298`):** a throw with no parseable body → generic copy + "Go back and fix" (to `DEFAULT_FALLBACK_STEP = 3`) / "Exit".

---

## 7. Images

- **Collection:** held in memory through the wizard. `ImagesImport` writes into `productData.imagesData` (`ImageSelectionProductDialog.tsx:48-54`); the array persists across steps in the wizard's `productData` state (`AddUpdateProductDialog.tsx:68`, updated via `handleChange`).
- **When bytes hit R2:** only at the **terminal** step. `UploadedProductDialog` auto-fires `uploadProduct(productData, { mode: 'create', onProgress })` on mount (`:98-111`); inside `uploadProduct`, files with a `.file` slot are uploaded via `uploadImages(...)` (`productService.ts:64-71`) **before** the create POST. So no per-step upload — all selected images upload in one batch at the end, immediately ahead of `POST /secure/products/create`.
- **Upload function:** `uploadImages` (`src/lib/images/uploadImages.ts`), invoked through `productService.uploadProduct` (`src/lib/services/productService.ts:42-95`). Progress is reported into `useUploadProgressStore` and surfaced as stage labels in the dialog (`UploadedProductDialog.tsx:88-111`, `:188-194`). On a persist failure after a successful upload, `cleanupOrphanImages(newKeys)` runs before the rethrow (`productService.ts:92`).
- **Client-side minimum image count?** **None.** Step 0 advances with zero images (`validateImagesData` returns `''` for an empty/undefined array — `productValidator.ts:92-93`), and `validateProduct` on the BasicInfo step is called with `validateImages=false`, so it never checks images either. At least one image is enforced **only server-side** at create time.

---

## Web-drift checks

Each answer is mobile's actual behavior, then whether it matches the web reference as described in the brief.

- **Region/city.** Web: not asked; defaults from `useAuthStore().user.regionAndCity`, shown disabled in basic-info, not sent. **Mobile: DRIFTS.** Mobile *asks* via an editable, `required` `CitySelector` on the BasicInfo step (`BasicInfoProductDialog.tsx:255-259`), initialized empty (`AddUpdateProductDialog.tsx:97`), never sourced from `user.regionAndCity`. It is **not** sent in the payload (matches web on the wire), but the UI behavior diverges: mobile makes the user pick a region/city that is then discarded. Worst-of-both — extra friction with no effect.

- **Step title.** Web: shown on every step except the terminal upload/result step. **Mobile: DRIFTS.** Title shows on **step index 0 only** (`AddUpdateProductDialog.tsx:179`, `currentStep === 0`). Steps 1, 2, and 3 show no title. (Web would show it on steps 0–2 and hide on the terminal step 3.)

- **Image minimum.** Web: no client-side minimum; you can reach the final step with zero images, server rejects. **Mobile: MATCHES.** No client minimum (`validateImagesData` passes empty; BasicInfo skips image validation). Only the server enforces ≥1.

- **Category/currency/filters.** Web: sends full `CategoryDTO` / `CurrencyDTO` / `SelectedFilterDTO` objects. **Mobile: MATCHES.** `toCreateWirePayload` forwards full `currency`, `topCategory`, `subCategory`, `finalCategory`, and full `filters` objects (`productWirePayload.ts:19-29`), not IDs.

- **Currency default.** Web: pre-defaults to `selectedBaseSite.allowedCurrencies[0]`. **Mobile: MATCHES.** `AddUpdateProductDialog.tsx:93` initializes `currency: validBaseSite?.allowedCurrencies?.[0] || undefined`. (Additionally, `onCategoryChangeInternal` sets `currency: selectedBaseSite.defaultCurrency` if a non-free category is chosen and currency is somehow empty — `BasicInfoProductDialog.tsx:142-146`. `defaultCurrency` and `allowedCurrencies` both exist on `BaseSiteDTO`.)

- **Filter pre-seeding.** Web: pre-seeds default filters for delivery/condition/availability. **Mobile: MATCHES (pre-seeds).** `getInitialFilters()` (`AddUpdateProductDialog.tsx:34-66`) seeds `delivery` → `options[1]`, `condition` → `options[2]`, `availability` → `options[0]` from `selectedBaseSite.catalog.topFilters`, written into `productData.filters` at init (`:98`). (The specific option indices are mobile's own choice; whether they match web's seeded defaults is not verifiable here since web was not read.)

- **Pre-validate call.** Web: `POST /secure/products/pre-validate` (name+description) before advancing past basic-info. **Mobile: MATCHES.** `classifyPreValidate(name, description)` → `POST /secure/products/pre-validate` runs in `onNextInternal` on the **BasicInfo step (index 1)** before advancing to Meta (`BasicInfoProductDialog.tsx:118-131`, `productService.ts:135-146`). Sends `{ name, description }` only.

- **reCAPTCHA.** Web: runs reCAPTCHA on the metadata step before advancing. **Mobile: NONE.** `MetaDataProductDialog` advances with a bare `onNextStep` (`:45-47`); there is no reCAPTCHA (or any token) anywhere in the create flow. Consistent with RN having no browser reCAPTCHA — reported as absent, not flagged as a bug.

---

## Drift / issues noticed (out of scope)

1. **Region/city collected-but-discarded (medium).** `BasicInfoProductDialog.tsx:255-259` requires the user to pick a region/city that `toCreateWirePayload` then drops. Matches the open `issues.md` 2026-06-01 item ("region/city is asked even though the user already has a region and city set"). Bringing mobile to web parity means removing the picker and either defaulting display-only from `user.regionAndCity` or omitting it entirely. Note the `required` flag here is cosmetic only — `validateProduct` has no `regionAndCity` branch, so a user can advance without picking one anyway.

2. **Title only on step 0 (medium).** `AddUpdateProductDialog.tsx:179` gates the title to `currentStep === 0`. Matches the open `issues.md` 2026-06-01 item. Web shows it on every non-terminal step. To match web, the condition should be `currentStep !== steps.length - 1` (same predicate already used for the progress bar at `:180`).

3. **`AddUpdateProductDialog` is a misnomer (low).** The file/component is named for an add+update flow but is registered and used **only** as the create dialog (`DialogManager.tsx:47`); the update flow is the separate `UpdateProductScreen`. The shared `NewProductRequestDTO` / `uploadProduct({mode})` plumbing is genuinely shared, but the dialog component itself is create-only. A future rename would reduce confusion; not touched here.

4. **`getInitialFilters` uses fixed option indices (low).** `options[1]` / `options[2]` / `options[0]` (`AddUpdateProductDialog.tsx:39,51,60`) assume a stable option ordering from the catalog. If the backend reorders a top-filter's options, the pre-seeded default silently changes. Latent; not in scope. (Whether these indices match web's seeded defaults could not be checked — web was not read.)

5. **`free` is never set in the create flow (informational).** `NewProductRequestDTO.free` is initialized `undefined` (`AddUpdateProductDialog.tsx:100`) and never written, and it is excluded from the wire — so "free zone" status is entirely server-derived from the category. Noted for completeness; consistent with the trust-boundary design.

---

# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Task:** Phase 2 read-only audit — inventory the mobile product CREATION flow (new-product dialog/wizard), document it in sections 1–7 plus web-drift checks, for side-by-side comparison against the web reference.

## Implemented

- Read-only audit. No code changed. Documented the create wizard end to end: entry points, component tree, the four steps, the title gating, region/city handling, the create payload shape and trust boundary, success/failure handling, and the image-upload timing.
- Answered all eight web-drift checks from the brief against mobile's actual code (two drift, five match, one legitimately-absent).
- Flagged five out-of-scope observations, two of which match already-open `issues.md` items (region/city, step title).

## Files touched

- none — read-only audit. Output written to `.agent/audit-product-create-parity.md` and `.agent/last-session.md` only.

## Tests

- Ran: none (read-only; no code changed).
- Result: N/A
- New tests added: none

## Cleanup performed

- none needed (no code touched).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change. This audit is the Phase-2 input for `product-create-parity`; any backlog-table edit is Mastermind's to draft after Phase 3 seam analysis, not this read-only session's.
- issues.md: no change. The two parity defects this audit confirms (region/city collected-but-discarded; title only on step 0) are already logged under the 2026-06-01 "Mobile on-device UI/UX findings (batch)" entry; no new entry authored.

## Obsoleted by this session

- nothing (read-only).

## Conventions check

- Part 4 (cleanliness): confirmed — no code added, nothing to clean.
- Part 4a (simplicity): N/A — no code authored. Structured evidence in "For Mastermind."
- Part 4b (adjacent observations): five observations flagged in "Drift / issues noticed" / "For Mastermind."
- Part 6 (translations): N/A this session — no keys added or changed.
- Other parts touched: Part 11 (trust boundaries) — audited; the create wire excludes `free`/`regionAndCity`/`imagesData` by allow-list and carries no client "before" values. Part 8 (direct-to-R2) — confirmed images upload direct, batched at the terminal step.

## Known gaps / TODOs

- Web was not read (brief: mobile side only), so "matches web" answers for filter pre-seed option indices and exact title-step semantics are stated as mobile behavior vs the brief's described web behavior, not a code-to-code diff.
- none deferred in code (no code in scope).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code authored.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Drift summary for Phase 3 seam analysis:**
  - **Region/city (medium):** mobile asks + requires region/city in the wizard (`BasicInfoProductDialog.tsx:255-259`, `CitySelector`), inits empty, never sources from `user.regionAndCity`, and discards it (not on the wire). Web defaults display-only from the profile and also doesn't send it. The wire matches; the UX drifts. Note `required` is cosmetic (no `validateProduct` branch).
  - **Step title (medium):** mobile shows the title on step index 0 only (`AddUpdateProductDialog.tsx:179`); web shows it on every non-terminal step. One-line fix candidate: reuse the existing `currentStep !== steps.length - 1` predicate.
  - Both already have open `issues.md` rows (2026-06-01 batch) — no new draft needed; the parity work is the natural carrier.
  - **Component naming (low):** `AddUpdateProductDialog` is create-only despite the name; consider a rename when the parity brief touches it.
  - **Pre-seeded filter indices (low):** `getInitialFilters` hardcodes option indices (`options[1]/[2]/[0]`); brittle to catalog reordering. Confirm against web's seeded defaults during Phase 3.
- No config-file drafts produced this session. Closure gate: no implicit config-file dependency — none required.
