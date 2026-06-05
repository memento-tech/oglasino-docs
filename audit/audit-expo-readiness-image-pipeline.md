# Audit — Expo release readiness: image pipeline

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only — no code changes, no git operations, no dependency installs, no runtime execution.

---

## Section 1 — Image picker surface

### Library

`expo-image-picker` v17.0.10 (`package.json:46`).

### Picker invocation sites

| File | Line(s) | Camera | Gallery | Context |
|------|---------|--------|---------|---------|
| `src/components/ImagesImport.tsx` | 99 (camera), 109-112 (gallery) | yes | yes (multi-select) | Product listing images |
| `src/components/messages/MessageInput.tsx` | 64-69 | no | yes (multi-select, limit 5) | Chat message attachments |
| `src/components/dialog/components/ProductReviewImageImport.tsx` | 61-64 (camera), 77-82 (gallery) | yes | yes (multi-select, dynamic limit) | Product review images |
| `src/components/dashboard/components/AvatarUpload.tsx` | 46-50 | no | yes (single, `allowsEditing: true`) | Profile avatar |
| `app/__smoke__/upload.tsx` | 124-127 | no | yes (single, quality 1) | Smoke harness (not production) |

### Client-side constraints

- **Quality:** 0.8 at picker level (all production surfaces). Additional two-pass processing in `processImage.ts` at 0.85 primary / 0.75 fallback.
- **Count limits:** Chat messages: 5 (`MAX_IMAGES` at `MessageInput.tsx:21`). Reviews: 5 (`MAX_NUMBER_OF_IMAGES_PER_MESSAGE` at `ProductReviewImageImport.tsx:18`). Product listings: configurable via `maxNumberOfImages` prop. Avatar: 1 (single selection).
- **File type:** `mediaTypes: 'images'` or `ImagePicker.MediaTypeOptions.Images` — images only across all surfaces. HEIC/HEIF detected and auto-converted to JPEG at picker boundary (`preparePickerAssets.ts:92-98`).
- **Dimensions:** No explicit picker-level dimension constraints. `processImage.ts` enforces MAX_DIMENSION = 8000px (input validation) and RESIZE_LONGEST_SIDE = 2400px (resize threshold).
- **File size:** No client-side picker-level size limit. `processImage.ts` enforces 10MB max input (`MAX_INPUT_BYTES`) and 5MB max output (`MAX_OUTPUT_BYTES`).

### Camera + library options

- **ImagesImport** (product listings): Both, via `ImageSourceSheet` component (`ImagesImport.tsx:210-215`).
- **ProductReviewImageImport** (reviews): Both, via `Alert.alert` dialog (hardcoded Serbian text: "Dodavanje slika").
- **MessageInput** (chat): Gallery only.
- **AvatarUpload** (profile): Gallery only, with editing (crop).

### Permission handling

**Permission helper:** `src/lib/permissions/imagePermissions.ts`

- `ensureCameraPermission()` (lines 3-9): checks `getCameraPermissionsAsync()`, requests via `requestCameraPermissionsAsync()` if not granted.
- `ensureGalleryPermission()` (lines 11-17): checks `getMediaLibraryPermissionsAsync()`, requests via `requestMediaLibraryPermissionsAsync()` if not granted.

All picker invocations are guarded by permission checks before launching.

**iOS declarations** (`ios/Oglasino/Info.plist`):
- `NSCameraUsageDescription` (line 56-57): "Allow $(PRODUCT_NAME) to access your camera"
- `NSPhotoLibraryUsageDescription` (line 62-63): "Allow $(PRODUCT_NAME) to access your photos"

**Android declarations** (`android/app/src/main/AndroidManifest.xml`):
- `READ_EXTERNAL_STORAGE` (line 3)
- `WRITE_EXTERNAL_STORAGE` (line 7)
- No explicit `CAMERA` permission in manifest (Expo auto-adds based on pod usage).

**app.config.ts:** No explicit image-picker permission plugins. Expo handles permissions automatically for `expo-image-picker`.

---

## Section 2 — Image processing pipeline

### Transformations between picker and upload

