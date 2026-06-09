# Session summary

**Repo:** oglasino-expo
**Branch:** dev (brief said `new-expo-dev`; the repo is currently on `dev`. The finished analytics work and the ATT removal are committed on `dev` — commits `ddaab53` → `bfbcbce`. Read-only audit, no checkout performed.)
**Date:** 2026-06-06
**Task:** Confirm the mobile analytics + consent model exactly as it ships in the current code (for a Privacy Policy that describes the mobile analytics-consent mechanism, distinct from the web cookie banner).

## Implemented

Read-only audit. No code changes. Each of the brief's five statements is confirmed or corrected against the code below, with file citations.

---

## The as-built model (core finding)

**One device-local "allow analytics" decision, default-deny, gating a first-party GA4 (Firebase) analytics SDK. No App Tracking Transparency, no IDFA, no ad/marketing signals, no cross-app tracking.** First-run prompt + settings toggle both present.

This is deliberately *not* the web four-signal cookie model — there are no cookies, no Google Consent Mode, and (as it now ships) no ATT layer. The single consent boolean maps directly onto the analytics SDK's collection switch.

### Statement 1 — single device-local "allow analytics" decision, on-device, default-deny — **CONFIRMED**

- **One axis.** `ConsentDecision = { analytics: boolean; decidedAt: number; version: 1 }` — `src/lib/types/consent/ConsentDecision.ts`. `analytics` is the only user-controlled signal.
- **Storage location.** AsyncStorage, key `'consent_decision'`, written/read by the per-concern module `src/lib/storage/consentStorage.ts` (house pattern — own key constant + direct AsyncStorage, like `themeStorage`/`softUpdateDismissal`). A reactive `src/lib/store/useConsentStore.ts` (Zustand) mirrors it write-through so UI and the gate read one live value.
- **Default.** Before any decision the store holds `decision: null` (and `getConsentDecision()` returns `null` when the key is absent or the stored value is malformed). `null` is treated as "no decision yet" = analytics **off**. Default-deny — see Statement 2.
- A corrupt/unreadable stored value defaults to `null` (deny), never throws (`consentStorage.ts:21-42`).

### Statement 2 — consent gates the analytics SDK init; no init/collection unless granted — **CONFIRMED (at the application layer), with one native-layer caveat to flag**

Confirmed at the app/JS layer:
- **The gate.** `isAnalyticsConsentGranted()` — `src/lib/consent/analyticsGate.ts` — returns `true` only when `useConsentStore.getState().decision?.analytics === true`; absent/`null`/not-hydrated/any throw → `false`. Synchronous, default-deny.
- **Init is gated on it.** `syncAnalyticsState()` — `src/lib/analytics/analyticsClient.ts:38-69` — reads the gate; when **closed** it does **not** call `getAnalytics()` (so the SDK instance is never created) and dispatch can never fire. When **open** it lazily inits once and calls `setAnalyticsCollectionEnabled(instance, true)`.
- **Dispatch is also gated.** Every GA4 event flows through `track()` — `src/lib/analytics/track.ts:17-30` — which triple-guards: SDK initialized, `isAnalyticsConsentGranted()`, instance present. It is the only caller of `logEvent`.
- **Reactivity.** `AnalyticsInit` (`src/components/init/AnalyticsInit.tsx`, mounted from `AppInit`, which runs only at `bootStatus === 'ready'`) reconciles at launch and re-runs `syncAnalyticsState()` whenever the consent decision flips — so turning analytics off stops collection without an app restart.

**Caveat worth knowing for the Privacy Policy (see "For Mastermind", adjacent finding):** the default-deny path in `syncAnalyticsState()` only *returns* — it never proactively calls `setAnalyticsCollectionEnabled(false)` (it does so only if the SDK was already initialized). There is **no `firebase.json` in this branch** (never tracked in git history; not on disk) pinning native auto-collection off. So at the native layer, default-deny relies on the app never initializing the SDK rather than on an explicit "collection disabled" instruction. For the policy's strongest claim ("nothing is collected until you opt in"), this native default is the one residual gap — flagged, not in scope to fix here.

### Statement 3 — NO App Tracking Transparency; no IDFA / ad-tracking permission request — **CONFIRMED (and this is a change from the original plan)**

