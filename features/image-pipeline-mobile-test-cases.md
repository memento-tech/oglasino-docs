# Image Pipeline — Mobile On-Device Test Cases

On-device smoke test cases for the `oglasino-expo` image pipeline. This is the gate-2 / Ψ artifact: it is executed by Igor on real devices (mid-range Android + iPhone) against a rebuilt dev/preview build. Passing this is the gate before the image-pipeline Expo-backlog row flips to `mobile-stable`.

**Prerequisite:** the pending iOS+Android rebuild must have landed (the build must carry the current `new-expo-dev` image-pipeline code). On-device smoke against a stale build is not meaningful.

## Scope

Covers the four upload surfaces (product, profile, chat, review) plus display, progress text, and failure paths. Does not cover backend or web (those are separately verified).

## Test cases

### Upload — product create
1. Create a product, add 1 photo from camera. Expect: photo processes, progressive status text shows (Checking → Resizing → Compressing → Uploading → Done), product creates with the image visible.
2. Create a product, add 5 photos from gallery in one pick. Expect: all 5 process and upload; per-file status text visible; product shows all 5.
3. Add a HEIC photo (iPhone). Expect: converts to JPEG at the picker boundary; status text shows the HEIC-conversion stage in the device language (not English on a non-EN locale); uploads successfully.
4. Add a photo, then abandon (background the app / leave the wizard) before tapping create. Expect: no crash; on return, no half-state. (Orphaned bytes, if any, are reclaimed by the backend sweeper — not a device-visible outcome.)

### Upload — profile picture
5. Set a profile picture from gallery. Expect: processes, uploads, new avatar displays.
6. Replace an existing profile picture. Expect: new one displays; no error.

### Upload — chat attachment
7. Send a photo in a chat. Expect: uploads, appears in the thread, the other participant can view it (private view-token path).
8. Send a photo, then immediately background the app. Expect: on return, the message either completed or shows a clear failure — not a silent vanish.

### Upload — review image
9. Leave a review with 1–3 photos. Expect: uploads via the product-scope path; review posts with images.

### Display
10. Open a product with multiple photos. Expect: images load via expo-image; card/hero variants render; no broken-image placeholders on a normal connection.
11. Open the same product offline (airplane mode) after it was viewed online. Expect: cached images still render (expo-image disk cache).

### Failure / edge paths
12. Attempt upload on a flaky/slow connection. Expect: retry behavior per policy (5xx backoff, network-error single retry); a 429 honors Retry-After and retries once (mobile divergence from web — this is expected).
13. Pick an oversized image (> processing limits). Expect: a clear, translated error — not a silent failure or crash.
14. Trigger an upload that fails at the create/persist step (e.g. force a backend error). Expect: the already-uploaded image is cleaned up (orphan DELETE fires on the persistence-failure path), and the user sees a translated error.

## Pass criteria

All 14 cases pass on both a mid-range Android device and an iPhone. Any failure blocks the `mobile-stable` flip; file the failure in `issues.md` and route to a fix chat.

## Notes

- HEIC conversion is native (`expo-image-manipulator`), not `heic2any` (web-only).
- The 429 Retry-After divergence (mobile retries once; web surfaces immediately) is a deliberate platform difference — case 12 should confirm mobile's behavior, not flag it as a bug.
- Progress text (cases 1–3) was wired in chat H session `oglasino-expo-image-pipeline-5`; before that the dialog showed only a spinner.
