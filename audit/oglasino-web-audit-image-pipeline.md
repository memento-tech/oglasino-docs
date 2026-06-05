# Audit — oglasino-web image pipeline (client side)

**Repo:** oglasino-web · **Branch:** `stage` · **Type:** read-only, code-as-ground-truth
**Date:** 2026-05-30 · **Feature spec:** `../oglasino-docs/features/image-pipeline.md` (`web-stable`)

This is the code-level reference the mobile (`oglasino-expo`) implementation will be built against. Every claim below is from the actual web code; where the code disagrees with the spec, the code is reported and the gap is flagged **DIVERGENCE**.

The whole client pipeline lives in `src/lib/images/`:

| File | Role |
|---|---|
| `processImage.ts` | Browser-side 8-step processing (validate, HEIC, resize, encode) |
| `imageDecoder.ts` | DOM decode helpers — dimensions, PNG-alpha, HEIC→JPEG |
| `preparePickerFiles.ts` | Picker-time HEIC pre-conversion (so thumbnails render) |
| `uploadImages.ts` | Token request, PUT to Worker, retry policy, orphan DELETE |
| `variants.ts` | `publicImageUrl` / `privateImageUrl` URL helpers + env reads |
| `errorMapping.ts` | error-code → i18n key, toast titles, stage labels |
| `src/lib/stores/viewTokens.ts` | Per-chat view-token Zustand store |
| `src/lib/stores/uploadProgress.ts` | Upload-progress UI state |

Auth/HTTP plumbing: `src/lib/config/api.ts` (`BACKEND_API` axios instance + Firebase token interceptor).

---

## Code-vs-spec divergence summary

| # | Area | Spec says | Code says | Impact for mobile |
|---|---|---|---|---|
| D1 | View-token store (6) | Deferred; `MessageImages` "still uses per-component `useState`" | `viewTokens.ts` is a **fully-built Zustand store** (60s pre-expiry buffer, in-flight dedupe, `invalidate`, `clear` at logout) and `MessageImages` consumes it | Spec "Deferred items" section is **stale**. Mobile replicates the store, not a useState stub. |
| D2 | Retry policy (3) | Deferred; "currently single-retry on TOKEN_EXPIRED only" | 5xx backoff `[1s,4s]` (2 retries) on token-request **and** PUT; 429 surfaces immediately w/ `retryAfterSec`; 401-on-private-GET invalidate+retry-once in `MessageImages` — all **implemented** (Phase 8) | Spec "Deferred items" section is **stale**. Mobile replicates the real policy. |
| D3 | Upload scope enum (1) | `product \| profile \| chat \| report` (report reserved v2) | `UploadScope = product \| profile \| chat \| report \| review` — **`review` is live** (used by `reviewService`) | Mobile must include `review` scope. |
| D4 | Pipeline order (2) | Dimensions (step 3) before HEIC (step 4) | **HEIC conversion runs before the dimension check** — browsers can't decode HEIC bytes to measure them | Intentional, documented in-code. Mobile ordering should match. |
| D5 | Translation keys (8) | 10 active + 7 "phase-8 reserved", inline `TODO(phase 7)` fallbacks | **13 live `ImageErrorTranslationKey`s** wired through `safeT` + `englishFallback`; **no `TODO(phase 7)` comments** remain; adds 4 DELETE-path keys not in spec table | Spec "Translation keys / Pending registration" framing is **stale**. |
| D6 | Variants (5) | `card`, `hero` only | Adds an **`original`** variant (`${CDN_BASE}/${key}`, no resizing) used for flag icons | Minor; mobile likely won't need `original`. |
| D7 | Internal codes (3/8) | n/a | Client mints extra codes: `TOKEN_COUNT_MISMATCH`, `TOKEN_REISSUE_FAILED`, `NETWORK_ERROR`, `PROCESSING_FAILED` — all map through `errorMapping` | Mobile error mapping should carry the same internal codes. |

The processing **thresholds** (10 MB raw / 8000² dims / 2400 resize / 5 MB cap / quality 0.85→0.75) and the **variant params** (card 400×300 cover q85; hero 1600×1200 scale-down q85 watermark-gated) match the spec exactly — no drift.

