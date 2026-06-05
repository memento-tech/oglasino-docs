# Session — Bug batch: cold-start push-notification AsyncStorage null crash + base-site-switch loading feedback

**Repo:** oglasino-expo
**Branch:** new-expo-dev (no switch, no commit, no push)
**Date:** 2026-06-03
**Slug:** bug-batch (order 2)
**Type:** bug fix — two unrelated issues reported directly by Igor (no brief; `.agent/brief.md` empty).

---

## Task (one sentence)

Fix the cold-start crash `[AsyncStorage] Passing null/undefined as value` thrown from `PushNotificationsInit.tsx:123`, and add immediate on-tap loading feedback when switching base site from `PortalConfigDialog` (currently 2-3s of dead time before anything visible happens).

---

## Outcome summary

| Issue | Symptom | Fix |
| --- | --- | --- |
| 1 — AsyncStorage null crash | Cold start throws because `getLastNotificationResponse()` returns a response whose `request.identifier` is `null`; `setItem(LAST_HANDLED_RESPONSE_KEY, null)` is rejected | Dedup on a non-null token: `request.identifier ?? `date:${notification.date}``. Crash gone; stale boot responses still dedup. |
| 2 — base-site switch UX | Tap → 2-3s with no feedback → base site visibly changes behind the still-open dialog → dialog closes only when the whole freshness gate finishes | Per-row `ActivityIndicator` shown immediately on tap via local `switchingCode` state; other rows disabled during the switch. |

Files changed:
- `src/notifications/components/PushNotificationsInit.tsx`
- `src/components/dialog/dialogs/PortalConfigDialog.tsx`

---

## Issue 1 — push-notification cold-start crash

`handleResponse` (PushNotificationsInit.tsx) is called from the boot path with the value of `Notifications.getLastNotificationResponse()`, which persists across launches. For the reported response, `notification.request.identifier` is `null` (a remote notification response can carry a null identifier). The code then did `await AsyncStorage.setItem(LAST_HANDLED_RESPONSE_KEY, identifier)` — AsyncStorage rejects null, hence the uncaught-promise crash on every cold start while that response is the last one.

A bare guard around the `setItem` is **not** sufficient: if we skip the write but still navigate, every cold start re-fires navigation off that stale boot response — the exact bug the dedup block (LHNR) was added to prevent. So the fix introduces a **non-null dedup token** that falls back to the notification's delivery timestamp when the identifier is absent:

```ts
const dedupKey = notification.request.identifier ?? `date:${notification.date}`;
```

`Notification.date: number` is confirmed present in the installed `expo-notifications` types (`node_modules/expo-notifications/build/Notifications.types.d.ts:583-586`) and is stable across re-reads of `getLastNotificationResponse()`, so:
- the crash is gone (never null passed to AsyncStorage), and
- a stale null-identifier boot response still dedups (navigates at most once), preserving the original LHNR intent.

Both the `getItem` comparison and the `setItem` write now use `dedupKey`.

---

## Issue 2 — base-site switch loading feedback (PortalConfigDialog)

Root cause of the dead time: the row's `onPress` did `await setBaseSiteForCode(code)` (= `bootStore.pickBaseSite`), which first awaits a network `fetchBaseSiteByCode` **before** any visible state change (the 2-3s), then sets `selectedBaseSite` (the visible background change), then runs the freshness gate (`ready → updating → ready`). The dialog only closes after the whole chain resolves.

Why the existing global feedback wasn't visible: `DialogManager` renders inside a React Native `<Modal>` (its own native window above the root view). The `'updating'` `LoadingOverlay` in `app/_layout.tsx` is part of the root tree, so it renders **behind** the modal. The dialog therefore has to surface its own loading state.