| Step | File | Function | Description |
|------|------|----------|-------------|
| HEIC detection | `src/lib/images/preparePickerAssets.ts:92-98` | `isHeic()` | Detects HEIC/HEIF via MIME type (`image/heic`, `image/heif`) with filename extension fallback |
| HEIC → JPEG conversion | `src/lib/images/preparePickerAssets.ts:59-68` | `convertHeicToJpeg()` | `manipulateAsync(uri, [], { compress: 0.95, format: SaveFormat.JPEG })`. Empty actions array = decode + re-encode only. Quality 0.95 to minimize cumulative loss. |
| Input validation | `src/lib/images/processImage.ts:107` | `processImageForUpload()` | Rejects inputs > 10MB (`MAX_INPUT_BYTES`) or > 8000px on longest side (`MAX_DIMENSION`) |
| Resize | `src/lib/images/processImage.ts:107-116` | `processImageForUpload()` | If longest side > 2400px (`RESIZE_LONGEST_SIDE`), resizes to 2400px maintaining aspect ratio via `manipulateAsync` with `{ resize: { width: 2400 } }` or `{ resize: { height: 2400 } }` |
| JPEG encode (primary) | `src/lib/images/processImage.ts:156-181` | `runManipulate()` | Quality 0.85 (`QUALITY_PRIMARY`). Output format JPEG unless `outputFormat: 'png-passthrough'` and input is PNG. |
| JPEG encode (fallback) | `src/lib/images/processImage.ts:156-181` | `runManipulate()` | If primary output > 5MB (`MAX_OUTPUT_BYTES`), re-encodes at quality 0.75 (`QUALITY_FALLBACK`). If still > 5MB, throws `FILE_TOO_LARGE`. |

**Library used for manipulation:** `expo-image-manipulator` v14.0.8 (`package.json:45`).

### EXIF stripping

No explicit EXIF stripping code on mobile. The re-encoding via `manipulateAsync` with JPEG output naturally strips EXIF. The `uploadInput.ts:1-11` adapter intentionally filters out picker fields (`base64`, `duration`, `exif`) at the boundary — only URI, width, height, fileName, mimeType, and fileSize are passed downstream. Server-side EXIF stripping is documented as handled by Cloudflare Image Resizing.

### Watermarking

- **Env config:** `EXPO_PUBLIC_WATERMARK_ENABLED=false` in both `.env.production:5` and `.env.development:5`.
- **Implementation:** `src/lib/images/variants.ts:1-34`. Watermarking is **server-side** (Cloudflare Image Resizing), not mobile-local. The `buildHeroParams()` function conditionally adds a `draw` parameter to the CDN URL when enabled: logo at `${CDN}/public/brand/logo-watermark.png`, positioned bottom-right, 120px wide, 0.7 opacity.
- **Currently disabled** on both environments.

### Thumbnail generation

Mobile does **not** generate thumbnails locally. Thumbnails are generated by Cloudflare Image Resizing via URL-based variant parameters. The `variants.ts` module constructs URLs with `card` (400×300, cover, quality 85) or `hero` (1600×1200, scale-down, quality 85) parameters.

---

## Section 3 — Upload to R2

### Token endpoint

Mobile calls `POST /secure/images/upload-tokens` (`src/lib/services/imageTokensService.ts:52-73`).

### Wire shape

**Request (mobile sends):**
```typescript
{
  scope: 'product' | 'profile' | 'chat' | 'report',
  count: number,          // number of tokens requested
  contentTypes: string[], // one per file, e.g. ['image/jpeg', 'image/jpeg']
  chatId?: string         // only when scope='chat'
}
```

**Response (mobile expects):**
```typescript
{
  tokens: [{
    token: string,      // JWT with scope/key/content-type claims
    key: string,        // R2 storage key, e.g. 'public/products/uuid.jpg'
    uploadUrl: string,  // full CDN PUT endpoint for R2
    expiresAt: string   // ISO timestamp
  }]
}
```

Token count mismatch (response tokens ≠ request count) throws `UploadError('TOKEN_COUNT_MISMATCH')` at `uploadImages.ts:116-120`.

### PUT to R2

