# Audit — web upload abandonment handling

**Repo:** oglasino-web
**Branch:** dev (not switched)
**Date:** 2026-05-31
**Type:** read-only audit. No code changed.
**Question:** When a user uploads product images and abandons the flow BEFORE the
product is created, what does web do about the already-uploaded R2 images?

---

## TL;DR

1. **Image bytes are uploaded to R2 only at the final step (Step 4 of the create
   wizard), not when the user picks/processes a file.** Picking a file only puts a
   `File` into React state. The R2 `PUT` fires when `UploadedProductDialog` mounts
   and runs `createNewProduct`.
2. **There is no cleanup-on-abandonment code.** No `beforeunload` / `unload` /
   `visibilitychange` / `pagehide` handler, no React unmount/cleanup effect that
   aborts or deletes, no wizard-cancel handler that deletes, and no
   `AbortController` wired into the product create flow. The only `DELETE
   /secure/images/{key}` calls (`cleanupOrphanImages`) fire on **persistence
   failure** paths, not on abandonment.
3. **Web relies on nothing client-side for the abandonment case and leaves the
   orphans for the backend sweeper.** This is stated in the code's own comments.

---

## Q1 — When do the bytes actually get uploaded to R2?

**Only at the final "create" step. Picking/processing a file does not upload.**

### Picking a file = React state only, no upload

- `ImagesImport.onImport` reads the picked files, wraps each as `{ file }`, and
  hands them up via `setImages` — `src/components/client/ImagesImport.tsx:79-81`.
- That flows into the wizard's `productData.imagesData` via `handleChange` —
  `src/components/popups/dialogs/CreateNewProductDialog.tsx:125-127` and the
  Step 1 wiring at `:147-153`. Nothing here touches the network.

### The upload call chain (fires at Step 4)

- Step 4 of the wizard renders `UploadedProductDialog` —
  `src/components/popups/dialogs/CreateNewProductDialog.tsx:180-190`.
- On mount, a one-shot `useEffect` (guarded by `triggeredRef`) calls
  `createNewProduct(productData, …)` —
  `src/components/popups/components/UploadedProductDialog.tsx:45-54`.
- `createNewProduct` → `extractAndUploadImages` —
  `src/lib/service/reactCalls/productService.ts:158`.
- `extractAndUploadImages` filters the `imagesData` entries that have a `.file`
  and calls `uploadImages(...)` —
  `src/lib/service/reactCalls/productService.ts:128-133`.
- `uploadImages` processes each file, requests upload tokens
  (`POST /secure/images/upload-tokens`, `src/lib/images/uploadImages.ts:200-204`),
  then does the parallel `PUT`s —
  `src/lib/images/uploadImages.ts:153-166`.

### The exact wire call (bytes → R2)

- `tryPut` issues the raw-blob `PUT` to the presigned/worker `uploadUrl` —
  **`src/lib/images/uploadImages.ts:342-350`**:

  ```ts
  response = await fetch(token.uploadUrl, {
    method: 'PUT',
    headers: {
      'x-upload-token': token.token,
      'Content-Type': processed.blob.type,
    },
    body: processed.blob,
    signal,
  });
  ```

**Conclusion:** the bytes cross the wire to R2 at Step 4, triggered by
`UploadedProductDialog` mounting — i.e. after the user has clicked through
steps 1–3 and reached the final step. Steps 1–3 hold the files purely in browser
memory.

---

## Q2 — Is there ANY cleanup-on-abandonment code?

**None found for the abandonment case.** Detail by mechanism:

### `beforeunload` / `unload` / `visibilitychange` / `pagehide`

- **None found.** A repo-wide grep for `beforeunload`, `unload`,
  `visibilitychange`, `pagehide` across `src/` returns zero matches. There is no
  tab-close / navigation guard anywhere in the web app.

### React unmount / cleanup effect

- **None found.** The only `useEffect` that drives the upload is
  `src/components/popups/components/UploadedProductDialog.tsx:45-118` and it
  returns no cleanup function. Unmounting that dialog (e.g. wizard closes) does
  nothing to in-flight or completed uploads.

### Wizard-cancel handler that deletes

- **None found.** The wizard's `onClose` (`CreateNewProductDialog.tsx:185`,
  passed to `UploadedProductDialog`) and the dialog's close buttons
  (`UploadedProductDialog.tsx:179-184`, `:213`) only close the modal /
  `window.location.reload()`. No `cleanupOrphanImages` / `DELETE` call is wired
  to any cancel/close path.