---

## 1. Upload-token request

**Call site:** `requestUploadTokens()` — `src/lib/images/uploadImages.ts:174`, invoked from `uploadImages()` (`:146`). `POST /secure/images/upload-tokens` via `BACKEND_API` (`:200`).

**Request body** (`:181`):

```
{ scope, count: contentTypes.length, contentTypes: string[], chatId? }
```

Field derivation:
- `scope` — caller-supplied arg (`uploadImages(files, scope, chatId)`); `UploadScope = 'product'|'profile'|'chat'|'report'|'review'` (`:52`). **DIVERGENCE D3** — `review` is live, spec omits it.
- `count` — `contentTypes.length` (`:183`).
- `contentTypes` — `processed.map(p => p.blob.type)` (`:145`); these are the **processed** blob types (post HEIC→JPEG / transcode), not the original file types.
- `chatId` — only added when truthy (`:186`); the client never sends `chatId: null`.

**Firebase auth attachment** — not done in `uploadImages`; done by the `BACKEND_API` request interceptor in `src/lib/config/api.ts:94`:
- `auth.currentUser.getIdTokenResult()` (`:119`), cached with a 1-minute early-refresh margin (`:121`), set as `Authorization: Bearer <token>` (`:137`).
- Respects a caller-supplied `Authorization` header (`:111`, used by the user-deletion forced-refresh path).
- Cached token cleared on `onIdTokenChanged(null)` (`:87`); `auth/user-disabled` flips the ban dialog (`:129`).
- The same interceptor also sets `X-Base-Site` / `X-Lang` from `getTenantLocale` (`:97`), and the instance is created with `withCredentials: true` (`:18`).

**Response shape** (`UploadTokensResponse`, `:96`): `{ tokens: { token, key, uploadUrl, expiresAt }[] }`.

**Token→file mapping:** response consumed at `:146`; count guard `tokens.length !== processed.length` → `TOKEN_COUNT_MISMATCH` (`:147`). The `tokens[i]` entry pairs positionally with `processed[i]` for the PUT (`:154`). `uploadImages` returns `tokens.map(t => t.key)` (`:171`) — full prefixed keys, in input order.

---

## 2. Browser-side processing pipeline

**Files:** `src/lib/images/processImage.ts` (`processImageForUpload`, `:94`), `imageDecoder.ts` (decode helpers), `preparePickerFiles.ts` (`preparePickerFiles`, picker-time HEIC pre-convert).

**Step → code map** (`processImage.ts`):

1. **Validate type** (`:106`) — allowlist `Set` of `image/jpeg|png|webp|heic|heif` (`:79`); else `INVALID_TYPE`.
2. **Validate raw size** (`:112`) — `> MAX_RAW_BYTES` → `FILE_TOO_LARGE`.
3. **HEIC/HEIF → JPEG** (`:122`) — `decoder.convertHeicToJpeg(file, quality)`. **DIVERGENCE D4**: runs *before* the dimension check (spec orders dimensions first). In-code comment: browsers can't natively decode HEIC bytes.
4. **Validate dimensions** (`:133`) — `decoder.readImageDimensions` (`createImageBitmap`, else `<img>` fallback); `> MAX_RAW_DIMENSION` on either axis → `DIMENSIONS_TOO_LARGE`.
5. **Resize decision** (`:150`) — `needsResize = max(w,h) > maxDimension`.
6. **Format normalization** (`pickTargetFormat`, `:205`) — PNG: `detectPngTransparency` → alpha keep `png`, else `jpeg`; WebP→`webp`; JPEG & HEIC-converted→`jpeg`. Alpha-detect failure defensively keeps PNG (`:212`).
7. **Encode + quality fallback** (`:160`) — `runCompression` at `DEFAULT_QUALITY`; if `> maxBytes`, re-encode once at `FALLBACK_QUALITY`.
8. **Return `ProcessedImage`** (`:189`) — `{ blob, originalSize, processedSize, originalDimensions, processedDimensions, format }`.

**Thresholds as coded** (`:87`–`:92`) — all match spec:

| Constant | Value |
|---|---|
| `MAX_RAW_BYTES` | `10 * 1024 * 1024` (10 MB) |
| `MAX_RAW_DIMENSION` | `8000` |
| `DEFAULT_MAX_DIMENSION` | `2400` (resize trigger + target) |
| `DEFAULT_MAX_BYTES` | `5 * 1024 * 1024` (5 MB) |
| `DEFAULT_QUALITY` | `0.85` |
| `FALLBACK_QUALITY` | `0.75` |

**Libraries:**
- `browser-image-compression` — dynamic `import()` inside `runCompression` (`:233`); options: `maxWidthOrHeight`, `useWebWorker: true`, `fileType`, `initialQuality`, `signal`, and **`maxSizeMB: 100`** — a deliberate soft-cap so the library's own iterative quality reduction never fires; the two-pass `0.85→0.75` fallback is owned in `processImage` (`:249` comment).
- `heic2any` — dynamic `import()` in `imageDecoder.convertHeicToJpeg` (`:71`); `heic2any({ blob, toType: 'image/jpeg', quality })`, result re-wrapped as a `.jpg` `File`. Also pre-invoked in `preparePickerFiles` at picker time (quality `0.85`) so HEIC thumbnails render in `<img>`.

**Final Content-Type:** `ProcessedImage.blob.type` (set by `browser-image-compression` from `fileType`) is the source of truth — fed both into the token request `contentTypes` (`uploadImages.ts:145`) and the PUT `Content-Type` header (`uploadImages.ts:346`). Type and wire stay consistent so the Worker's JWT content-type check passes.

---

## 3. Direct PUT to the Worker

**Call site:** `tryPut()` — `src/lib/images/uploadImages.ts:335`, native `fetch(token.uploadUrl, { method: 'PUT', ... })` (`:342`).

**Raw bytes, not multipart** — confirmed: `body: processed.blob` (`:348`) is a native `Blob`; headers are only `x-upload-token` and `Content-Type: processed.blob.type` (`:344`). No `FormData`, no `multipart/form-data`.

**Error handling on the PUT:**
- Network/abort failure → `{code:'CANCELLED'}` if aborted, else `{code:'NETWORK_ERROR'}` (`:351`).
- Non-`ok` response → parse body JSON, read `error.code` (`:359`); fall back to `'UNKNOWN'` if body isn't JSON.
- **Retry shape** (`tryPutWithRetry`, `:306`): retry only on 5xx with backoff `[1000, 4000]` ms (interruptible via `AbortSignal`); every non-5xx (`TOKEN_EXPIRED`, `CONTENT_TYPE_MISMATCH`, `FILE_TOO_LARGE`, 429, …) surfaces immediately.
- **`TOKEN_EXPIRED`** gets a dedicated single retry upstream (`uploadOneWithTokenExpiredRetry`, `:255`): re-request one fresh token for the same content-type, retry the PUT once; failure → `TOKEN_REISSUE_FAILED`.
- Batch semantics: `Promise.allSettled` over all PUTs, first rejection re-thrown as `UploadError` (`:167`) — fail-fast, earlier successes become R2 orphans (cleaned via §7).
- **DIVERGENCE D2**: spec lists this 5xx/429/401 policy as "deferred"; it is implemented.

**Mapping to UI/i18n:** failures throw `UploadError(failedFileIndex, failedFileName, errorCode, httpStatus, details?)` (`:69`). The UI layer turns it into a toast via `errorMapping.buildUploadErrorTitle(err, tErrors)` (see §8). `details.retryAfterSec` carries through for 429.

---

## 4. Key → entity association

In every path the **full prefixed key** from the token response flows verbatim into the entity field — no stripping, no re-prefixing, no bare UUIDs.

- **Product create/update** — `productService.ts`: `extractAndUploadImages()` (`:96`) calls `uploadImages(files, 'product')`, returned keys land in the request DTO's `imageKeys: string[]`, POSTed to `/secure/products/create` / `/secure/products/update`. Splice preserves order: `allKeys[f.originalIndex] = uploadedKeys[i]` (`:138`).
- **Profile / avatar** — OAuth-signup avatar in `authService.ts` (`uploadImages([photoFile],'profile')`, key → Firestore `users/{uid}.profileImageKey`); profile edit on `app/[locale]/owner/user/page.tsx` sends `profileImageKey` via the user-update DTO.
- **Chat message** — `MessageInput.tsx:130` calls `uploadImages(images, 'chat', chatId)`; keys go into an `ImagesMessageBlock.imageKeys` (`:158`), written to Firestore via `useChatStore.ts` (`chats/{chatId}/messages/{id}`).