Fix (all in `PortalConfigDialog.tsx`):
- New local `const [switchingCode, setSwitchingCode] = useState<string | null>(null)`.
- `onPress` now early-returns if a switch is in flight or the row is already selected, then `setSwitchingCode(baseSite.code)` **before** the await — React flushes the re-render at the first `await`, so the spinner appears immediately.
- The tapped row renders an `<ActivityIndicator>` in place of its flag `<Image>` while `switchingCode === baseSite.code`; all base-site rows are `disabled` while a switch is in flight (prevents double-tap / racing two switches).
- No manual reset of `switchingCode` is needed: `openDialog` is single-slot (replaces the current dialog), so every outcome (INFO dialog, `onClose`, or a `toMaintenance` divert inside `pickBaseSite`) unmounts this component.

Behaviour preserved: the logged-in/different-domain caution (`INFO_DIALOG`) and the same-domain `onClose()` branches are unchanged.

---

## Verification

- `npx tsc --noEmit` — **clean (exit 0).**
- `npm run lint` — **0 errors, 100 warnings.** All warnings pre-existing; my two files add no new lint findings (the `router`/`segments` unused-var warnings in PortalConfigDialog and the hook-deps warnings in PushNotificationsInit predate this session and sit on lines I did not touch). The 100-warning total is the accumulated `new-expo-dev` baseline, not a regression from this session.
- `npm test` (vitest run) — **38 files passed, 404 tests passed.** Suite green. No tests cover the two touched files (none exist).

---

## Cleanup performed

None needed — both changes are net-additive to existing functions; no dead code, commented-out code, debug logging, or unused imports introduced. `ActivityIndicator` and `useState` were the only new symbols, both used.

---

## Obsoleted by this session

Nothing.

---

## Conventions check

- **Part 4 (cleanliness):** no `console.log`/debug logging, no TODO/FIXME, no commented-out code, no unused imports/vars introduced. tsc 0 errors, lint 0 errors, 404 tests green.
- **Part 4a (simplicity):** chose the minimal correct fix in each case. For Issue 1, resisted a bare null-guard (which would re-open the stale-navigation bug) in favour of a one-line non-null dedup token. For Issue 2, used a single local `switchingCode` string rather than adding store state or restructuring `pickBaseSite` — the loading concern is dialog-local UX, not boot-machine state.
- **Part 4b (adjacent observations):** `PortalConfigDialog.tsx:45-46` has two pre-existing unused vars (`router` via `useRouter()`, `segments` via `useSegments()`) flagged by lint. Not introduced or used by this session; left untouched to avoid scope creep — noted here for a future cleanup pass.
- **Stack reminders / wire contract:** Issue 1 relies only on `expo-notifications`' `Notification.date` (verified in installed types), no backend contract involved. Issue 2 reuses the existing `bootStore.pickBaseSite` flow unchanged — no new endpoints, no contract change.

---

## Config-file impact

No edit to any of the four config files by me. Closure-gate check: this session adopts/retires no `state.md` Expo-backlog row and touches no named feature. If Igor wants these two fixes tracked, they belong in `issues.md` (Docs/QA's file) as bug entries:
- cold-start AsyncStorage null crash in `PushNotificationsInit` (null `request.identifier`) → fixed via `date` fallback dedup token.
- base-site switch had no on-tap loading feedback (global overlay hidden behind dialog Modal) → fixed with in-dialog per-row spinner.

I drafted these here rather than editing `issues.md`. No implicit config dependency remains unstated.

---

## For Mastermind

- Two direct bug fixes on `new-expo-dev`, no brief:
  - **Push cold-start crash:** `getLastNotificationResponse()` can return a response with a null `request.identifier`; persisting it to AsyncStorage threw. Fixed by deduping on `request.identifier ?? `date:${notification.date}`` so the write is never null and stale boot responses still dedup.
  - **Base-site switch UX:** `pickBaseSite` awaits a network call before any visible change and the global `'updating'` overlay renders behind the dialog's native Modal, so the tap felt unresponsive for 2-3s. Added an immediate per-row `ActivityIndicator` + disabled rows during the switch, scoped entirely to `PortalConfigDialog`.
- **Gates:** tsc clean, lint 0 errors / 100 warnings (pre-existing baseline), 404 tests green.
- **Adjacent:** two pre-existing unused vars (`router`, `segments`) in `PortalConfigDialog.tsx:45-46` — candidate for a future lint-cleanup pass; not touched here.
