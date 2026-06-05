# Audit — ImageSourceSheet liveness check (READ-ONLY)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (no switch, no edits)
**Type:** read-only liveness audit
**Date:** 2026-05-31

---

## Question

Is `src/components/ImageSourceSheet.tsx` mounted/rendered anywhere in the live app, or is it dead code?

---

## 1. Every importer of `ImageSourceSheet`

Searched all `*.ts/*.tsx/*.js/*.jsx` (excluding `node_modules`).

| File:line | Kind | Real import? |
| --- | --- | --- |
| `src/components/ImageSourceSheet.tsx:14` | the `export function` declaration itself | n/a (definition) |
| `src/components/ImagesImport.tsx:21` | `import { ImageSourceSheet } from './ImageSourceSheet';` | **YES — real import** |
| `src/components/ImagesImport.tsx:214` | `<ImageSourceSheet … />` JSX render | **YES — real render** |

**One real importer:** `src/components/ImagesImport.tsx`. No comment or string-only mentions exist elsewhere.

---

## 2. Trace up — is the importer live?

### 2a. Inside `ImagesImport.tsx` — is the sheet actually opened?

`ImageSourceSheet` is rendered (`ImagesImport.tsx:214`) gated by local state `sheetOpen`:

- `const [sheetOpen, setSheetOpen] = useState(false);` — `ImagesImport.tsx:44`
- `setSheetOpen(true)` wired to a Pressable — `ImagesImport.tsx:146`
- `setSheetOpen(true)` wired to a second Pressable — `ImagesImport.tsx:190`
- `visible={sheetOpen}` passed to the sheet — `ImagesImport.tsx:215`
- `onClose={() => setSheetOpen(false)}` — `ImagesImport.tsx:216`
- `onCamera={pickFromCamera}` / `onGallery={pickFromGallery}` — `ImagesImport.tsx:217-218`

So the sheet is not merely mounted-but-dead: user taps on the import controls flip `sheetOpen` true and the Modal becomes visible.

### 2b. Who renders `ImagesImport`?

Three real render sites (excluding the definition and a doc-comment mention in `src/components/images/ImageStatusOverlay.tsx:9` and `src/lib/stores/uploadProgress.ts:2`, which are comments only):

| Render site | file:line |
| --- | --- |
| Owner product edit route | `app/owner/dashboard/products/[productId].tsx:1` (import), `:295` (render) |
| Product review dialog | `src/components/dialog/dialogs/ProductReviewDialog.tsx:1` (import), `:192` (render) |
| Image-selection step of product-creation dialog | `src/components/dialog/dialogs/product-creation/ImageSelectionProductDialog.tsx:1` (import), `:48` (render) |

### 2c. Are those three reachable?

1. **Route screen — confirmed reachable.** `app/owner/dashboard/products/[productId].tsx` is an expo-router route (owner dashboard, edit-a-product screen). `ImagesImport` is rendered unconditionally inside the `productDetails && (…)` block at `:291-302`, not commented out. This alone confirms reachability.
2. **ProductReviewDialog — confirmed reachable.** Registered in the dialog registry: `src/components/dialog/DialogManager.tsx:23` (import) and `:46` (`productReviewDialog: ProductReviewDialog`). Renders `ImagesImport` at `ProductReviewDialog.tsx:192`.
3. **ImageSelectionProductDialog — confirmed reachable.** Rendered by `AddUpdateProductDialog` (`AddUpdateProductDialog.tsx:12` import, `:128` render), which is itself registered in the dialog registry: `DialogManager.tsx:30` (import) and `:47` (`addNewProductDialog: AddUpdateProductDialog`).

---

## 3. Verdict

**LIVE — reachable via three independent chains:**

- `app/owner/dashboard/products/[productId].tsx:295` → `ImagesImport` → `ImageSourceSheet` (`ImagesImport.tsx:214`)
- `DialogManager` (`productReviewDialog`) → `ProductReviewDialog.tsx:192` → `ImagesImport` → `ImageSourceSheet`
- `DialogManager` (`addNewProductDialog`) → `AddUpdateProductDialog.tsx:128` → `ImageSelectionProductDialog.tsx:48` → `ImagesImport` → `ImageSourceSheet`

`ImageSourceSheet` is **not** dead code.

---

## 4. (LIVE follow-up) Hardcoded Serbian strings

Three hardcoded `sr` strings, plus a `//TODO TRANSLATIONS ADD` marker at `ImageSourceSheet.tsx:13`:

| file:line | Exact text | Intended meaning |
| --- | --- | --- |
| `ImageSourceSheet.tsx:23` | `Izaberite izvor slike` | "Choose image source" |
| `ImageSourceSheet.tsx:34` | `Kamera` | "Camera" |
| `ImageSourceSheet.tsx:42` | `Galerija` | "Gallery" |

(Note: "Izaberite" is imperative-plural/formal; the project's `sr` convention is sentence case — see `FRONTEND-TRANSLATION-KEYS-TODO.md` §1. A backend reviewer would set final wording at seed time.)

---

## 5. (LIVE follow-up) Does any existing seeded key match?

**Important caveat on method:** this app's translations are **backend-seeded and fetched at runtime** (`src/i18n/fetchNamespace.ts`, `useTranslations.ts`); there is no local JSON catalog of keys in this repo. So "does a key exist" can only be answered authoritatively against the backend seed (out of mobile's repo scope). What I *can* establish from the mobile side is: (a) which camera/gallery/source keys are already referenced in code, and (b) what the image-pipeline key catalog documents.

### 5a. Keys actually referenced in the mobile codebase (COMMON / INPUT / BUTTONS)

Grep of every `tCommon/tInput/tButtons(...)` call for image/camera/gallery/source:

| Key referenced | Namespace | file:line | Matches a sheet string? |
| --- | --- | --- | --- |
| `product.has.no.images` | COMMON | `ImagesCarousel.tsx:63` | no |
| `product.image.failed.to.load` | COMMON | `ImagesCarousel.tsx:79`, `ZoomableImage.tsx:182` | no |
| `image.max` | INPUT | `ImagesImport.tsx:55`, `ProductReviewImageImport.tsx:92` | no |
| `images.import` | BUTTONS | `ImagesImport.tsx:147,194` | no |

No referenced key carries a "choose image source" / "camera" / "gallery" meaning.

### 5b. Image-pipeline key catalog (`jobs/image_pipeline/FRONTEND-TRANSLATION-KEYS-TODO.md`)

The catalog of existing + planned image keys (§5 "already exist", §3/§4 new) lists: `image.max`, `image.too.big`, `image.duplicate`, `image.broke`, `image.not.good`, `images.holder.label`, `images.import`, `button.close.label`, `button.cancel.label`, plus the Phase 4–8 processing/error keys (`image.processing.*`, `image.uploading*`, `image.upload.failed`, `image.invalid`, etc.). **None of these is a camera/gallery/source-picker label.**

### 5c. Conclusion

From the mobile side there is **no existing key matching "Choose image source" / "Camera" / "Gallery"** — neither referenced in code nor documented in the image-pipeline catalog. Subject to the backend-seed caveat in §5, three **new** keys appear to be needed. Natural fit, following the established patterns:

- `BUTTONS` namespace for the two action labels (matches `images.import`, `images.holder.label`), e.g. `image.source.camera`, `image.source.gallery`.
- A sheet title — `INPUT` (matches the upload-input precedent) or `DIALOG` (it is a modal sheet), e.g. `image.source.title`.

This is a recommendation for the implementer/Mastermind, not an instruction I act on. **Backend (Docs/QA → Backend) owns final key names + seeding; mobile would swap the three inline strings for `t…(...)` calls once seeded.** No mobile-only key invention.

---

## Notes / scope

- This was a read-only audit. No files were edited; no branch switched; no commits.
- I could not query the backend translation seed directly (separate repo, not in mobile's reach). The §5 conclusion is therefore "no match found within mobile's reach" + the documented catalog — strong but not a substitute for a backend seed-table check.

## Config-file impact

None. This is a read-only audit; no `state.md` Expo-backlog row is adopted or retired by this session. If Mastermind decides to schedule the `ImageSourceSheet` i18n swap as work, that backlog entry is theirs to add — I draft nothing into the four config files.

## For Mastermind

- `ImageSourceSheet` is **LIVE** (three reachable chains, §3). Any plan that treated it as dead code should be revised.
- It carries a `//TODO TRANSLATIONS ADD` (`ImageSourceSheet.tsx:13`) and three hardcoded `sr` strings (§4). To fix per project i18n conventions, **three new backend-seeded keys** are needed (§5c) — this needs a backend seed step before mobile can swap the inline strings. Suggested owner sequence: Docs/QA records the keys → Backend seeds → Mobile swaps inline strings for `t…()` calls in a small follow-up.