Direct HTTP via `expo-file-system` (`src/lib/images/uploadPrimitive.ts:23-88`). Uses `FileSystem.uploadAsync()` or `FileSystem.createUploadTask()` in `BINARY_CONTENT` mode (raw bytes, not multipart):

```
PUT {uploadUrl}
Headers:
  x-upload-token: {JWT token}
  Content-Type: {image/jpeg or image/png}
Body: raw image bytes
```

Supports progress callback via `onProgress` and abort via `AbortSignal`.

### Upload concurrency

**Parallel with fail-fast semantics** (`uploadImages.ts:134-167`). Uses `Promise.allSettled` to fire all per-file pipelines concurrently. On any failure, successfully-uploaded keys are cleaned up via `cleanupOrphanImages` in the background.

### What mobile does with returned R2 keys

After successful PUT, `uploadOneFile` returns `tokenEntry.key` (the pre-assigned R2 key, `uploadImages.ts:224`). Keys are collected into an `imageKeys` array and POSTed to `POST /secure/product/addUpdate` in the product creation/update payload (`productService.ts:55-67`).

### Progress indicator

Yes, wired via `useUploadProgressStore` Zustand store (`src/lib/stores/uploadProgress.ts`). Per-file stage tracking with stages: `idle` → `validating` → `resizing` → `encoding` → `uploading` → `complete` (or `converting-heic` for HEIC inputs, or `cancelled`/`error` on failure). Consumed by `ImagesImport`, `AvatarUpload`, `ProductReviewImageImport`, `MessageInput`.

---

## Section 4 — Integration with product create and edit

### Create flow — upload invocation point

The create flow is a 4-step wizard (`AddUpdateProductDialog.tsx`):
- **Step 0:** `ImageSelectionProductDialog` — user picks images. Files are stored in `imagesData[].file` (in-memory `ImagePickerAsset` references). **No upload occurs.**
- **Step 1:** `BasicInfoProductDialog` — name, description.
- **Step 2:** `MetaDataProductDialog` — category, filters, price.
- **Step 3:** `UploadedProductDialog` — upload is triggered automatically via `useEffect` on `productData` change (`UploadedProductDialog.tsx:92-94`). Calls `uploadProduct(productData)` which: (a) uploads all files to R2, (b) collects R2 keys, (c) POSTs to `/secure/product/addUpdate`.

### Model C compliance

**Confirmed.** Images are held in memory (as `ImagePickerAsset` file references in `imagesData`) through steps 0-2. Upload to R2 occurs only at the final step (step 3). This matches the documented "Model C: hold images in memory through wizard; upload only at final step" decision from `decisions.md` 2026-05-13.

Progressive status is provided via the `useUploadProgressStore` Zustand store, though the UI shows an `ActivityIndicator` spinner rather than the "Uploading images..." → "Creating product..." → done message flow described in the Model C decision.

### Edit flow — existing vs new images

The edit screen (`app/owner/dashboard/products/[productId].tsx`) initializes `imagesData` from existing `imageKeys.map((key) => ({ key }))`. When the user adds new images, they appear as `ImageData { file: ImagePickerAsset }`.

`productService.ts:33-42` discriminates:
- `data.file` present → `{ kind: 'file', uploadIndex }` → queued for upload
- `data.key` present (no file) → `{ kind: 'key', key }` → reused as-is

Both new keys (from upload) and existing keys are merged into `allKeys` and sent in `imageKeys` on the wire (`productService.ts:55-57`).

### Orphan cleanup on persist failure

When `addUpdateProductData` returns `null` (persist failure) and `newKeys.length > 0`, orphan cleanup runs in the background:

```typescript
if (response === null && newKeys.length > 0) {
  void cleanupOrphanImages(newKeys);
}
```
(`productService.ts:71-74`)

`cleanupOrphanImages` (`uploadImages.ts:333-336`) issues `DELETE /secure/images/{key}` for each key via `deleteImageKey` (`imageTokensService.ts:82-88`). Failures are logged and swallowed — backend's scheduled sweeper picks up anything missed.

The same pattern exists in `reviewService.ts:122-127`.

### Upload-phase failure orphan cleanup

