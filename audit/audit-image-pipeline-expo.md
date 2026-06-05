# Audit — image-pipeline expo Phase 2 (READ-ONLY)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (confirmed via `git branch --show-current` → `new-expo-dev`; no branch switch performed)
**Type:** read-only. No edits, no commits, no staging. Code on disk only; no pre-existing reports/summaries read.
**Date:** 2026-05-31

Every "mobile currently does X" claim cites file:line. "Not found" / "zero callers" written where something genuinely isn't present.

---

## 1. Progress text in the product upload dialog

**The mobile equivalent of `UploadedProductDialog` is `src/components/dialog/dialogs/product-creation/UploadedProductDialog.tsx`** — it mounts as step 4 (the final step) of the create wizard (`AddUpdateProductDialog.tsx:159-169`).

**Where upload progress is rendered, and what it surfaces:**
- The dialog renders **only a spinner** while uploading: `UploadedProductDialog.tsx:182-186` — `{loading && (<View …><ActivityIndicator size={40} color={'green'} /></View>)}`. There is no per-stage text ("Checking…", "Resizing…", "Compressing…", "Uploading…", "Done") rendered anywhere in this component.

**Whether the pipeline already emits per-stage progress the dialog could subscribe to — YES:**
- The dialog already drives the per-file progress store: it reads `useUploadProgressStore.getState()` (`UploadedProductDialog.tsx:84`), calls `progress.start(fileNames)` (line 89), and forwards an `onProgress(fileIndex, state)` callback into `uploadProduct` that calls `progress.setState(...)` with the live `stage` value (`UploadedProductDialog.tsx:96-107`), then `progress.finish()` in `finally` (line 147).
- The store carrying these stages is `src/lib/stores/uploadProgress.ts` — `useUploadProgressStore`, with `UploadFileStage` (lines 19-25) = `'idle' | ProcessingStage | 'uploading' | 'complete' | 'error' | 'cancelled'`, and `states: Record<number, UploadFileState>` (line 38) holding per-file `stage`, `fileName`, `originalSize`, `processedSize`.
- So the data is fully plumbed into the store during the step-4 upload, but the dialog imports the store only to **write** to it (`UploadedProductDialog.tsx:11-14`) — it does **not** subscribe to `states` to render any text. The store IS subscribed and rendered elsewhere (`ImageStatusOverlay.tsx`, see below), but those consumers live on the picker step, not on the upload step. During step 4 the user sees only the spinner.

**Stage→label mapping helper — EXISTS:**
- `src/lib/images/errorMapping.ts:51-85` — `stageLabel(tInputs, stage, params)`. The key rule is `${STAGE_KEY_PREFIX}${STAGE_KEY_OVERRIDES[stage] ?? stage}` where `STAGE_KEY_PREFIX = 'image.processing.'` (line 27) and `STAGE_KEY_OVERRIDES` (lines 35-43) = `{ uploading: 'uploading.label', complete: 'complete.label', converting_heic: 'converting_heic' }`.
- This matches web's documented behavior: `'image.processing.' + stage`, with `uploading`→`.label`, `complete`→`.label`, and the hyphenated `converting-heic` stage value mapping to the underscored seeded segment `converting_heic`. Parameterized siblings `uploading.with.size` / `complete.with.sizes` handled at lines 57-69. English last-resort fallback at `englishStageLabel` (lines 87-112).
- The helper is already consumed by `src/components/images/ImageStatusOverlay.tsx:62` (`stageLabel(tInputs, state.stage, {...})`, with `tInputs = useTranslations(TranslationNamespace.INPUT)` at line 56) — i.e. the per-stage text renderer exists and works; the upload dialog simply doesn't use it.

**`image.processing.*` keys reachable on mobile — YES:**
- `stageLabel` reads them through the **INPUT** namespace (`ImageStatusOverlay.tsx:56`). INPUT is a member of the namespace enum (`src/i18n/types.ts:16` — `INPUT = 'INPUT'`), and boot fetches **every** enum member (`src/lib/store/bootStore.ts:451` — `for (const ns of Object.values(TranslationNamespace))`). So INPUT is in mobile's fetch set and the backend-seeded `image.processing.*` keys resolve.