- **Dependency removed.** `expo-tracking-transparency` (`~6.0.8`) was added (commits `ddaab53`, `963893f`) and then removed in commit `d348b4b` "Remove unused deps" (2026-06-05). It is gone from `package.json`/`package-lock.json`.
- **Call site removed.** The ATT call (`requestTrackingPermissions…`) lived in `AnalyticsInit.tsx` and was removed in commit `963893f` (−36 lines in that file).
- **Zero references today.** No `tracking-transparency` / `requestTrackingPermissions` / `getTrackingPermissions` / `IDFA` / `NSUserTracking` anywhere in the working tree.
- The current code documents this explicitly: "First-party analytics only (no IDFA/ads), so no App Tracking Transparency." (`AnalyticsInit.tsx:13`, `analyticsClient.ts:17`). Collection depends **solely** on the consent decision.

The app does **not** request IDFA / ad-tracking permission. ✅

*Important for accuracy:* this **diverges from the documented plan.** The spec and decision log still describe a `(ATT-allows-on-iOS, or Android) AND isAnalyticsConsentGranted()` init (`decisions.md` 2026-06-02 "consent+ATT-gated init"; 2026-05-30/05-31 consent entries; stale references in `analyticsGate.ts:9` and `AnalyticsInit.tsx` JSDoc). The shipped code has no ATT. The Privacy Policy should follow the **code** (no ATT), and the docs need correcting — drafted in "For Mastermind."

### Statement 4 — first-party only (GA4 via the app's own Firebase data stream); no ad/marketing signals, no cross-app tracking — **CONFIRMED**

- **First-party GA4 via Firebase.** Analytics is `@react-native-firebase/analytics` (`analyticsClient.ts:1-6`) — GA4 through the app's own Firebase app. Per-tier Firebase config files wire the data stream (`app.config.ts` `googleServicesFile` ternaries → `GoogleService-Info.*.plist` / `google-services.*.json`). Per `decisions.md` 2026-06-02 it reports into the same GA4 property as web via the mobile Firebase **App ID** (not web's Measurement ID).
- **No ad/marketing SDKs.** No `admob`, `react-native-google-mobile-ads`, `gms-ads`, `appsflyer`, `facebook`, `segment`, `mixpanel`, or `amplitude` in `package.json`.
- **No ad/marketing signals.** The event catalog is first-party product/UX telemetry only — `page_view`, `login`, `sign_up`, `search`, `view_search_results`, `filter_change`, `product_view`, `product_create_started`, `product_create_completed`, `contact_seller_clicked`, `message_sent`, `form_submit_failed`, `exception` (across `src/lib/analytics/*Events.ts`, `track.ts`, `GA4RouteListener.tsx`). No `ad_storage` / `ad_user_data` / `ad_personalization` axes exist on mobile.
- **No cross-app tracking.** With ATT/IDFA removed there is no device advertising identifier collected and no cross-app linkage. `user_id` is set only to the app's own backend user id on auth-resolve and cleared on sign-out (`AnalyticsInit.tsx:36-43`, `analyticsClient.ts:72-79`).

### Statement 5 — first-run consent prompt + settings toggle to change it later — **CONFIRMED (both exist)**

- **First-run prompt.** `src/components/init/ConsentPrompt.tsx` — a one-time `Modal` overlay, mounted **ready-gated** in `app/_layout.tsx:125` (`bootStatus === 'ready' && <ConsentPrompt />`). It shows only when `hydrated && decision === null` (`shouldShowConsentPrompt`), offers **Allow** and **Decline** as equal first-class choices (each records a real `decidedAt`, so it never re-shows), and links to the privacy policy (`router.push('/privacy')`). The store is hydrated by `ConsentInit` (`src/components/init/ConsentInit.tsx`), mounted **unconditionally** at `app/_layout.tsx:86` so the decision is available regardless of boot phase. If the user backgrounds without choosing, `decision` stays `null` and the prompt re-shows next `ready` boot.
- **Settings toggle.** `app/owner/user.tsx:315-341` — a standalone "Allow analytics" `Switch`. Reads live from `useConsentStore` (`value={consentDecision?.analytics === true}`), **immediate write-through** on flip (no Save button), `disabled` until the store has `hydrated` (no flicker from default-off to stored value). Labels come from the COOKIES namespace (`mobile.consent.settings.label` / `.description`).
  - *Minor path note:* the spec says `app/owner/dashboard/user.tsx`; the actual file is `app/owner/user.tsx` (the navigation-foundation work moved it). Same screen.

---

## Files touched

- None (read-only audit).

## Tests

- Not run (no code changes). Existing coverage observed: `analyticsGate.test.ts`, `consentStorage.test.ts`, `ConsentPrompt.test.ts`, `track.test.ts`, and per-event `*.test.ts`.

## Cleanup performed