During the upload phase itself, if any file fails, `uploadImages.ts:150` cleans up already-succeeded keys before rethrowing: `if (succeededKeys.length > 0) void cleanupOrphanImages(succeededKeys)`.

---

## Section 5 — Image display

### Display components

| Component | File | Variant | Context |
|-----------|------|---------|---------|
| `ProductTopImage` | `src/components/product/ProductTopImage.tsx` | `card` (400×300) | Product list thumbnails |
| `ImagesCarousel` | `src/components/ImagesCarousel.tsx` | `hero` (1600×1200) | Product detail gallery |
| `ZoomableImage` | `src/components/ZoomableImage.tsx` | (receives URL) | Full-screen pinch-zoom viewer |
| `OglasinoAvatar` | `src/components/user/OglasinoAvatar.tsx` | `card` (400×300) | User avatars |
| `MessageImages` | `src/components/messages/MessageImages.tsx` | private URL with view token | Chat message images |

### CDN URL pattern

- **Base URL:** `EXPO_PUBLIC_CDN_URL` (`.env.production`: `https://cdn.oglasino.com`, `.env.development`: `https://cdn-stage.oglasino.com`)
- **Public images:** `{CDN}/cdn-cgi/image/{variant-params}/{key}` for transformed variants, or `{CDN}/{key}` for original (`variants.ts:42-47`)
- **Private images (chat):** `{CDN}/{key}?token={encodeURIComponent(viewToken)}` (`variants.ts:49-51`)

### Image caching

Default React Native `Image` component with OS-level HTTP cache. **No explicit caching layer.** `expo-image` v3.0.11 is installed in `package.json` but is **not imported anywhere in `src/`** — it is effectively unused.

View tokens for private chat images are cached per-chat in a Zustand store (`src/lib/stores/viewTokens.ts`) with a 60-second pre-expiry buffer and in-flight request deduplication.

### Size selection per surface

Yes — `card` variant (400×300, cover, quality 85) for thumbnails and avatars; `hero` variant (1600×1200, scale-down, quality 85) for detail views and carousels.

### Broken image handling

- **ImagesCarousel:** Tracks `failedImages` by URI. `onError` marks image failed, shows Oglasino icon + localized message (`product.image.failed.to.load`).
- **ZoomableImage:** Single `failed` state; on error shows icon + message. Resets on URI change.
- **ProductTopImage:** `failed` state hides the image (relies on background color placeholder).
- **MessageImages:** 401 triggers token cache invalidation + single retry. Generic error clears images silently.
- **No retry mechanism** for non-token-related broken images (4xx/5xx from CDN surface immediately).

---

## Section 6 — Trust boundary check

### Request 1: `POST /secure/images/upload-tokens`

| Field | Source | Trust concern |
|-------|--------|---------------|
| `scope` | Hardcoded per call site ('product', 'profile', 'chat', 'report') | **Safe.** Not user input; computed from app flow. |
| `count` | Derived from `files.length` | **Safe.** Computed from the number of picked images. |
| `contentTypes[]` | Derived from file MIME type / extension analysis | **Safe.** Computed, not user-supplied text. |
| `chatId` | From chat store state (only when `scope='chat'`) | **Check needed.** Client-supplied. Whether server verifies the authenticated user is a participant in this chat is a server-side question mobile cannot answer, but mobile does send it. |

**No owner, product association, or moderation state fields** are sent on the token request. The token endpoint does not accept a `productId` — product association is established only when the product is persisted via `addUpdateProductData`, not at token issuance.

### Request 2: `PUT {uploadUrl}` (direct to R2 via Worker)

| Field | Source | Trust concern |
|-------|--------|---------------|
| `x-upload-token` header | JWT from token response | **Safe.** Server-issued, not client-constructed. |
| `Content-Type` header | Derived from file analysis, matches `contentTypes[]` claim in token request | **Safe.** Computed. |
| Request body | Raw image bytes | **Safe.** Binary payload. |

**No owner, product, or moderation fields** on the PUT request. The JWT carries scope and key claims; the Worker validates the JWT.

### Request 3: `POST /secure/product/addUpdate`