- On Step 4 the dialog is also rendered with `closableOutside={false}` and
  `addCloseButton={false}` — `CreateNewProductDialog.tsx:213-214` — so the user
  can't dismiss it by clicking outside, but tab-close / browser-back / navigation
  is unguarded.

### `AbortController` tied to navigation/unmount

- **None found in the product create flow.** `createNewProduct` is invoked with
  no `signal` — `UploadedProductDialog.tsx:54` passes only `onProgress`. The
  pipeline supports an `AbortSignal` end-to-end (`uploadImages.ts:114`, forwarded
  to `fetch` at `:349`), but the product wizard never constructs an
  `AbortController`, so an in-flight upload cannot be cancelled by leaving the
  flow.

### `DELETE /secure/images/{key}` (orphan cleanup) — exists, but NOT on abandonment

- `cleanupOrphanImages` issues `BACKEND_API.delete('/secure/images/{key}')` —
  `src/lib/images/uploadImages.ts:401-413`.
- It is called **only on persistence-failure paths**, after R2 upload succeeded
  but the entity create/update did not:
  - product create: `src/lib/service/reactCalls/productService.ts:177, 183, 189`
  - product update: `src/lib/service/reactCalls/productService.ts:234, 240, 245`
  - chat send failure: `src/messages/store/useChatStore.ts:569`
  - review save failure: `src/lib/service/reactCalls/reviewService.ts:86, 91`
- All of these require `createNewProduct` (or the equivalent) to run to the point
  of receiving a non-2xx / thrown response from the backend `POST`. If the user
  abandons (closes the tab, navigates away) after the R2 `PUT` succeeds but
  before the create `POST` resolves, **none of these paths execute** — the JS
  context is torn down mid-promise and no `DELETE` is sent.

---

## Q3 — Does web leave orphans for the backend sweeper?

**Yes, for the abandonment case web relies on nothing client-side and leaves the
orphans for the backend sweeper.** This is the code's own stated design:

- `src/lib/images/uploadImages.ts:16-18` — "Earlier successes in a failed batch
  become R2 orphans — see `jobs/image_pipeline/FUTURE-CLEANUP.md`."
- `src/lib/images/uploadImages.ts:388-399` — `cleanupOrphanImages` doc:
  "best-effort… Failures are logged and swallowed — **the backend's scheduled
  sweeper will pick up anything we miss.**"
- `src/lib/images/uploadImages.ts:409` — warn message: "(sweeper will catch the
  rest)".

`cleanupOrphanImages` is a best-effort optimization on the
upload-succeeded-but-persist-failed path, not an abandonment guard. The
authoritative orphan reclamation is the backend sweeper.

---

## Orphan window (precise)

Because upload happens only at Step 4:

- **Abandon at Steps 1–3** (files picked, not yet at Step 4): **no R2 orphan** —
  the files never left the browser.
- **Abandon at Step 4, after the R2 `PUT` lands but before/while the create
  `POST` completes** (tab close, navigation, crash): **R2 orphan, no client
  cleanup** — the sweeper must reclaim it.
- **Create `POST` returns a handled failure** (4xx/5xx/network, dialog stays
  open): `cleanupOrphanImages(newKeys)` fires client-side
  (`productService.ts:177/183/189`). This is *not* the abandonment case, but it's
  the only case where web deletes orphans itself.

---

## Notes / out-of-scope observations (Part 4b)

- **Adjacent (low):** `src/components/popups/components/UploadedProductDialog.tsx:110`
  contains a `console.error(error)` in the non-`UploadError` catch branch. Not in
  this audit's scope and not introduced by this session — flagging only because it
  was read while tracing the upload trigger. Severity low (diagnostic, behind a
  type guard). I did not change it; read-only audit.

---

## Cleanup performed

- None needed (read-only audit, no code touched).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing.

## For Mastermind

- This is a Phase 2 audit; the deliverable is this file. No code written, no
  config-file edits required, no pending config drafts.
- The single seam worth surfacing for Phase 3: **web has no abandonment-time
  cleanup at all**, and the only orphan reclamation web contributes is the
  best-effort `cleanupOrphanImages` on persistence-failure. If the feature intends
  web to actively clean up on abandonment, that is net-new work (a
  `beforeunload`/`pagehide` `DELETE` and/or an `AbortController` wired through the
  Step 4 dialog) — none of it exists today. Otherwise the backend sweeper is the
  sole guarantee.