- None needed (read-only).

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** change needed (drafted in "For Mastermind") — the 2026-06-02 GA4-mobile entry's "consent+ATT-gated init" no longer matches the shipped code (ATT removed); and its "auto-`screen_view` disabled in `firebase.json`" claim does not hold on the `dev` branch (no `firebase.json` present). Docs/QA to apply; I do not edit it.
- **state.md:** optional note (drafted in "For Mastermind") — the `Consent Mode — Mobile` and `Google Analytics v1` entries describe an ATT-gated analytics init that no longer reflects the code.
- **issues.md:** no change required by me (the native auto-collection gap is offered as a candidate entry in "For Mastermind" for Mastermind to triage).

## Obsoleted by this session

- Nothing deleted (read-only). Identified as *made stale by the shipped code* (not deletable from this repo / not in scope):
  - The "iOS ATT check" references in `src/lib/consent/analyticsGate.ts:9` JSDoc and `AnalyticsInit.tsx`/`analyticsClient.ts` comments are partially stale — they describe ATT as a live gate while also stating ATT was removed. Flagged (Part 4b), not fixed (out of read-only scope).
  - The `decisions.md`/spec ATT-gating language (other repo / config file) — drafted correction in "For Mastermind."

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, nothing added.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no key changes; observed the seven `mobile.consent.*` COOKIES keys are referenced as expected).
- Other parts touched: Part 11 (trust boundary) — confirmed: the consent decision is device-local, read only client-side to gate the client's own analytics init; never sent to the server, never used in any authorization/moderation decision.

## Known gaps / TODOs

- None introduced. The native auto-collection caveat (Statement 2) is a pre-existing observation, surfaced for the policy's accuracy, not a TODO I created.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Adjacent finding #1 (Part 4b) — native analytics auto-collection not pinned off.** `src/lib/analytics/analyticsClient.ts` — severity **medium** (privacy-relevant). The default-deny path only `return`s; it never calls `setAnalyticsCollectionEnabled(false)`, and there is no `firebase.json` in this branch (never tracked; not on disk) to disable native auto-collection by default. So the "nothing collected until opt-in" guarantee rests on the app never initializing the SDK, not on an explicit native off-switch — at the native layer the Firebase Analytics default is collection-enabled. The on-device DebugView smoke recorded in `decisions.md` 2026-06-02 was on `new-expo-dev` and referenced a `firebase.json` that is absent here. **I did not fix this — out of read-only scope, and the fix touches native config (`app.json`/`firebase.json`), which a hard rule forbids me to edit without instruction.** Candidate `issues.md` entry if the policy wants the stronger guarantee. For the policy as worded on facts I can confirm: analytics is off by default and the app's own event dispatch + SDK init are consent-gated; the residual is native baseline auto-collection at cold start before opt-in.

- **Adjacent finding #2 (Part 4b) — stale ATT comments in shipped code.** `src/lib/consent/analyticsGate.ts:9` (and JSDoc in `AnalyticsInit.tsx` / `analyticsClient.ts`) still describe a "chat F iOS ATT check" as the consuming contract, while the same files also state ATT was removed. Severity **low** (could mislead a future reader into re-adding ATT). I did not fix this — out of read-only scope.

- **Branch note.** Brief says `new-expo-dev`; repo is on `dev`, where the analytics + ATT-removal commits actually live (`ddaab53`…`bfbcbce`). The audited code is the real, finished code. Worth confirming the Privacy Policy is written against `dev`'s state.

- **Drafted config-file correction (decisions.md — 2026-06-02 "GA4 mobile v1 shipped" entry).** For Docs/QA to apply:
  > Correction (2026-06-06 audit): the shipped code on `dev` removed App Tracking Transparency — `expo-tracking-transparency` was deleted ("Remove unused deps", `d348b4b`) and the `requestTrackingPermissions` call was removed from `AnalyticsInit.tsx` (`963893f`). Mobile analytics is first-party only (no IDFA/ads); init is gated on `isAnalyticsConsentGranted()` **alone**, not on ATT. The earlier "consent+ATT-gated init" description is superseded. Separately, no `firebase.json` exists on `dev`; the "auto-`screen_view` disabled in `firebase.json`" claim is not verifiable on this branch.

- **Bottom line for the Privacy Policy:** statements 1, 3, 4, 5 are confirmed as written. Statement 2 is confirmed at the application layer (consent gates SDK init and every event dispatch; off by default); the only nuance to phrase carefully is the native baseline auto-collection at cold start (adjacent finding #1).