**Verdict:** Still a problem. The dialog surfaces only a spinner (`UploadedProductDialog.tsx:182-186`). The store, the per-stage events, the `stageLabel` helper, and the INPUT-namespace key resolution are all already present — the dialog just doesn't subscribe to `useUploadProgressStore.states` and render `stageLabel(...)`. Wiring is small and self-contained.

---

## 2. B15 — two hardcoded Serbian strings

### 2a. `src/components/ImageSourceSheet.tsx` — EXISTS, hardcoded Serbian

File present. Hardcoded Serbian, not translation-key lookups (the component imports no `useTranslations`; line 13 carries `//TODO TRANSLATIONS ADD`):
- `ImageSourceSheet.tsx:23` — `Izaberite izvor slike` ("Choose image source")
- `ImageSourceSheet.tsx:34` — `Kamera` ("Camera")
- `ImageSourceSheet.tsx:42` — `Galerija` ("Gallery")

So this is **three** hardcoded strings in this file, not one. Already-seeded `image.*`/`images.*` candidates to check against: none of `images.holder.label`, `images.import`, `image.max` map cleanly to "Choose image source"/"Camera"/"Gallery" — these would likely need new keys (or reuse of generic camera/gallery keys if they exist; I did not find an exact seeded match by grep). **Note (closure / liveness):** I did not find any importer of `ImageSourceSheet` in `src`/`app` (grep for `ImageSourceSheet` returns only its own definition). If it is also unused, B15-2a may itself be a dead-code question rather than a string-fix — flagging for the implementation brief to confirm its live status before minting keys.

### 2b. `src/components/dialog/components/ProductReviewImageImport.tsx` — EXISTS, hardcoded Serbian — but the file is DEAD (see item 4)

File present. Multiple hardcoded Serbian strings (component imports `useTranslations` only for `image.max`; the rest are literals; line 21 carries `//TODO TODO`):
- `ProductReviewImageImport.tsx:40` — `Dodavanje slika`
- `ProductReviewImageImport.tsx:40` — `Izaberite način dodavanja slika`
- `ProductReviewImageImport.tsx:41` — `Kamera`
- `ProductReviewImageImport.tsx:42` — `Galerija`
- `ProductReviewImageImport.tsx:43` — `Otkaži`
- `ProductReviewImageImport.tsx:58` — `Dozvola za kameru nije odobrena`
- `ProductReviewImageImport.tsx:73` — `Dozvola za galeriju nije odobrena`
- `ProductReviewImageImport.tsx:106` — `Dodajte slike (opciono)`
- `ProductReviewImageImport.tsx:146-149` — `Možete dodati do 5 slika koje prikazuju proizvod, komunikaciju ili dodatne detalje koji potvrđuju vaše iskustvo.\nSlike treba da budu relevantne i pristojne.`

The single non-hardcoded string here is `image.max` (line 92, `tInput('image.max', …)`).

**Reconciliation with item 4:** `ProductReviewImageImport.tsx` has **zero importers** (see item 4 for the grep). The live review-image path is `ImagesImport`, mounted by `ProductReviewDialog.tsx:1` (import) and `:192` (`<ImagesImport …/>`). Therefore **do NOT fix the strings in `ProductReviewImageImport.tsx` — delete the file** (item 4). Its B15 strings disappear with it.

**Verdict (item 2):** The spec's framing of "two files, one string each" is inaccurate against the code. The live B15 surface is `ImageSourceSheet.tsx` (3 strings, lines 23/34/42) — fix-the-string candidate (after confirming the component is actually mounted). `ProductReviewImageImport.tsx` is dead — delete-the-file, not fix-the-string.

---

## 3. Dead validator `productUpdateNameValidator.ts`

- File at the named path: **not found.** `find … -name productUpdateNameValidator.ts` returns nothing.
- References anywhere in the repo (`grep -rn "productUpdateNameValidator\|ProductUpdateNameValidator"` over `*.ts`/`*.tsx`): **zero hits.**

**Verdict:** Not found — there is no such file and zero references. Nothing to delete; drop this item.

---

## 4. Ride-along dead code

### 4a. `src/components/dialog/components/ProductReviewImageImport.tsx`

Importers (`grep -rn "ProductReviewImageImport"` over `src`/`app`):
- `src/components/images/ImageStatusOverlay.tsx:10` — a **comment** ("…AND ProductReviewImageImport.tsx — same JSX both places…"), not an import.
- `src/lib/stores/uploadProgress.ts:2` — a **comment** listing consumers, not an import.
- `src/components/dialog/components/ProductReviewImageImport.tsx:22` — its own `export default function`.

