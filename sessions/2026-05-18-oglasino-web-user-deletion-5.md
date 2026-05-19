# Session summary

**Repo:** oglasino-web
**Branch:** feature/user-deletion
**Date:** 2026-05-18
**Task:** Web Engineering Brief E — User Deletion Google-reauth flow investigation. Read the delete-account flow on disk, identify anything obviously wrong on close read, instrument the runtime path with `console.log` statements, write up findings. No functional fixes — fix brief comes after Mastermind reads the runtime capture.

## Implemented

- **Phase 1 — close read complete.** Read `DeleteAccountConfirmationDialog`, `userService.deleteCurrentUser`, `api.ts` (both interceptors), `AccountStateDialogsInit`, `useAuthStore`, `authService.syncUserToBackend`, `DialogManager` + `dialogRegistry`, `InfoDialog`, plus the supporting `DrawerDialog`, `SessionGuard`, `UseTokenRefresh`, `AppInit`, `app/[locale]/layout.tsx`, `app/[locale]/owner/user/page.tsx`, `useDialogStore`. Findings below.
- **Phase 2 — instrumentation in place.** Added 25+ `[user-deletion:debug]` log points across five files at every required diagnostic step in the brief. Errors go through `console.error` with the full object spread; normal flow uses `console.log`.
- **Phase 3 — verification.** `npx tsc --noEmit` clean. `npm run lint` 0 errors / 208 warnings (same count as Brief D's start — no new lint warnings introduced; the new `console.log` calls do not trip any rules).
- **Phase 4 — runtime-capture playbook for Igor.** See "For Igor — capture instructions" section at the bottom of this file.

## Files touched

- `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx` (+57 / -4)
- `src/lib/service/reactCalls/userService.ts` (+9 / -0)
- `src/lib/config/api.ts` (+32 / -1)
- `src/components/client/initializers/AccountStateDialogsInit.tsx` (+18 / -0)
- `src/lib/service/reactCalls/authService.ts` (+12 / -0)

## Tests

- Ran: `npx tsc --noEmit` — clean.
- Ran: `npm run lint` — 0 errors / 208 warnings (identical to Brief D baseline).
- No new tests added — Phase 2 is `console.log` instrumentation only.

## Findings from code read

Each finding is `file:line` + what looks wrong + which Igor symptom it explains. Confidence ranked at the bottom.

### Finding A — Loading state and dialog state are never cleared on the success path (HIGH)

**File:** `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:85-98`

The success-path try block sequence is:
1. `await currentUser.getIdToken(true)`
2. `await deleteCurrentUser(freshToken)` → backend returns 200 (confirmed by Igor's backend log at `POST /api/secure/user/me/delete -> 200 (788 ms)`)
3. `sessionStorage.setItem('account-just-deleted', scheduledDeletionAt)`
4. `await auth.signOut()`

**No `setLoading(false)` and no `onClose()` are called on success.** The inline comment justifies this with "SessionGuard drops the user from the protected page; this dialog unmounts naturally."

**Why that's wrong.** Two architectural facts make the "unmounts naturally" assumption false:

- `DeleteAccountConfirmationDialog` is rendered by `<DrawerDialogManager />` at the root layout (`app/[locale]/layout.tsx:59`), **outside** the `<SessionGuard>` subtree (which only wraps `app/[locale]/owner/layout.tsx:27`). Whatever SessionGuard does to the page tree has no effect on the dialog manager's render — the dialog manager reads `useDialogStore.currentDialogId` and renders accordingly. Nothing in the success path clears that store.
- `SessionGuard.tsx:24-54` runs its check exactly once on mount via `await auth.authStateReady()`. It does not subscribe to `onAuthStateChanged` / `onIdTokenChanged`. When `auth.signOut()` fires from inside a guarded page, the guard does not re-run its `auth.currentUser === null` check — `loading` stays `false`, children stay rendered.

Net effect at runtime — backend deletion succeeds, sessionStorage flag is set, Firebase signs the user out — but the dialog keeps `currentDialogId === 'deleteAccountConfirmationDialog'` and local `loading === true`. The button keeps spinning. The user stays on `/owner/user`. **Matches Igor's exact wording: "Something changes behind the dialog (probably I get logged out). The dialog stays open. The button stays in its loading/spinner state. No redirect happens."**

Why a refresh resolves the visible symptom — SessionGuard does run on every fresh mount. After F5 it sees `auth.currentUser === null` and `router.replace('/<locale>')`. The new home-page mount instantiates `<AccountStateDialogsInit />` once, the useEffect reads `account-just-deleted`, opens the post-deletion dialog.

**Confidence: HIGH.** The runtime trace will confirm that all the success-path log points fire (through "after auth.signOut()") and no further code runs — exactly the symptom.

### Finding B — Post-deletion dialog renders left-aligned + has a hidden duplicate `DialogDescription` (MEDIUM)

**File:** `src/components/popups/dialogs/InfoDialog.tsx:59-64` interacting with `src/components/popups/dialogs/DrawerDialog.tsx:82-86`

Brief D's fix routed JSX-typed descriptions through `<DialogDescription asChild>`. The branch reads:

```tsx
{dialogDescription &&
  (typeof dialogDescription === 'string' ? (
    <DialogDescription className="mb-4 text-center">{dialogDescription}</DialogDescription>
  ) : (
    <DialogDescription asChild>{dialogDescription}</DialogDescription>
  ))}
```

For the post-deletion dialog, `AccountStateDialogsInit.tsx:68-77` passes a JSX `<div className="flex flex-col gap-3 text-left">…</div>`. The `asChild` pass-through means **no centering class is applied** — the caller's `text-left flex flex-col gap-3` is the rendered alignment. The visible block sits left-aligned.

The surrounding dialog renders:
- **Title:** `DialogTitle className="text-center"` (DrawerDialog.tsx:83) — centered.
- **Outer hidden `<DialogDescription>`:** DrawerDialog.tsx:84 unconditionally renders `<DialogDescription className="text-center" hidden={dialogDescription ? false : true}>{dialogDescription}</DialogDescription>`. InfoDialog passes `dialogDescription=""` to DrawerDialog, so this element is in the DOM as `<p hidden class="text-center">`. It's invisible but still present, and it carries the description's a11y semantics ahead of the inner real description.
- **Inner `<DialogDescription asChild>` from InfoDialog:** renders as the caller's left-aligned div.
- **Single close button:** centered (InfoDialog.tsx:68 `flex w-full items-center justify-center`).

The result: centered title + left-aligned multi-line body + centered single button. The asymmetry is what makes it "look ugly and not centered." **There may also be a Radix dev-mode console warning about two `DialogDescription` elements in one Dialog** — worth Igor confirming in the captured console.

Note this is **regression of intent, not Brief D being wrong**. Brief D had to break the `<p>`-around-`<div>` invalid-HTML problem; using `asChild` was the right structural fix. What got lost was the centering. The right end-state is either (a) caller-controlled centering (the caller's `<div>` gets `text-center items-center`), or (b) a wrapper inside `InfoDialog` that centers regardless of node vs string. That decision is for the fix brief.

**Explains Igor symptom: "looks ugly and not centered."**

### Finding C — `X-Account-Restored` response header may not reach axios due to CORS exposure (HIGH suspect)

**File:** `src/lib/config/api.ts:25-31`

The response-success interceptor reads `response.headers['x-account-restored']`. For cross-origin XHR responses, browsers expose only the [CORS-safelisted response headers](https://fetch.spec.whatwg.org/#cors-safelisted-response-header-name) (`cache-control`, `content-language`, `content-length`, `content-type`, `expires`, `last-modified`, `pragma`) **unless the server includes the header in `Access-Control-Expose-Headers`**. `X-Account-Restored` is not safelisted.

If the backend's CORS configuration does not include `Access-Control-Expose-Headers: X-Account-Restored` (or `*` with credentials caveats), axios's `response.headers` will not contain the key — even though it's present in the raw HTTP response. The interceptor silently skips `setRestored(true)`. The restoration dialog never opens.

I cannot verify this from the frontend. The Phase 2 instrumentation logs the value the interceptor sees on every firebase-sync response — if the backend log shows the header was set on the response but the frontend log shows `xAccountRestored: undefined`, CORS exposure is the smoking gun.

The fix is backend-side (one line in `WebMvcConfigurer`'s CORS config, or in whichever filter handles CORS for the worker). Not a frontend fix — but the **diagnosis** lives here, and any web-side workaround (e.g. an alternative wire mechanism via the response body) is the wrong shape per the spec.

**Confidence: HIGH suspect** because the symptom matches: re-login completes silently, no restoration dialog. **Confidence pending runtime trace** because there are other paths to the same symptom (Finding D below).

### Finding D — `restored`-watcher useEffect has a stale closure risk if React StrictMode is on (LOW–MEDIUM)

**File:** `src/components/client/initializers/AccountStateDialogsInit.tsx:90-98`

The dep array is `[restored, openDialog, setRestored, tCommonSystem]`. The effect opens the dialog then immediately calls `setRestored(false)`. In React 18 StrictMode + dev mode, effects run twice on mount in development. If `restored` was true on first mount (write-then-mount race), the effect would fire, open dialog, set restored=false. Second invocation in StrictMode sees restored=false and exits. That's actually fine.

The more interesting race: if `setRestored(true)` is invoked from the response interceptor while `AccountStateDialogsInit` is mid-render, the useEffect fires post-commit. The dialog opens. Fine. But the interceptor is set up once per app load (module-scoped `BACKEND_API` instance) — it fires for ALL responses across the entire app session. If a *second* firebase-sync response also has `X-Account-Restored: true` (which would only happen on a separate restoration event, which shouldn't be possible in one session), the dialog reopens. Not a current bug, just fragile.

**Why this is worth noting for Issue 3e:** the `UseTokenRefresh` listener (`UsetTokenRefresh.tsx:9-30`) and `loginWithGoogleFirebase` BOTH POST `firebase-sync` on a fresh sign-in. The backend logs in the brief confirm two `firebase-sync` calls per login event. If the backend only sets `X-Account-Restored: true` on the FIRST sync (the one that actually transitioned the deletion state) and the SECOND sync sees an already-active user with no header — and if the order back from the network swaps — there's no race, because the interceptor only ever SETS the flag, never clears it. So the flag should land true regardless of order. The only failure mode is "neither response carries the header" — which points back to Finding C.

**Confidence: LOW for the race, MEDIUM for the observation that this codepath has two concurrent `firebase-sync` POSTs on every login.**

### Finding E — Danger Zone styling is unmarked-destructive (LOW, explains Issue 1)

**File:** `app/[locale]/owner/user/page.tsx:361-381`

The section is structurally:
- `<div className="border-border-mild mt-6 flex w-full flex-col gap-3 border-t pt-6">`
- `<h2>Danger Zone</h2>` (no special styling)
- `<h3>Delete account</h3>`
- `<ul>` of four bullet consequences
- `<p className="italic">` restore note
- `<Button variant="destructive">`

The `border-border-mild` border is the same one every other settings block uses (compare lines 268, 275, 286, 298, 311, 323 — identical class). The h2 is `text-lg font-semibold` — same family as the rest of the page. The only "danger" signal is the literal text "Danger Zone" + `variant="destructive"` on the button. The button's destructive variant in this repo (shadcn defaults) renders as a muted-tone fill — visually similar to a regular outline button if the theme tokens are set the way they are.

Brief D doesn't include this finding because it wasn't in the PR review patch — it's a fresh visual complaint from manual testing. **Fix is out of scope this brief**; for the fix brief, the natural moves are (a) a wrapper class with `border-destructive bg-destructive/5` (or repo-appropriate equivalent), (b) a danger icon next to the h2, (c) a visually distinct fill on the button. The brief explicitly said not to touch styling here.

### Finding F — Confirmation dialog itself has no destructive treatment (LOW, explains Issue 2)

**File:** `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:117-159`

Inherits `DrawerDialog`'s neutral chrome (centered title in `DialogTitle`, neutral border). The action button uses `variant="destructive"` (same as the Danger Zone button on the page) — Igor reports the same complaint about subdued styling. The cancel button uses `variant="outline"`. There is no destructive iconography in the dialog, no red banner, no warning glyph next to the title.

Same disposition as Finding E — fix is out of scope this brief. Worth flagging that the fix brief should treat E and F together so the visual language is consistent (e.g., the same icon and color tokens used in both places).

### Finding G — `auth.signOut()` triggers `UseTokenRefresh.onIdTokenChanged(null)` which races against `AccountStateDialogsInit`'s sessionStorage read (LOW, observation only)

**File:** `src/components/client/initializers/UsetTokenRefresh.tsx:9-30`

After `auth.signOut()`, `onIdTokenChanged` fires with `firebaseUser = null`. `UseTokenRefresh` writes a null cookie via `/api/auth/token`. No `firebase-sync` is POSTed in the null branch.

This is fine and not implicated in any of Igor's symptoms, but worth noting: after the dialog's success path runs `auth.signOut()`, `UseTokenRefresh` is the only listener that runs. `AccountStateDialogsInit`'s post-deletion-flag read only happens on `AccountStateDialogsInit` mount — which only happens on a fresh root-layout mount. Today, after `auth.signOut()` the user stays on the same page (Finding A), so `AccountStateDialogsInit` does not re-mount, so the flag sits in sessionStorage until the next page load.

If Finding A is fixed by calling `router.replace('/<locale>')` from the success path AFTER setting the sessionStorage flag, `AccountStateDialogsInit` will re-mount on the home-page render and pick up the flag. That's the cleanest fix path and matches the existing flag-on-mount design.

### Finding H — `useAuthStore.loginWithGoogle` does NOT clear `restored=false` before signing in (LOW, observation)

**File:** `src/lib/store/useAuthStore.ts:179-196`

If a prior session's `setRestored(true)` was never cleared (e.g., the dialog never auto-dismissed because the user navigated away), `restored` stays `true` in the store. On a fresh login, `AccountStateDialogsInit`'s `restored`-watcher would immediately re-fire and open a stale restoration dialog. The `setRestored(false)` inside the watcher does eventually clear it, but the wrong title may flash. Low risk; the auto-dismiss timer (10s) makes this unlikely in practice.

### Sanity check — did the deletion actually reach the backend?

**Yes.** Igor's backend logs in the brief show:
- `POST /api/secure/user/me/delete -> 200 (788 ms)` at 10:22:21.680
- `insert into user_deletion_audit_log` with `triggered_by = 'user_request'` at 10:22:21.656
- `update users set deletion_status='PENDING_DELETION'` at 10:22:21.664
- `update user_deletion_requests set status='PENDING'` at 10:22:21.669

The deletion request was sent, the backend acknowledged it, the audit log was written, the user row was flipped to `PENDING_DELETION`, and a `user_deletion_requests` row was created. The frontend's stuck-dialog symptom is purely the result of the success-path code never closing the dialog (Finding A) — the deletion itself is already on disk and the 7-day grace timer is running.

This is the most important diagnostic answer Mastermind asked for in the brief: **the deletion was sent and acknowledged**; everything downstream is presentation/state-cleanup bugs, not deletion-flow bugs.

## Cleanup performed

- None. The session is instrumentation-only. The `console.log` lines added here are temporary — see "For Mastermind" item 5 below for the recommendation on what to do with them in the fix brief.

## Config-file impact

- **conventions.md:** no change. No new conventions surfaced; the existing conventions cover this work.
- **decisions.md:** no change. Per spec §20.8 the user-deletion decisions entry is drafted post-merge.
- **state.md:** no change. Status stays where Mastermind/Docs/QA last left it; this is mid-stream investigation, not a status flip.
- **issues.md:** no change in this session. Finding C (`X-Account-Restored` CORS exposure) is a backend-side observation; if confirmed by Igor's runtime trace and not absorbed into the fix brief, it's a candidate for an `issues.md` entry — but that decision belongs to Mastermind, not to me.

## Obsoleted by this session

- Nothing. Instrumentation-only changes; the existing flow is not refactored.

## Conventions check

- **Part 4 (cleanliness):** intentionally violated, but bounded. The brief explicitly authorizes `console.log` for the duration of this investigation. Lint count is unchanged (0 errors / 208 warnings). The console.log calls all carry the `[user-deletion:debug]` prefix for easy grep / removal. **The fix brief must remove or upgrade them — see "For Mastermind" item 5.**
- **Part 4a (simplicity):** confirmed. No new abstractions. One small helper (`decodeJwtAuthTime`) added locally to `DeleteAccountConfirmationDialog`; it lives only inside `handleDelete`'s file and exists to make Phase 3's runtime trace useful. Removed at the same time as the console.log calls.
- **Part 4b (adjacent observations):** see Findings E, F, G, H above. E and F are the visual issues called out in the brief; G and H are bonus low-severity observations surfaced while reading.
- **Part 6 (translations):** N/A this session — no translation keys touched.
- **Part 7 (error contract):** observed but unchanged. The `isErrorWithCode` helper handles both `error.response.data.errors` and `error.data.errors` shapes (the unwrapped axios rejection from the BACKEND_API interceptor). The Phase 2 logs surface both.
- **Part 11 (trust boundaries):** N/A this session — no new surface area.

## Known gaps / TODOs

- None. The fix brief is the natural follow-up; it will (a) close the success-path stuck-dialog (Finding A), (b) restore centering on the post-deletion dialog (Finding B), (c) reach a verdict on Finding C with Igor's captured `xAccountRestored` log line, (d) apply destructive styling to the Danger Zone section and the confirmation dialog (Findings E + F), and (e) remove or upgrade the `console.log` instrumentation added here.

## For Igor — capture instructions (Phase 3)

When you reproduce the failing flow with DevTools open, capture:

**Console tab** — filter on `[user-deletion:debug]`. Capture every log line from button click through whatever the final state is. The lines you'll see in a successful run:

1. `handleDelete entry; provider=google.com`
2. `before reauthenticateWithPopup (google)`
3. `after reauthenticateWithPopup (google) — OK`
4. `before currentUser.getIdToken(true)`
5. `after getIdToken(true); first20=eyJhbGciO... auth_time=<unix-ts>`
6. `userService.deleteCurrentUser request {...}`
7. `request interceptor: caller-supplied Authorization respected {...}`
8. `response interceptor success { url: '/secure/user/me/delete', status: 200, xAccountRestored: undefined }`
9. `userService.deleteCurrentUser response { status: 200, data: { scheduledDeletionAt: '2026-05-25T...' } }`
10. `after deleteCurrentUser; scheduledDeletionAt=<iso>`
11. `before sessionStorage.setItem(account-just-deleted)`
12. `before auth.signOut()`
13. `after auth.signOut() — handler reached end of success path`

If lines 1–13 all appear and no dialog close fires after, Finding A is confirmed.

**Network tab** — filter on `me/delete`. Capture:
- Request headers (especially the `Authorization` header — first ~30 chars are fine).
- Request body (will be `{}`).
- Response status (expect 200).
- Response body (expect `{"scheduledDeletionAt":"..."}`).

Then filter on `firebase-sync` for both the deletion flow AND the subsequent re-login. For the re-login firebase-sync responses, capture:
- Response headers — **specifically look for `X-Account-Restored: true`**. If present in headers but the console shows `xAccountRestored: undefined`, that confirms Finding C (CORS exposure).

**Page sequence to capture:**
- After clicking the destructive button: dialog opens — confirm it sits.
- After Google popup completes: the dialog you SEE.
- After F5: what URL you land at, what dialog opens, how it's aligned.
- After re-login: do you see the restoration dialog? Capture the console output during the login.

**Anything red in console** — unhandled promise rejections, React errors, Radix warnings (especially anything mentioning "DialogDescription").

## For Mastermind

1. **Ranked suspect list per issue.**

   **Issue 1 — Danger Zone doesn't look dangerous.** Finding E. **Confidence: HIGH** (this is the code on disk; pure visual). One-line(ish) fix: wrap the section with destructive-tinted classes, change the button to a fully destructive treatment. Trivial fix-brief work.

   **Issue 2 — Confirmation dialog doesn't look dangerous.** Finding F. **Confidence: HIGH.** Co-equal with E; should be solved together so the visual language is consistent across the danger-zone surface and the confirmation step.

   **Issue 3 — Google reauth flow appears broken.** Three sub-issues:

   - **3a (dialog stays open, button stays spinning, no redirect):** Finding A. **Confidence: HIGH.** Success path does not close the dialog or clear loading. The author's "SessionGuard drops the user from the protected page; this dialog unmounts naturally" comment is incorrect: the dialog lives at root layout, outside SessionGuard. Fix shape: after `auth.signOut()`, explicitly `closeDialog()` + `router.replace('/<locale>')`. (Order matters: sessionStorage set → signOut → closeDialog → router.replace.)
   - **3b (post-deletion dialog ugly and not centered):** Finding B. **Confidence: HIGH.** Brief D's `asChild` pass-through dropped centering for the JSX-child branch of `InfoDialog`'s DialogDescription. Fix shape: either change the caller's `<div className="...text-left">` to `text-center items-center`, or restructure `InfoDialog` so it owns the centering regardless of child type. Likely the latter — `text-left` is suspicious; the entire restoration / banned / post-deletion design seems to want centered. Worth confirming with Igor.
   - **3c ("Go to home" button on home page):** Not in findings — the `closeButtonLabel` for the post-deletion dialog is literally `buttons.account.deleted.go.home.label`, which translates to "Go to home." After F5, the user is already on home — clicking it just closes the dialog. **Severity: LOW.** Fix shape: rename the translation key to `buttons.account.deleted.acknowledge.label` (or similar) and let the close button just close. Backend Engineer's SQL job for the new translation key per Conventions Part 6.
   - **3d (no restoration dialog after sign-in):** Finding C. **Confidence: HIGH suspect, pending runtime trace.** If Igor's captured console shows `xAccountRestored: undefined` while the Network tab shows the header IS in the response, CORS exposure is the cause and the fix is in the backend (`Access-Control-Expose-Headers: X-Account-Restored`). If the captured log shows `xAccountRestored: 'true'` BUT the dialog still doesn't open, the bug is in the watcher's render lifecycle (closer look at Finding D). **The runtime trace will pick between these.**

2. **Are any of these one-line fixes?**

   - 3a: ~3-5 lines (close dialog, navigate, choose ordering). Trivial. Could fit in a small follow-up brief by itself if Mastermind wants 3a out the door faster than the rest.
   - 3b: ~2-3 lines (depending on whether the fix is in the caller or in InfoDialog). Trivial.
   - 1 + 2 (visual): both moderate — depends on how thorough the redesign is. Could land as one brief.
   - 3c: 1 line in the frontend + 1 translation key in the backend SQL seed. Trivial both ways.
   - 3d: 1-line backend fix IF Finding C holds. Web side has nothing to do unless the runtime trace surprises us.

   So: **3a + 3b + 3c + 1 + 2 are all small.** If 3d turns out to be CORS, it's also small (backend side). The single fix brief that bundles all of these is reasonable.

3. **Anything else worth noting from the close read.**

   - The `<DialogDescription>` double-render in `InfoDialog` (Finding B / F) is structural and may be why Brief D's PR-review Finding #1 needed `asChild` in the first place. Worth Mastermind asking whether the right move is to push the description-rendering responsibility entirely into `InfoDialog` (and pass `dialogDescription=undefined` to `DrawerDialog`), eliminating the always-rendered hidden outer description. That's a small refactor and would also reduce the risk of future "two descriptions" Radix warnings.
   - `UseTokenRefresh` POSTing `firebase-sync` on every `onIdTokenChanged` (Finding G's neighbour) means every login causes two `firebase-sync` calls in close succession. Not a deletion bug, but it's load Mastermind may want to know about — already noted in the original audit "For Mastermind" item 5.

4. **Recommendation on the fix brief scope.**

   Single brief that addresses Issue 1, Issue 2, Issue 3a, Issue 3b, Issue 3c, and reaches a verdict on Issue 3d (web-side or backend-side, depending on Igor's runtime capture). All findings I'm confident about are small in-place edits. The instrumentation removal goes in the same brief as a cleanup step at the end.

5. **`console.log` disposition recommendation.**

   This codebase does not have a logger abstraction beyond `console.warn` / `console.error` calls scattered through `authService.ts`, `useAuthStore.ts`, `serviceLog.ts`. There is no leveled logger, no production-vs-dev gate, no telemetry plumbing. The Phase 2 logs I added are debug-quality output — useful for an investigation, not appropriate for production console.

   **Recommendation: the fix brief should DELETE all `[user-deletion:debug]` log points.** The places where post-launch diagnostic value is most likely:
   - `api.ts` response interceptor's `X-Account-Restored` check — if this becomes the failure mode again later, having a log when the header is detected is useful. But adding it now without a logger means production users see noise in their dev console.
   - `syncUserToBackend`'s response disabled/deletionStatus log — same reasoning.

   Both belong in a (future) shared logger that gates on a feature flag or dev-mode. Until then, delete them. **None of the instrumentation should ship to main.**

6. **Adjacent observations recap.**

   - Finding E (Danger Zone visual) — to be fixed in the next brief.
   - Finding F (Confirmation dialog visual) — to be fixed in the next brief.
   - Finding G (signOut → UseTokenRefresh observation) — informational, no action.
   - Finding H (login does not clear `restored=false` first) — informational, defensive fix optional.
   - Brief D adjacent items (#3, #4, #7, #8, #10, #11, #12 from the PR review) — unchanged from session 4's For Mastermind item 4; not re-surfaced here.
