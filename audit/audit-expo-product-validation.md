# Audit — oglasino-expo product create/edit/validation surface (post-Φ4)

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Mode:** read-only. Code read as ground truth on disk; no prior audit, spec prose, or
session summary treated as authority for what the code *is*. The frozen contract in
`oglasino-docs/features/product-validation.md` `## Platform adoption` is the target
mobile must conform *to* — this audit does not assume the code already matches it.

---

## TL;DR — the load-bearing findings

1. **BLOCKER — mobile create AND edit still POST to the OLD endpoint `/secure/product/addUpdate`** (`src/lib/services/productService.ts:93`). The frozen contract's three routes (`/api/secure/products/create`, `/update`, `/pre-validate`) appear **nowhere** in the codebase. If the backend retired `addUpdate` in the validation refactor, mobile create/edit is **broken today**. (Backend route table to be confirmed by the backend chat — see "For Mastermind".)
2. **No pre-validate exists on mobile.** There is no second server round-trip; the only server contact on the create path is the final upload+POST.
3. **Service layer DID convert to surface-via-throw (Φ4).** `productService` throws on persistence failure; `parseServiceError` returns `{ errors, byField }` and never throws — both as expected. But **no product screen consumes `parseServiceError`** — backend validation errors are caught-and-toasted generically or not handled at all.
4. **Update flow violates trust boundary (conventions Part 11).** The full stored `UpdateProductRequestDTO` — including client-supplied `oldName`/`oldDescription` "before" values and immutable `productState`/`moderationState`/categories/`regionAndCity` — is spread onto the wire with no narrowing.
5. **Validation surface is 100% client-side and still on the prior `product.internal.*` keys under the frozen `VALIDATION` namespace.** 0 of 10 sampled `ERRORS`-namespace spec keys resolve.
6. **All three Bucket-1 bugs are present** (B13 wrong-field error key, B14 hardcoded banned-words list, B16 `console.error` calls).

The rebuild is effectively a from-scratch rewire of the create/edit submit path, not a patch: new endpoints, new DTO narrowing, server-driven error rendering, and a namespace/key migration.

---

## A. The create surface

### Entry point — who mounts the wizard

`AddUpdateProductDialog` is registered in the dialog manager, not mounted directly by
screens. It is keyed under the dialog id `addNewProductDialog`:

- `src/components/dialog/dialogRegistry.ts:6` — `NEW_PRODUCT_DIALOG = 'addNewProductDialog'`
- `src/components/dialog/DialogManager.tsx:29` (import), `:46` — `addNewProductDialog: AddUpdateProductDialog`

Callers open it via `openDialog(DialogId.NEW_PRODUCT_DIALOG)`:

- `src/components/navigation/BottomBar.tsx:144` — `onPress={() => openDialogSafe(() => openDialog(DialogId.NEW_PRODUCT_DIALOG))}`
- `src/components/dashboard/layout/DashboardSidebar.tsx:65` — `<Pressable onPress={() => openDialog(DialogId.NEW_PRODUCT_DIALOG)}>`
- `app/(portal)/(public)/blog/free-zone.tsx:62` and `:118` — the free-zone create path

### Files and actual step order on disk

All five files exist in `src/components/dialog/dialogs/product-creation/`. The orchestrator
`AddUpdateProductDialog.tsx` builds a `steps` array (`useMemo<Step[]>`, lines 117–166) and
renders `steps[currentStep].component` (line 184). `currentStep` is a 0-based index
(`useState(0)`, line 33); `handleNext`/`handlePrevious` step it (lines 109–115). Still a
4-step linear wizard, but **the order differs from the brief's stated pre-Φ structure** —
image selection is now step 1, not the middle/last step:

1. index 0 (`no:1`) — `ImageSelectionProductDialog` (`AddUpdateProductDialog.tsx:121-127`)
2. index 1 (`no:2`) — `BasicInfoProductDialog` (`:131-139`)
3. index 2 (`no:3`) — `MetaDataProductDialog` (`:143-151`)
4. index 3 (`no:4`) — `UploadedProductDialog` (`:155-162`)