No `import … ProductReviewImageImport` anywhere. **Zero callers.**

The live review-image path: `ProductReviewDialog.tsx` mounts **`ImagesImport`** — `import ImagesImport from '@/components/ImagesImport'` (`ProductReviewDialog.tsx:1`), rendered at `ProductReviewDialog.tsx:192-198`. It does NOT mount `ProductReviewImageImport`.

**Verdict:** Zero callers, safe to delete. (Deleting it also resolves B15-2b — do not double-handle.)

### 4b. `app/__smoke__/upload.tsx`

- File present (`app/__smoke__/upload.tsx`); it is a smoke/test harness — header docblock: "Day-one smoke harness for the image upload mechanism… To delete after verification: `rm -r app/__smoke__/`" (`upload.tsx:1-90`). Default export `UploadSmokeScreen` (line 114).
- Importers / route registration (`grep -rn "__smoke__\|UploadSmokeScreen\|smoke/upload"`): no code `import`s it. The only references are its own docblock lines (`upload.tsx:41,59,60`) and one comment pointer in `src/lib/images/uploadPrimitive.ts:5`.
- **Caveat — it is still a reachable route.** Because it lives under `app/` it is auto-registered by expo-router's file-based routing as the screen `__smoke__/upload` (deep-linkable as `oglasino://__smoke__/upload`, per its own docblock lines 59-60). So "zero importers" is true, but it is not unreachable: it ships as a navigable route in any build that includes `app/`.

**Verdict:** Zero importers; it is a self-contained test harness reachable only via the file-based route. Safe to delete (`rm -r app/__smoke__/`), per its own header instruction, once the device wire-shape smoke is no longer needed. Not dead in the "unreachable" sense — it is a deliberately-routable harness — so deletion is a cleanup decision, not a correctness fix.

---

## 5. Duplicate `isPngInput`

Both definitions exist:

- `src/lib/images/processImage.ts:228-232`
  ```ts
  function isPngInput(input: ProcessImageInput): boolean {
    if (input.mimeType === 'image/png') return true;
    const lower = input.uri.toLowerCase().split('?')[0];
    return lower.endsWith('.png');
  }
  ```
- `src/lib/images/uploadImages.ts:367-371`
  ```ts
  function isPngInput(file: ProcessImageInput): boolean {
    if (file.mimeType === 'image/png') return true;
    const lower = file.uri.toLowerCase().split('?')[0];
    return lower.endsWith('.png');
  }
  ```

**Identical logic**, differing only in the parameter name (`input` vs `file`) — same `ProcessImageInput` type, same `image/png` mimeType check, same `.toLowerCase().split('?')[0].endsWith('.png')` URI check. Byte-identical except the identifier. Both are module-local (not exported). Mobile-internal consolidation candidate (no web target, as the brief notes). Call sites: `processImage.ts:118`, `uploadImages.ts:361`.

---

## 6. DTO cleanup — CONFIRM ALREADY DONE

Mobile update-payload construction: `toUpdateWirePayload` in `src/lib/services/productWirePayload.ts:43-57`. Exact field set it sends:

```ts
return { id, name, description, price, currency, filters: product.filters ?? [], imageKeys };
```

→ `{ id, name, description, price, currency, filters, imageKeys }`.

- `oldName` / `oldDescription` / `productState` / `moderationState` are **NOT** in the payload — they are explicitly excluded by allow-list (docblock `productWirePayload.ts:32-42`). The return type is `UpdateProductWireDTO` (a structural `Pick`, per `UpdateProductWireDTO.ts:9-10`), so the forbidden fields cannot ride the wire even though the in-memory `NewProductRequestDTO` carries them (`UpdateProductRequestDTO.ts:6-10` still declares `productState?`, `moderationState?`, `oldName`, `oldDescription` on the loaded object — that is the source object, not the wire payload). `productService.ts:88` calls `updateProduct(toUpdateWirePayload(productData, options.id, allKeys))`. Tests assert the drop: `productWirePayload.test.ts:104-108` enumerate `oldName`/`oldDescription`/`productState`/`moderationState` as fields the payload must NOT contain.

**Verdict:** Already done by A — drop this item.

---

## 7. Review upload scope — RISK CHECK