| Field | Source | Trust concern |
|-------|--------|---------------|
| `imageKeys[]` | R2 keys returned by token endpoint + existing keys from prior fetch | **Investigate.** Keys for new uploads are server-assigned (safe). Keys for existing images come from a prior `getDashboardProductDetails` response — client passes them back unmodified, but the server must verify the user owns the product those keys belong to. |
| `name`, `description`, `price`, etc. | User input | Standard user input; trust boundary is server-side validation. |
| `oldName`, `oldDescription` | From `UpdateProductRequestDTO` | **TRUST BOUNDARY FINDING — see below.** |
| `productState`, `moderationState` | From `UpdateProductRequestDTO` | **TRUST BOUNDARY FINDING — see below.** |

### Request 4: `DELETE /secure/images/{key}`

| Field | Source | Trust concern |
|-------|--------|---------------|
| `{key}` (path parameter) | R2 key from a prior upload session | **Check needed.** Client tells the server which key to delete. Server must verify the authenticated user owns that key. Mobile cannot verify this. |

### Request 5: `POST /secure/images/view-tokens`

| Field | Source | Trust concern |
|-------|--------|---------------|
| `scope` | Hardcoded `'chat'` | **Safe.** |
| `chatId` | From chat context | Same concern as Request 1 `chatId`. |

### Trust boundary findings

**Finding 1: `UpdateProductRequestDTO` carries `oldName` and `oldDescription` as client-supplied fields.**

The `UpdateProductRequestDTO` (`src/lib/types/product/UpdateProductRequestDTO.ts:9-10`) includes `oldName: string` and `oldDescription: string`. These are client-supplied "previous values" for change detection. Per `decisions.md` 2026-05-13 (product-validation session 2), this is a known trust-boundary violation that was **already fixed on the backend** — `oldName`/`oldDescription` were removed from the backend's `UpdateProductRequestDTO` and comparison moved server-side. However, the mobile DTO still declares these fields.

The `productUpdateNameValidator.ts` file still exists and exports `isMassiveChange(oldName, newName)` (line 1), but has zero callers outside its own module — the import is never used in any component or service. This is dead code.

**Impact:** The backend ignores these fields (they were removed from the backend DTO). No trust boundary is violated in practice because the server doesn't consume them. But the mobile DTO is stale and the validator is dead code.

**Finding 2: `UpdateProductRequestDTO` carries `productState` and `moderationState` as client-supplied fields.**

The DTO (`src/lib/types/product/UpdateProductRequestDTO.ts:6-7`) includes `productState?: ProductState` and `moderationState?: ModerationState`. If the client sends these and the backend consumes them for state transitions, that's a Part 11 violation — moderation state must be server-derived.

**Impact:** Cannot determine server-side behavior from mobile code alone. Flagging for cross-repo verification.

---

## Section 7 — Orphan risk and cleanup

### User closes app mid-upload

Partial uploads are orphaned. The R2 Worker accepts the PUT and stores the bytes; the mobile app has no `AppState` listener or background task to clean up in-flight uploads on app termination. Backend's scheduled sweeper is the safety net.

### Upload succeeds but product POST fails

Documented cleanup path: `productService.ts:71-74` calls `cleanupOrphanImages(newKeys)` in the background (best-effort `Promise.allSettled` → `DELETE /secure/images/{key}` per key). Same pattern in `reviewService.ts:122-127`. Failures are logged and swallowed; backend sweeper catches the rest.

### Wizard abandoned at step 2 (before upload)

**No orphan risk.** Per Model C, no upload occurs during steps 0-2. Images are held in memory as `ImagePickerAsset` references. When the wizard is dismissed, the component tree unmounts, the in-memory references are garbage-collected, and no R2 keys exist. Verified: no `uploadImages` or `uploadProduct` call exists in `ImageSelectionProductDialog`, `BasicInfoProductDialog`, or `MetaDataProductDialog`.

### Periodic orphan sweep on mobile

**No.** Mobile does not run any periodic orphan cleanup. The backend's scheduled sweeper is the sole periodic mechanism. This is expected — a mobile app is not the right place for a sweep job.

---

## Section 8 — Dead code

### Unused image-related code