**Evidence of full prefixed keys:** `uploadImages` returns the backend `key` field unchanged (`uploadImages.ts:171`); the DELETE path re-encodes the same key path-segment-wise (`encodeKeyPath`, §7), which only makes sense for a prefixed `public/products/…` / `private/chats/…` key. No call site reconstructs a key from a UUID.

---

## 5. Public display (products, profiles)

**Helper** — `publicImageUrl(key, variant = 'card')` (`src/lib/images/variants.ts:41`):
- empty key → `''`;
- `variant === 'original'` → `${CDN_BASE}/${key}` (**DIVERGENCE D6**, not in spec table);
- else → `${CDN_BASE}/cdn-cgi/image/${params}/${key}`.

**Variant params (as coded)** — match spec:
- `card` (`:37`): `width=400,height=300,fit=cover,format=auto,quality=85` — no watermark.
- `hero` (`buildHeroParams`, `:18`): `width=1600,height=1200,fit=scale-down,format=auto,quality=85`; watermark `draw` appended **only when `WATERMARK_ENABLED`**.

**Watermark flag** — `WATERMARK_ENABLED = process.env.NEXT_PUBLIC_WATERMARK_ENABLED === 'true'` read once at module load (`:14`). When true, `buildHeroParams` appends an encoded `draw` JSON pointing at `${CDN_BASE}/public/brand/logo-watermark.png` (bottom-right, width 120, opacity 0.7). Gating is entirely inside the URL builder — no component reads the flag. (Note: spec prose reads "hero … watermark"; in code the watermark is correctly **flag-gated**, default off.)

**Consumers of `publicImageUrl`:**
- Product detail carousel — `ProductImageCarusel.tsx` (hero + card).
- Product top image / cards — `ProductTopImage.tsx`, product card render path.
- Review images — `ProductReview.tsx` (card).
- Profile avatar — `OglasinoAvatar.tsx` (server component, card).
- Fullscreen viewer — `FullscreenViewer.tsx` (hero + card).
- Picker preview — `ImagesImport.tsx` (card).
- Admin — `AdminReviewOverviewDialog.tsx`.
- `original` variant for flag icons — `UserBasicDataSelectorDialog.tsx`, `PortalConfigDialog.tsx`.
- SEO/OG metadata generators under `src/metadata/` (`generateProductPageMetadata`, `generateUserPageMetadata`, `generateCatalogPageMetadata`, `generateHomePageMetadata`, plus the structured-data generators).

---

## 6. Private display (chat attachments)

**View-token call** — `fetchTokenFromBackend()` in `src/lib/stores/viewTokens.ts:80`: `POST /secure/images/view-tokens` body `{ scope: 'chat', chatId }`; response `{ token, expiresAt, scope, chatId }`. `expiresAt` parsed to ms epoch; non-finite → throws (not cached).

**Helper** — `privateImageUrl(key, viewToken)` (`variants.ts:47`) → `${CDN_BASE}/${key}?token=${encodeURIComponent(viewToken)}`. No `/cdn-cgi/image/` (no resizing for private images, per spec).

**View-token state — DIVERGENCE D1 (headline).** The spec's "Deferred items" says the Zustand store is planned and `MessageImages` still uses per-component `useState`. **Reality:** `useViewTokenStore` (`viewTokens.ts:34`) is fully implemented:
- `getToken(chatId)` returns cached token when `expiresAt > now + 60_000ms` (`PREEXPIRY_BUFFER_MS`, `:32`); else fetches.
- In-flight dedupe via `inFlight` promise map; failures are not cached (`:57`).
- `invalidate(chatId)` drops the entry; `clear()` wipes all (called at logout — `useAuthStore.ts:261`).