Mobile review-image upload path: `ProductReviewDialog.submitReview` (`ProductReviewDialog.tsx:65-140`) → `reviewProduct(...)` (`reviewService.ts:70`). The token/upload request inside `reviewProduct` calls `uploadImages(filesToUpload, { scope: 'product', … })` — **`reviewService.ts:98`** (with the comment "Reviews share the products prefix per contract §6.2").

Additional safety: the scope type itself forbids `'review'` — `ImageScope = 'product' | 'profile' | 'chat' | 'report'` (`src/lib/services/imageTokensService.ts:9`). A literal `scope: 'review'` would be a compile error. All scope call sites confirmed in-enum: `reviewService.ts:98` → `product`, `productService.ts:67` → `product`, `authService.ts:80` → `profile`, `viewTokens.ts:100` → `chat`.

**Verdict:** Correct. Review uploads request tokens with `scope="product"` (`reviewService.ts:98`), routed through the shared product-scope orchestrator. No `scope="review"` anywhere. Not a bug.

---

## 8. Upload timing — early vs late

For the product **create** flow, image bytes upload **late** — at the final create step, not at photo-pick.

- The picker step collects images into in-memory state only. `ImagesImport.tsx` contains no upload call (`grep "uploadImages\|uploadProduct\|reviewProduct\|requestUploadTokens\|imageTokens"` → zero hits); it only mutates the `images` array via `setImages`. Step 1 of the wizard is `ImageSelectionProductDialog`, which feeds `handleChange({ imagesData })` (`AddUpdateProductDialog.tsx:128-130`) — in-memory only.
- The actual upload fires when the **final** step's dialog mounts: `UploadedProductDialog` is step 4 (`AddUpdateProductDialog.tsx:159-169`), and its `useEffect` (`UploadedProductDialog.tsx:151-153`) runs `uploadProductInternal()`, which calls `uploadProduct(productData, { mode: 'create', … })` (`UploadedProductDialog.tsx:94`). That is the upload-trigger call site.
- (Edit flow upload-trigger, for completeness: `app/owner/dashboard/products/[productId].tsx:206`.)

**Verdict (informational):** Late — bytes hit R2 only at the final create step (`UploadedProductDialog.tsx:94`, fired on mount via the line-151 effect), matching web. The orphan window is the same shape as web's; picking a photo does not upload it.

---

## Summary of verdicts (items 3, 4, 6 explicit per brief)

| # | Item | Verdict |
|---|------|---------|
| 1 | Progress text in upload dialog | **Still a problem** — spinner only (`UploadedProductDialog.tsx:182-186`); store + `stageLabel` + INPUT keys all present, dialog doesn't subscribe/render. |
| 2 | B15 hardcoded Serbian | Live surface is `ImageSourceSheet.tsx` (3 strings, lines 23/34/42, fix-the-string — confirm it's mounted first). `ProductReviewImageImport.tsx` is dead → delete-the-file (item 4), do not fix its strings. |
| 3 | `productUpdateNameValidator.ts` | **Not found, zero references** — no file, nothing to delete; drop. |
| 4a | `ProductReviewImageImport.tsx` | **Zero callers, safe to delete.** Live path = `ImagesImport` (`ProductReviewDialog.tsx:1,192`). |
| 4b | `app/__smoke__/upload.tsx` | **Zero importers**, self-contained harness, but a reachable expo-router route. Safe to delete per its own header; cleanup decision, not a correctness fix. |
| 5 | Duplicate `isPngInput` | Both exist (`processImage.ts:228`, `uploadImages.ts:367`); **byte-identical logic** (param name aside). Mobile-internal consolidation. |
| 6 | DTO cleanup (`toUpdateWirePayload`) | **Already done by A, drop** — sends `{id,name,description,price,currency,filters,imageKeys}`; forbidden fields absent (`productWirePayload.ts:43-57`). |
| 7 | Review upload scope | **Correct** — `scope="product"` (`reviewService.ts:98`); `'review'` not in the `ImageScope` enum. Not a bug. |
| 8 | Upload timing | **Late** (final create step, `UploadedProductDialog.tsx:94`), matching web. Informational. |

---

## Config-file impact

Nothing. This is a read-only audit; it makes no code change and surfaces no required edit to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`. The image-pipeline Expo-backlog row in `state.md` already reads `in-progress` (smoke pending) and needs no change from this audit. No closure-gate config dependency.