Shared `productData` (type `NewProductRequestDTO`) is initialized in `AddUpdateProductDialog`
(lines 89–102) and threaded into every step via `handleChange`
(`setProductData((prev) => ({ ...prev, ...data }))`, lines 105–107).

So the pre-Φ "wizard in that order" survives in count but **not in order**: on disk it is
Images → Basic Info → Meta → Upload.

### Per-step: client validation and server calls

**Step 1 — `ImageSelectionProductDialog.tsx`**
- Client: `validateImagesData(productData)` (`productValidator.ts:213`), called in `handleNext` (`ImageSelectionProductDialog.tsx:34`). Checks per-image `file.fileSize > 5 * 1024 * 1024` → `image.too.big`, and duplicate URIs → `image.duplicate` (validator 222–233). **No MIME/content-type check.** Picker is `<ImagesImport ... maxNumberOfImages={5} />`.
- Server: **none.** Images held in memory in `productData.imagesData`.

**Step 2 — `BasicInfoProductDialog.tsx`**
- Client: `validateProduct(productData, false, regexData)` (`productValidator.ts:147`), awaited in `onNextInternal` (`:59`); second arg `false` skips image checks. Validates `name`/`description` heuristically (required, repeating chars, punctuation/emoji, ALL CAPS, keyword stuffing, banned words, promo, links, contacts — validator 158–192) and sets `errors.allFieldsRequired` (206–208). Advance gated on `errors.allFieldsRequired || errors.nameErrorKey || errors.descriptionErrorKey` (`:62`). `regexData` from `useConfiguration('validation.regex.*')` (46–53).
- **No pre-validate server call.** The only network call in this step is the optional AI helper `getOpenAiSuggestionForProduct({ title: productData.name })` (`:82` → `POST /secure/openai`, `openAiService.ts:9`), which fires only on the AI-suggest button tap, not on Next. **There is no `name`/`description` content moderation round-trip on mobile.**