| Item | File | Status |
|------|------|--------|
| `productUpdateNameValidator.ts` | `src/lib/validators/productUpdateNameValidator.ts` | **Dead code.** `isMassiveChange` has zero callers. The function was part of the pre-validation `oldName` flow removed from backend in product-validation session 2. |
| `UpdateProductRequestDTO.oldName` / `oldDescription` | `src/lib/types/product/UpdateProductRequestDTO.ts:9-10` | **Stale fields.** Backend no longer accepts these. |
| `UpdateProductRequestDTO.productState` / `moderationState` | `src/lib/types/product/UpdateProductRequestDTO.ts:6-7` | **Likely stale.** Needs cross-repo verification of whether backend still accepts them on `addUpdate`. |
| `expo-image` | `package.json` (v3.0.11) | **Installed but unused.** Zero imports in `src/`. Only RN's built-in `Image` is used. |
| `expo-secure-store` | `package.json` (v15.0.8) | **Installed but unused.** Zero imports in `src/`. (Also noted by general audit.) |
| `//TODO TRANSLATIONS ADD` | `src/components/ImageSourceSheet.tsx:13` | Hardcoded Serbian strings ("Izaberite izvor slike", "Kamera", "Galerija") awaiting i18n. |
| `//TODO TODO` | `src/components/dialog/components/ProductReviewImageImport.tsx:20` | Unclear double-TODO marker. Component is functional. |

### Commented-out blocks

No commented-out blocks found in image upload orchestration code (`uploadImages.ts`, `processImage.ts`, `uploadPrimitive.ts`, `imageTokensService.ts`, `productService.ts`).

### Smoke harness

`app/__smoke__/upload.tsx` exists (371 lines, last modified 2026-05-09). Per brief instructions: noted, not audited as production code. Brief documents it as "deletable as a unit."

---

## Section 9 — Cross-cutting observations

### `expo-secure-store` and image upload tokens

Confirmed: `expo-secure-store` is installed (`package.json`) but has zero imports across `src/`. Upload tokens are transient — fetched from `/secure/images/upload-tokens`, held in local variables during the upload pipeline, and discarded after the PUT completes. View tokens for chat images are cached in a Zustand in-memory store (`src/lib/stores/viewTokens.ts`) with TTL-aware expiry. Neither token type is stored in secure storage. This is correct for the current architecture — upload tokens are single-use JWTs consumed immediately; view tokens have a server-defined TTL and are re-fetched on expiry.

### Token-refresh on 401 mid-flight

The global axios interceptor (`src/lib/config/api.ts:33-62`) has **no** 401 → token-refresh logic. It handles `ERR_NETWORK`, `ECONNABORTED`, and 404 specifically; all other errors (including 401) pass through as-is.

For image upload tokens, the upload pipeline has its own 401 handling at the PUT level (`uploadImages.ts:282-299`): if the Worker returns 401 with error code `TOKEN_EXPIRED`, the pipeline refetches a single replacement token via `refetchSingleToken()` and retries the PUT once. This is scoped to the Worker's JWT validation, not to Firebase auth expiry.

For the token *issuance* request itself (`POST /secure/images/upload-tokens`), if the user's Firebase ID token has expired and `auth.currentUser.getIdToken()` returns a stale token, the backend returns 401. The axios request interceptor (`api.ts:21-29`) calls `auth.currentUser.getIdToken()` on every request, which Firebase SDK auto-refreshes when needed. So the issuance path is implicitly covered by Firebase's token refresh. However, if the Firebase token refresh itself fails (e.g., user signed out, network down), the 401 from the backend propagates as an `ImageTokenError` with no automatic retry — the user sees an upload failure toast.

---

## Section 10 — For Mastermind

### Trust-boundary findings

1. **`UpdateProductRequestDTO` still carries `oldName` and `oldDescription`** (`src/lib/types/product/UpdateProductRequestDTO.ts:9-10`). The backend removed these fields from its DTO in product-validation session 2 (2026-05-13). The mobile DTO is stale. The companion `productUpdateNameValidator.ts` (`isMassiveChange`) is dead code with zero callers. Neither causes a runtime trust violation (backend ignores the fields), but the mobile code misleads any reader into thinking client-supplied previous values are part of the wire contract.