`MessageImages.tsx` consumes it (`:32` `useViewTokenStore.getState().getToken(chatId)`); its only `useState` holds the **resolved URL strings** and a `retryAttempt` counter — not the token. **401/expiry handling:** `<img onError>` can't see HTTP status, so `handleImageError` (`:58`) treats any error as possibly-401: `invalidate(chatId)` + bump `retryAttempt` (effect re-runs, fresh token). Capped at one retry (`:59`); after that the broken-image fallback renders with no toast.

**Renderers of private chat images:** `MessageImages.tsx` (thumbnail + multi-image affordance) and `ViewMessageImagesDialog.tsx` (fullscreen viewer; receives already-resolved `?token=` URLs, fetches no token itself).

---

## 7. DELETE fast-path

**Call site:** `cleanupOrphanImages(keys)` — `src/lib/images/uploadImages.ts:401`. Fires `DELETE /secure/images/${encodeKeyPath(key)}` per key (`:404`), `Promise.allSettled`, failures logged via `console.warn` and swallowed (never throws; callers also wrap in `.catch(() => {})`).

**Key passing:** `encodeKeyPath` (`:424`) splits on `/`, `encodeURIComponent`s each segment, rejoins — **preserves slashes** so the backend `**` route matches the full multi-segment prefixed key; a plain `encodeURIComponent` would 404.

**When it fires** — not "wizard abandonment"; it fires when an **entity save fails after the uploads already succeeded** (orphan cleanup):
- Chat send failure — `useChatStore.ts:569`.
- Product create — `productService.ts:177/183/189`; product update — `:234/240/245`.
- Review create/update — `reviewService.ts:86/91`.
- Avatar update — `app/[locale]/owner/user/page.tsx:241`.

The backend daily/weekly sweeper is the backstop for anything the client misses.

---

## 8. Error-code → i18n mapping

**Mapper:** `errorCodeToTranslationKey(errorCode)` — `src/lib/images/errorMapping.ts:28`. N→1 `switch` collapsing ~28 Worker/backend/client codes into **13 live keys** (`ImageErrorTranslationKey`, `:13`). All in the `ERRORS` namespace:

| Key | Representative codes |
|---|---|
| `image.invalid` | `TOKEN_MISSING`, `TOKEN_MALFORMED`, `PATH_TRAVERSAL`, `INVALID_TYPE`, `INVALID_SCOPE`, `INVALID_COUNT`, `CONTENT_TYPES_MISMATCH`, `CHAT_ID_REQUIRED`, `CHAT_ID_NOT_ALLOWED`, `DIMENSIONS_TOO_LARGE`, `DECODE_FAILED` |
| `image.forbidden` | `TOKEN_SCOPE_MISMATCH`, `TOKEN_KEY_MISMATCH`, `NOT_CHAT_MEMBER` |
| `image.bad.format` | `CONTENT_TYPE_NOT_ALLOWED`, `CONTENT_TYPE_MISMATCH` |
| `image.too.big` | `FILE_TOO_LARGE` |
| `image.rate.limited` | `RATE_LIMITED` (interpolates `retryAfterSec`) |
| `image.server.error` | `R2_WRITE_FAILED`, `R2_READ_FAILED`, `INTERNAL`, `PROCESSING_FAILED` |
| `image.token.expired` | `TOKEN_EXPIRED` |
| `image.session.expired` | `TOKEN_ALREADY_CONSUMED`, `TOKEN_REISSUE_FAILED`, `UNAUTHENTICATED`, `TOKEN_SIGNATURE_INVALID`, `TOKEN_ISSUER_INVALID` |
| `image.not.owner` / `image.in.use` / `image.too.old` / `image.invalid.key` | DELETE-path codes `NOT_OWNER` / `IMAGE_IN_USE` / `IMAGE_TOO_OLD` / `INVALID_KEY` |
| `image.upload.failed` | catch-all (`NETWORK_ERROR`, `UNKNOWN`, unmapped) |

**Live vs. fallback — DIVERGENCE D5.** The spec describes 10 active + 7 "phase-8 reserved" keys with inline `TODO(phase 7)` markers. In code there are **no `TODO(phase 7)` comments**; all 13 keys are resolved through `safeT(t, key)` (`:110`), which treats next-intl's "return the key on miss" as `null` and falls back to `englishFallback(err)` (`:138`) — full inline-English coverage for every key. So the keys are "live with a graceful English safety net," not pending stubs. The 4 DELETE-path keys are mapped but today only surface in `console.warn` (cleanup is fire-and-forget), not user toasts.