**Step 3 — `MetaDataProductDialog.tsx`**
- Client: effectively none. `onNextStepInternal` clears a local `errorMessage` then calls `onNextStep()` (29–32); `errorMessage` is never set, so it is dead. Inner `<MetaDataProduct>` only mutates `filters`/category via `onChange` (no validation, no `BACKEND_API`).
- Server: **none.** (No reCAPTCHA step exists, unlike the web spec's step 3.)

**Step 4 — `UploadedProductDialog.tsx`**
- Auto-runs on mount: `useEffect(() => { uploadProductInternal(); }, [productData])` (92–94). **No submit button** — entering the step performs upload + create.
- Calls `uploadProduct(productData, { onProgress })` (`productService.ts:25`), which (1) uploads new files via `uploadImages(..., { scope: 'product' })`, (2) builds a fresh payload `{ ...productData, imageKeys: allKeys, imagesData: undefined }` (64–68), (3) POSTs via `addUpdateProductData` → `POST /secure/product/addUpdate` (`:93`). On post-upload persist failure it runs `cleanupOrphanImages(newKeys)` then rethrows (75–78).
- Success shows the product URL, sets `uploadSuccess`, calls `onFinish` (69–72). `UploadError` is caught once and shown via `buildUploadErrorTitle` (79–86).

**In-flow observations:** untracked bare `// TODO` at `UploadedProductDialog.tsx:161`; dead `errorMessage` state in `MetaDataProductDialog`; `validateProduct` writes `nameErrorKey` (not `descriptionErrorKey`) for the description link/contact branches (`productValidator.ts:189,191` — see F/B13).

---

## B. The edit surface

### Real path (did NOT move in Φ2)

`app/owner/dashboard/products/[productId].tsx` — default export `UpdateProductScreen`
(line 34). The only sibling in `app/owner/dashboard/products/` is `index.tsx` (the list).
No `edit/` segment, no `[productId]/edit.tsx`, no modal variant. `productId` comes from
`useLocalSearchParams<{ productId: string }>()` (43–45), parsed with `parseInt`.

### Load

In a `useEffect` keyed on `[productId]` (65–90): `getDashboardProductDetails(parseInt(productId))`
(import line 20; defined `productsSearchService.ts:36` → delegates to private `getProductDetails(productId, 'dashboard')`,
which maps scope `'dashboard'` to base URI `/secure/products` and produces
**`GET /secure/products?productId=${productId}`**, `BACKEND_API.get(uri)`, returns `res.data`).
Typed `Promise<UpdateProductRequestDTO>`.

On success the result is stored into **two** state slices — `productDetails` (working copy)
and `oldProductDetails` (baseline), from the same object reference (76–77) — and
`visibleImage` seeded from `data.imageKeys[0]` (79–81). On `!data` or a thrown error it sets
`error = tDash('product.fail')` (71–73, 82–86) and **`console.error(err)` at line 83** (see F/B16).

**Type-vs-runtime gap:** state is `useState<NewProductRequestDTO>()` (49–50) but the fetched
object is the wider `UpdateProductRequestDTO` (`id`, `oldName`, `oldDescription`,
`productState`, `moderationState`). The narrow annotation is a compile-time cast only — the
extra keys persist at runtime (load-bearing for D and G).

### Change detection (two independent client-side mechanisms)

1. **Deep-equal against baseline.** `validateProductInternal(withDeepEqual)` (96–139): `if (withDeepEqual && deepEqualTest(productDetails, oldProductDetails)) { toast 'product.unchanged.title'; return false; }`. `deepEqualTest` is `src/lib/utils/utils.ts:16` (recursive structural equality). Invoked only on Save (`handleUpdate` → `validateProductInternal(true)`, 142); Preview passes `false` (192).
2. **Massive-name-change guard.** `isMassiveChange(oldProductDetails.name, productDetails.name)` from `@/lib/validators/productUpdateNameValidator` (import 26; call 110–123). On a hit sets `nameErrorKey: 'product.internal.name.exessive.change'` (note the misspelling) and aborts. Compares against the in-memory `oldProductDetails.name`, **not** the DTO's `oldName`/`oldDescription` (which the screen never reads).

No client-supplied old-values diff is *computed* on device for the wire; detection is local. (But the `oldName`/`oldDescription` fields still ride the wire untouched — see G.)

### Submit

`handleUpdate` (141–189), on the Save button (`onPress={handleUpdate}`, 339):
`uploadProduct(productDetails, { onProgress })` (import 19, call 154) → same service path as
create → **`POST /secure/product/addUpdate`** with the full spread payload. Returns
`NewProductResponse | null` (`{ id, name }`). **No `id` is on the declared `NewProductRequestDTO`
wire type** — update identity rides along implicitly on the spread of the wider runtime object.

Error handling (168–188): truthy result → success toast; falsy/`null` → `product.update.fail.*`
toast; `UploadError` caught once (`CANCELLED` swallowed, else `buildUploadErrorTitle`);
non-`UploadError` rethrown. **No `parseServiceError` consumption — backend field errors are
not rendered inline; a 400/422 becomes a generic fail toast.**

### Client validation before submit

`validateProductInternal` runs `validateProduct(productDetails, true, regexData)` (images
validated here), then the massive-change guard, then (Save only) the deep-equal check.
City/region (`CitySelector disabled={true}`, 273) and category (`CategorySelector disabled={true}`,
316) are rendered **disabled** with no-op `onChange` — not editable, not separately validated,
but still carried in the payload.

---

## C. The service layer post-Φ4 (most important)

### 1. `parseServiceError.ts` — signature, shape, never-throws

`src/lib/utils/parseServiceError.ts`. Signature (line 40):
`export function parseServiceError(error: unknown): ParsedServiceError`. Return shape matches
the brief's expectation (23–26):

```ts
export type ParsedServiceError = {
  errors: ServiceFieldError[];
  byField: Record<string, ServiceFieldError>;
};
```

`ServiceFieldError = { field: string | null; code: string; translationKey?: string }` (9–13).
`errors` collects every wire entry in order (non-string `code` skipped, line 60; `field`
normalized to `null` when not a string, 63; `translationKey` included only when string, 65).
`byField` is first-error-per-field, excluding object-level (`field === null`) errors (69).

**Never throws — confirmed.** No `throw` statement. Non-contract input returns the neutral
`{ errors: [], byField: {} }`: non-object/null (43), non-array `data.errors` (46); access is
optional-chained (45) so a thrown non-axios value also yields empty. It does not touch i18n.

### 2. `productService.ts` — behavior + exact endpoints

`src/lib/services/productService.ts`.

**Error style: THROWS (Φ4).** Every persistence method is `try { … } catch (err) { logServiceError(…); throw err; }`. Return type annotations still read `Promise<… | null>` / `Promise<boolean>`, but the catch rethrows — `null`/`false` is never produced on error.

**Create / update — one combined method, OLD endpoint (LOAD-BEARING).** There is no separate
create / update / pre-validate method. Both create and edit funnel through `addUpdateProductData`
(89–99):

```ts
const res = await BACKEND_API.post('/secure/product/addUpdate', productData);
```

invoked from `uploadProduct` (line 74), which rethrows after orphan cleanup (73–78).

- **Create:** `addUpdateProductData` → `POST /secure/product/addUpdate`.
- **Update:** the same method, the same `POST /secure/product/addUpdate`.
- **Pre-validate:** **does not exist.** No `pre-validate` / `preValidate` call anywhere in `src/lib/services/`.
- **GET-by-id load:** not in `productService.ts`. It lives in `productsSearchService.ts:13/36` (`getDashboardProductDetails` → `GET /secure/products?productId=`). The only product GETs in `productService.ts` are `markAsSeen` (`GET /public/product/seen/<id>`, 83) and `getNumberOfViews` (`GET /public/product/views/<id>`, 134).

**The frozen contract's three routes are NOT adopted.** `/api/secure/products/create`,
`/update`, `/pre-validate` appear nowhere in the repo. Mobile points create/edit at the OLD
`'/secure/product/addUpdate'`. **If the backend has retired `addUpdate`, mobile create/edit is
broken today.** This must be confirmed against the backend's current route table before the
rebuild proceeds.

Other product methods (all throw): `activateProduct` → `GET /secure/products/activate?productId=`
(103); `deactivateProduct` → `GET /secure/products/deactivate?productId=` (113); `deleteProduct`
→ `DELETE /secure/products?productId=` (123).

### 3. The three carry-forward uncaught call sites (found by symbol; verified in current code)

**(a) `app/owner/dashboard/user.tsx` — `updateUser` (~old line 163): UNHANDLED.**
`updateUser` (`userService.ts:29`) rethrows on error (35). At `user.tsx:163`:
`const result = await updateUser({ … }).finally(() => setLoading(false));`. `saveChanges`
(80) has **no try/catch** around this awaited call — only `.finally()` clearing the spinner
(the try/catch at 131–156 wraps the *upload* block only). On reject, `saveChanges` rejects from
`onPress={saveChanges}` (336); the `if (!result)` branch (180) is dead because `result` is never
assigned on reject. No user-facing surface for the backend error; potential unhandled rejection.

**(b) `BasicInfoProductDialog.tsx` — `getOpenAiSuggestionForProduct` (line 82): UNHANDLED.**
`onAiSuggest` (80) calls it with **no try/catch**; `getOpenAiSuggestionForProduct` rethrows
(`openAiService.ts:13`). On reject, `setLoading(false)` (86) never runs — the `LoadingOverlay`
(99) leaks, leaving the dialog stuck under a spinner — and `onAiSuggest` rejects from
`onPress` (122). No catch, no toast.

**(c) `FollowUserButton.tsx` — `markFollowUser` (line 43): PARTLY handled / effectively UNHANDLED.**
`onFollowUser` (32) wraps the call in `try` (42) / **`finally`** (55) with **no `catch`**.
`markFollowUser` rethrows (`followService.ts:10`). The `finally` only re-enables the button; on
a real thrown error the `result === undefined` branch (46, which handles only a missing-`following`
*success* body) is never reached, and the promise rejects up through `onPress` (63). The error is
not surfaced.

None of the three call `parseServiceError`.

---

## D. Wire DTO types

**Serialization does NO field stripping.** `uploadProduct` builds the body via a blind spread
`{ ...productData, imageKeys: allKeys, imagesData: undefined }` (`productService.ts:64-68`) and
`addUpdateProductData` posts it verbatim (`:93`). There is **no `toWirePayload`, no whitelist, no
serializer** on the product path. Therefore **whatever keys live on the runtime object are sent**,
regardless of the static annotation.

### Fields on each DTO

`NewProductRequestDTO` (`src/lib/types/product/NewProductRequestDTO.ts:7-20`): `name: string`,
`description: string`, `price: string`, `currency: CurrencyDTO`, `free: boolean`,
`topCategory: CategoryDTO|undefined`, `subCategory: CategoryDTO|undefined`,
`finalCategory: CategoryDTO|undefined`, `regionAndCity: RegionAndCityDTO|undefined`,
`filters: SelectedFilterDTO[]|undefined`, `imageKeys?: string[]|undefined`,
`imagesData: ImageData[]|undefined`.

`UpdateProductRequestDTO extends NewProductRequestDTO` (`UpdateProductRequestDTO.ts:5-11`) — all
of the above **plus** `productState?: ProductState`, `moderationState?: ModerationState`,
`id: number`, `oldName: string`, `oldDescription: string`.

### Forbidden fields — present vs sent

| Forbidden field | On the type? | Sent on the wire? |
|---|---|---|
| `oldName` | On `UpdateProductRequestDTO` (`:9`); absent from create DTO | **CREATE: absent. UPDATE: SENT** (returned by `getDashboardProductDetails`, never stripped). |
| `oldDescription` | On `UpdateProductRequestDTO` (`:10`); absent from create DTO | **CREATE: absent. UPDATE: SENT.** |
| `productState` | On `UpdateProductRequestDTO` (`:6`); absent from create DTO | **CREATE: absent. UPDATE: SENT** if backend populated it. |
| `moderationState` | On `UpdateProductRequestDTO` (`:7`); absent from create DTO | **CREATE: absent. UPDATE: SENT** if backend populated it. |
| `free` | **On `NewProductRequestDTO`** (`:12`) — so also on update | **CREATE: SENT** (key present on the object, serialized; init `undefined` at `AddUpdateProductDialog.tsx:101`). **UPDATE: SENT.** |
| `regionAndCity` | **On `NewProductRequestDTO`** (`:16`) | **CREATE: SENT** (`CitySelector onChange`, `BasicInfoProductDialog.tsx`). **UPDATE: SENT** (disabled selector, value persists). |
| `topCategory` (update-only forbidden) | Inherited (`:13`) | **UPDATE: SENT** — disabled (`[productId].tsx:316`), but stored value echoed back, not omitted. |
| `subCategory` (update-only forbidden) | Inherited (`:14`) | **UPDATE: SENT.** |
| `finalCategory` (update-only forbidden) | Inherited (`:15`) | **UPDATE: SENT.** |

**Net:** create sends `free` + `regionAndCity` (the other four create-forbidden fields are
correctly absent because they're not on `NewProductRequestDTO`); update sends **all nine**
forbidden fields. A rebuild needs an explicit wire-payload whitelist in `productService` (or
distinct create/update serializers).

### `productUpdateNameValidator` / `isMassiveChange`

`src/lib/validators/productUpdateNameValidator.ts` **still exists** and exports `isMassiveChange`
(line 1) plus `similarityRatio`, `levenshteinDistance`, `hasWordOverlap`, `isLengthChangeTooBig`.
**It has exactly one live caller:** `app/owner/dashboard/products/[productId].tsx` (import `:26`,
call `:113`). No other importer in the repo. The massive-name-change check is therefore performed
**on-device**, not delegated to the backend — contrary to the contract, where `NAME_MASSIVE_CHANGE`
(422) is a server decision.

---

## E. Translation keys

### Namespace + key pattern in use TODAY

The create/edit validation surface is **entirely client-side** and still emits the prior-state
**`product.internal.*` keys under the frozen `VALIDATION` namespace** — NOT the contract's
`ERRORS` namespace with `product.<field>.<code>` keys.

`productValidator.ts` emits (name, `errors.nameErrorKey`): `product.internal.name.required` (159),
`…abuse` (161), `…emoji` (163), `…big.letters` (165), `…keyword.stuffing` (167),
`…forbidden.words` (169), `…promo.words` (171), `…links` (173), `…contacts` (175). Description
(`errors.descriptionErrorKey`, plus the two B13-mislabeled ones at 189/191):
`product.internal.description.required` (183), `…forbidden.words` (185), `…spam` (187),
`…links` (189, written to `nameErrorKey`), `…contacts` (191, written to `nameErrorKey`). Image
(`validateImagesData`): bare `image.too.big` (223), `image.duplicate` (229).

`AddUpdateProductErrors` (`src/lib/types/product/AddUpdateProductErrors.ts`) has only
`nameErrorKey`, `descriptionErrorKey`, `imagesErrorKey`, `allFieldsRequired` — **no price or
category error slot exists.** There is no category or price content validation in the validator
at all.

**Rendering:** `BasicInfoProductDialog.tsx` uses `useTranslations(TranslationNamespace.VALIDATION)`
(41) for name/description/images errors (117, 139, 187) plus hardcoded `'form.incomplete'` (197).
`ImageSelectionProductDialog.tsx` renders the carousel image errors via
`useTranslations(TranslationNamespace.COMMON_SYSTEM)` (20, 34–36). `UploadedProductDialog.tsx`
uses `ERRORS` (35) only for upload failures via `buildUploadErrorTitle` (81), which maps backend
image codes to dotted `image.*` keys (`src/lib/images/errorMapping.ts`). The edit screen uses
`VALIDATION` (36) for name/description (239, 254) and sets `product.internal.name.exessive.change`
(116).

`grep` confirms **15 `product.internal` occurrences** across `src`/`app`, and **zero** occurrences
of the contract's underscore pattern (`product.<field>.banned_words` etc.) on the create/edit
surface. The only frozen-key literal anywhere is in `parseServiceError.test.ts`
(`product.name.banned_words`); `parseServiceError` is wired only into `ReportDialog.tsx`, never
into product create/edit.

### How mobile fetches/stores translations

`TranslationNamespace` (`src/i18n/types.ts`) includes both `ERRORS` (8) and `VALIDATION` (9).
`src/i18n/fetchNamespace.ts` fetches per-namespace: `GET /public/translations?namespace=<ns>&lang=<lang>`,
flattening dotted keys via `toNested`. `bootStore.ts` Gate 4 (~447–559) iterates
`Object.values(TranslationNamespace)`, refetches stale slices, and registers each via
`i18n.addResourceBundle(activeLang, ns, payloads[ns], true, true)` (548). So **both `ERRORS` and
`VALIDATION` bundles load** — resolution depends purely on which namespace the render call targets
and whether the backend seeds that exact key.

### 10 spec keys — do they resolve against what mobile emits?

The surface emits `product.internal.*` (VALIDATION) and `image.too.big`/`image.duplicate`
(COMMON_SYSTEM). It emits **none** of the contract's `ERRORS` keys, so none can resolve:

| Spec key (ERRORS) | Resolves? | Why |
|---|---|---|
| `product.name.required` | NO | emits `product.internal.name.required` (VALIDATION) |
| `product.name.too_long` | NO | no length check; closest is `…name.abuse` |
| `product.name.banned_words` | NO | emits `product.internal.name.forbidden.words` |
| `product.description.required` | NO | emits `product.internal.description.required` |
| `product.description.contains_link` | NO | emits `product.internal.description.links` (and to the wrong field — B13) |
| `product.category.required` | NO | no category validation / no error slot |
| `product.price.required` | NO | no price validation / no error slot |
| `product.image.invalid_type` | NO | no content-type check; size/dup only |
| `product.image.too_big` | NO | emits bare `image.too.big` |
| `product.system.rate_limited` | NO | no `product.system.*` key anywhere |

**Result: 0 of 10 resolve.** Mobile is still on the prior `product.internal.*` / `VALIDATION`
state. Adoption requires either re-pointing the client validator's emitted keys + namespace
(`VALIDATION` → `ERRORS`, dotted-code shape) or replacing client field validation with
server-driven errors via `parseServiceError`. (The backend-driven `image.*` upload errors in
`errorMapping.ts` are a separate `ERRORS` path and use neither the `product.image.*` nor the
`product.system.*` shape.)

---

## F. Bucket-1 bugs (confirmed present in current code)

### B13 — description link/contact errors written to `nameErrorKey` (PRESENT)

`src/lib/validators/productValidator.ts:188-191`:

```ts
} else if (containsLink(productData.description)) {
  errors.nameErrorKey = 'product.internal.description.links';
} else if (containsContacts(productData.description)) {
  errors.nameErrorKey = 'product.internal.description.contacts';
}
```

The two branches just above (184–187) correctly target `descriptionErrorKey`. So a description
link/contact error renders under the **name** field. Still at lines 189 and 191 (the brief's
reported line numbers held).

### B14 — hardcoded `BANNED_WORDS` array (PRESENT)

`src/lib/validators/productValidator.ts:7-70` — `const BANNED_WORDS = [...]`, **62 entries**
(EN + SR/BS profanity/contraband; `heroin` is duplicated at lines 24 and 53). Consumed by
`containsBannedWords` (119–122), which takes a `bannedWords: string` parameter (passed
`regexData.bannedWords` at call sites 168 and 184) but **ignores it entirely**, hardcoding
`BANNED_WORDS`. So the backend-configured banned-words list is fetched into `regexData` and then
discarded; the on-device hardcoded list wins.

### B16 — `console.*` violations on this surface (PRESENT, 3)

- `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:83` — `if (__DEV__) console.error(error);` (DEV-guarded)
- `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx:104` — `console.error('Copy failed', e);` (unguarded)
- `app/owner/dashboard/products/[productId].tsx:83` — `console.error(err);` (unguarded)

No `console.*` in `productValidator.ts` or `productService.ts`.

---

## G. Trust boundaries (conventions Part 11)

**Shared transport for both flows.** Create and update both go through `uploadProduct` →
`addUpdateProductData` → `POST /secure/product/addUpdate`, with a blind spread `{...productData}`
and **no `toWirePayload`/omit/pick narrowing** (grep returned none). Every enumerable property on
the caller's runtime object reaches the wire; the static `NewProductRequestDTO` annotation does
not strip runtime keys. The audit therefore reduces to "what is on the runtime object each caller
passes in."

### CREATE flow

The object originates in `AddUpdateProductDialog.tsx` (`setProductData({...})`, ~89–102), mutated
only via `handleChange` and field `onChange` handlers. Seeded with exactly the
`NewProductRequestDTO` keys — no `id`, `productState`, `moderationState`, or any `old*` value.

| Field | Source | Classification |
|---|---|---|
| `name`, `description`, `price` | user input | trusted-from-client |
| `currency` | user picks from `allowedCurrencies` | trusted-from-client (server re-validates) |
| `free` | derived from category in dialog | trusted-from-client |
| `topCategory`/`subCategory`/`finalCategory` | user selection | trusted-from-client |
| `regionAndCity` | user selection (`CitySelector`) | trusted-from-client |
| `filters` | user selection | trusted-from-client |
| `imageKeys` | upload result (`allKeys`) | derived |
| `imagesData` | forced `undefined` | not sent |

**CREATE verdict: clean.** Named checks: (1) seed object holds only `NewProductRequestDTO` keys;
(2) no `old*`/`before*` value present; (3) no `id`/`productState`/`moderationState` introduced on
the create path. (Note: `free` and `regionAndCity` are sent and the contract says create should
*not* send them — but that is a DTO-shape/contract-conformance issue, not a Part 11 trust-boundary
violation: neither is a client-supplied "before" value used in a server moderation/state decision.)

### UPDATE flow

`[productId].tsx` fetches `getDashboardProductDetails` → `UpdateProductRequestDTO` (carrying
`id`, `oldName`, `oldDescription`, `productState`, `moderationState`). The **entire fetched DTO**
is stored as `productDetails` (76) — declared `useState<NewProductRequestDTO>()` (50), so the
extra keys are invisible to tsc but present at runtime — then handed to `uploadProduct(productDetails, …)`
(154). The blind spread emits all of them.

| Field | On wire? | Classification |
|---|---|---|
| `name`, `description`, `price`, `currency`, `free`, `filters` | yes | trusted-from-client |
| `topCategory`/`subCategory`/`finalCategory` | yes | **immutable leaked** (disabled selectors, original values echoed) |
| `regionAndCity` | yes | **immutable leaked** (disabled selector) |
| `id` | yes | trusted identifier (row identity) |
| `productState` | yes | **immutable/state field leaked** |
| `moderationState` | yes | **immutable/moderation field leaked** |
| `oldName` | yes | **VIOLATION — client-supplied "before" value for change detection** |
| `oldDescription` | yes | **VIOLATION — client-supplied "before" value for change detection** |
| `imageKeys` | yes (`allKeys`) | derived |
| `imagesData` | no (`undefined`) | not sent |

`oldName`/`oldDescription` are read-from-DB on the fetch but echoed back unchanged in the update
body — exactly the forbidden "client supplies the previous value so the server can diff" pattern
Part 11 names. The server must compare incoming `name` against its own stored version. (The
screen's own change detection at 112–113 uses the separate `oldProductDetails.name` state, not the
DTO's `oldName` — so `oldName`/`oldDescription` serve no client purpose and exist only to be sent.)

**UPDATE verdict: violation.** Named checks: (1) `getDashboardProductDetails` returns
`UpdateProductRequestDTO` carrying `oldName`/`oldDescription`; (2) the full DTO is stored and
passed to `uploadProduct` with no stripping; (3) `uploadProduct` builds the body via `{...productData}`
with no `toWirePayload`; (4) therefore `oldName`/`oldDescription` (client "before" values) plus the
immutable `categories`/`regionAndCity`/`productState`/`moderationState` all reach the wire.

**Remediation direction (not implemented):** an explicit allow-list narrowing before POST — keep
`id` + genuinely editable fields, strip `oldName`/`oldDescription`/`productState`/`moderationState`
and the disabled immutable fields. Belongs in a shared `toWirePayload` in `productService.ts` (or by
not typing the dashboard fetch as the wire DTO). Couples to the backend `addUpdate` → three-route
migration in C.

---

## Cross-section synthesis for the rebuild

The current mobile surface is **pre-contract on every axis the spec freezes**:

- **Endpoints:** one combined `/secure/product/addUpdate`, no pre-validate, no GET-by-id in the product service → must split into `create` / `update` / `pre-validate` and adopt the new paths (gated on backend confirmation that `addUpdate` is retired).
- **DTOs:** no narrowing; create leaks `free`+`regionAndCity`, update leaks all nine forbidden fields → needs `toWirePayload` allow-lists (or distinct create/update DTOs without the inherited/`old*` fields).
- **Errors:** client-only `product.internal.*` under `VALIDATION`, 0/10 contract keys resolve, no `parseServiceError` consumption on the product surface → needs a key/namespace migration to `ERRORS` + `product.<field>.<code>` and server-driven error rendering via `parseServiceError`/`byField`.
- **Validation sequence:** no pre-validate round-trip; no category/price/image-MIME structural checks; massive-change done on-device → needs the structural → pre-validate → submit sequence and removal of the on-device `isMassiveChange`.
- **Rides along:** B13, B14, B16, the dead `MetaDataProductDialog.errorMessage`, the untracked `// TODO`, the `product.internal.name.exessive.change` misspelling — the rebuild deletes these.

This is a rewrite of the submit/error path, not a patch.