2. **`UpdateProductRequestDTO` carries `productState` and `moderationState`** (`src/lib/types/product/UpdateProductRequestDTO.ts:6-7`). If the backend's `addUpdate` endpoint accepts and acts on client-supplied `productState` or `moderationState`, that's a Part 11 violation — moderation and product state must be server-derived. Cannot verify server-side from this repo. Needs cross-repo check.

3. **`chatId` on token requests is client-supplied.** Both `POST /secure/images/upload-tokens` (when `scope='chat'`) and `POST /secure/images/view-tokens` accept a `chatId` from the client. The server must verify the authenticated user is a participant in the named chat before issuing tokens scoped to that chat's private image path. Mobile cannot verify this; flagging for awareness.

### Orphan risks

4. **App termination mid-upload orphans R2 keys with no client-side recovery.** No `AppState` listener or background task attempts cleanup on app backgrounding or termination during an active upload. Backend sweeper is the safety net. Acceptable for v1; quantify orphan accumulation post-launch.

5. **Persist-phase cleanup is best-effort fire-and-forget.** `cleanupOrphanImages` logs and swallows failures. If the `DELETE /secure/images/{key}` calls fail (network, auth, server error), orphans persist until the backend sweeper runs. Acceptable architecture given the cost model.

### Divergence from Model C

6. **No divergence.** Images are held in memory through wizard steps 0-2, uploaded only at step 3 (the final step). Model C compliance confirmed at `AddUpdateProductDialog.tsx` step order and `UploadedProductDialog.tsx:92-94` upload trigger.

7. **Minor divergence in UX from Model C description.** The Model C decision describes "Progressive status messages during step 4 ('Uploading images...' → 'Creating product...' → done)." The current implementation shows an `ActivityIndicator` spinner during upload and a success/failure view after, but does not render the progressive "Uploading images..." → "Creating product..." transition text. The per-file progress store (`uploadProgress.ts`) tracks stages but the `UploadedProductDialog` does not surface them to the user at the aggregate level.

### Cross-feature observations (one line each)

8. **Hardcoded Serbian strings in `ImageSourceSheet.tsx` and `ProductReviewImageImport.tsx`.** Both components use Serbian text for image source selection UI. Part 6 (translations) violation. Out of scope for this audit — relevant to general audit or product-validation adoption.

9. **`expo-image` v3.0.11 installed but unused.** Zero imports in `src/`. Could be removed or adopted as the caching Image component (it supports memory + disk caching natively). Relevant to general audit.

10. **`console.error('Copy failed', e)` in `UploadedProductDialog.tsx:103`.** Part 4 violation (ad-hoc debug logging). Out of scope for this audit.

11. **`__DEV__` guarded `console.error` in `UploadedProductDialog.tsx:83`.** Acceptable dev-mode-only logging. Not a Part 4 violation.

### Questions whose answers would change scope

- Does the backend's `addUpdate` endpoint silently accept and discard `productState` / `moderationState` from the request body, or does it act on them? If acts: Part 11 violation requiring a backend fix.
- Is there a backend orphan sweeper running on a schedule today, or is it planned? The client code references it in comments but its existence is a backend-repo question.
- Should mobile adopt `expo-image` (which supports memory + disk caching and progressive loading) instead of the default RN `Image`? Would improve perceived performance on product lists.

---

## Section 11 — Cleanup performed

`none needed` — read-only audit.

---

## Section 12 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 13 — Conventions check

- Part 4 (cleanliness): N/A this session (read-only). Adjacent observations of Part 4 violations flagged in Section 10 items 8, 10.
- Part 4a (simplicity): N/A this session (read-only).
- Part 4b (adjacent observations): Flagged — items 8-11 in Section 10.
- Part 6 (translations): Spot-checked image-related strings. Hardcoded Serbian in `ImageSourceSheet.tsx` and `ProductReviewImageImport.tsx` flagged (Section 10 item 8).
- Part 8 (architectural defaults): Touched — "direct-to-R2 for image uploads" confirmed. Mobile uses the documented token-based direct PUT pattern via `expo-file-system`.
- Part 11 (trust boundaries): Touched — explicit check in Section 6. Three findings flagged in Section 10 items 1-3.