**Toast builder:** `buildUploadErrorTitle(err, tErrors)` (`:121`) — picks the key, interpolates (`{retryAfterSec}` for rate-limit, `{filename}`/`{code}` for the catch-all), falls back to English.

**Processing stage labels (separate subsystem):** `stageLabel` / `processingMessage` (`:215`/`:233`) render `INPUT`-namespace `image.processing.<stage>` keys. Note the **Part-6 parent/child-collision handling**: `uploading` and `complete` use `.label` suffixes (`STAGE_KEY_OVERRIDES`, `:206`) so the bare label can coexist with sibling `.with.size` / `.with.sizes` parameterized keys. Fallback chain: specific key → `image.processing.default` → `englishStageLabel` inline English.

---

## 9. Config / env

| Var | Where read | Use | Default |
|---|---|---|---|
| `NEXT_PUBLIC_CDN_URL` | `variants.ts:13` | base for all public/private image URLs + watermark asset URL | none (required; `!` non-null asserted) |
| `NEXT_PUBLIC_WATERMARK_ENABLED` | `variants.ts:14` | gate hero `draw` watermark param | falsey unless literal `'true'` |
| `NEXT_PUBLIC_API_URL` | `api.ts:7` | `BACKEND_API` base — serves `/upload-tokens`, `/view-tokens`, DELETE | none (required) — image-adjacent, not image-specific |

No other image-specific env vars. `.env.local.example` carries sample values (`cdn-stage.oglasino.com`, watermark `false`).

---

## 10. Seams toward mobile

**Browser/DOM-bound pieces with no React Native equivalent** (all in `processImage.ts` / `imageDecoder.ts`):
- `heic2any` (`imageDecoder.ts:71`) — DOM/canvas HEIC decode. RN needs a native module (`expo-image-manipulator` doesn't do HEIC→JPEG transcode directly; may need a native HEIF bridge or rely on the OS picker already yielding JPEG).
- `browser-image-compression` (`processImage.ts:233`) — canvas-based resize/transcode + WebWorker. RN replacement: `expo-image-manipulator` / `react-native-image-resizer` (no quality-fallback loop built in — that two-pass logic must be reimplemented).
- `createImageBitmap` / `<img>` dimension read + `getImageData` PNG-alpha detection (`imageDecoder.ts`) — no canvas in RN; use native image metadata APIs, and decide whether PNG-alpha detection is even needed on mobile.
- `File`/`Blob` types throughout (`processImage.ts`, `uploadImages.ts` `body: blob`) — RN has no `File`; uploads use a URI/`FormData`-or-`fetch`-with-blob-polyfill shape. The PUT body construction is the main rewrite surface.
- *Pure, portable:* the threshold constants, format-decision rules, `errorMapping`, `variants.ts` URL builders, and both Zustand stores (`viewTokens`, `uploadProgress`) are platform-agnostic and copy across largely unchanged.

**Origin / CORS:** the PUT is a naked `fetch` (`uploadImages.ts:342`) with no manual `Origin` header — the browser adds it and the Worker validates against its CORS allowlist. RN `fetch` sends no `Origin`; per spec "CORS does not apply to React Native," so the Worker must accept the no-Origin case (it does, by design). Mobile does **not** need to spoof an Origin.

**Auth-token attachment assumptions:** token attach lives entirely in `api.ts` (web Firebase SDK `auth.currentUser.getIdTokenResult`, module-level token cache, `withCredentials: true`). Mobile must replicate this in its own axios/fetch layer: `@react-native-firebase/auth` (or the JS SDK) for the ID token; `withCredentials` is a no-op on RN (no shared cookie jar — mobile is Bearer-only, which this client already is for the actual auth signal). The `X-Base-Site`/`X-Lang` headers (`api.ts:97`) come from web's `getTenantLocale(storedLocale)`; mobile needs an equivalent base-site/locale source feeding the same two headers.

---

*End of audit. Read-only; no code changed.*
