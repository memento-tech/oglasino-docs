# Firebase Cloud Messaging (FCM)

## Web push (both projects)

Each project has a web push VAPID key generated:

- **oglasino-stage-49abb VAPID key:** [stored in password manager —
  Igor]; consumed as `NEXT_PUBLIC_FIREBASE_VAPID_KEY` in stage Vercel env
- **oglasino-prod-7e5db VAPID key:** [stored in password manager — Igor];
  consumed as `NEXT_PUBLIC_FIREBASE_VAPID_KEY` in prod Vercel env

Web push is delivered via the existing service worker at
`oglasino-web/public/firebase-messaging-sw.js` (template at
`oglasino-web/scripts/firebase-messaging-sw.template.js`, built into the
final `.js` by `oglasino-web/scripts/build-firebase-sw.mjs` which
substitutes the project's Firebase config at build time).

The legacy "Cloud Messaging API (Legacy)" / "Server Key" is **left
disabled** on both projects. The backend uses the V1 HTTP API via the
Firebase Admin SDK (authenticated with the service account JSON), so no
legacy server key is needed.

## iOS push — APNs Authentication Key

Single APNs Authentication Key generated in Apple Developer account,
uploaded to BOTH Firebase projects.

| Property | Value |
|---|---|
| Key Name | Oglasino APNs |
| Key ID | [stored in password manager — Igor]; format: 10 alphanumeric chars |
| Team ID | [stored in password manager — Igor]; format: 10 alphanumeric chars |
| APNs Environment | Sandbox & Production (both — the first generated key was Production-only and was revoked; replacement key supports both) |
| Generated | 2026-05-09 |

**The same `.p8` file is uploaded four times:**

- oglasino-stage-49abb → Apple app `oglasino-ios-stage` → development APNs slot
- oglasino-stage-49abb → Apple app `oglasino-ios-stage` → production APNs slot
- oglasino-prod-7e5db → Apple app `oglasino-ios-prod` → development APNs slot
- oglasino-prod-7e5db → Apple app `oglasino-ios-prod` → production APNs slot

This is correct, not redundant. APNs Authentication Keys are scoped per
Apple Team ID, not per app or environment. Firebase routes pushes to
whichever slot matches the device's registered token environment
(development for Xcode debug builds, production for TestFlight / App
Store builds).

## Android push — FCM via google-services.json

No additional Firebase Console step required. FCM works automatically
once the Android app is registered in 1A.4 and the
`google-services.json` is bundled with the app build (handled by EAS in
Phase 3E).

## Push token storage and lifecycle

- **Web:** FCM token obtained via `getToken(messaging, { vapidKey, ... })`
  in `oglasino-web/src/lib/services/pushTokenService.ts`. Token sent to
  backend, persisted per user.
- **iOS / Android:** Expo push token via `expo-notifications`, sent to
  backend.
- **Backend:** stores tokens in `user_push_tokens` table, uses them in
  `DefaultNotificationsService.sendAsyncNotification` to send pushes via
  the appropriate `PushService` implementation (FCM for web, Expo for
  mobile).

## Security notes

- VAPID keys (web): the PUBLIC half of the keypair is exposed to the
  browser via `NEXT_PUBLIC_FIREBASE_VAPID_KEY`. The private half stays
  inside Firebase. Public exposure is by design — VAPID's purpose is
  cryptographic identification of the push sender.
- APNs `.p8` file: NEVER commit anywhere, NEVER in any env var, NEVER
  in any build artifact. Lives only in Igor's password manager and
  uploaded directly to Firebase Console UI. Firebase stores it
  server-side and never exposes it back.
- Firebase Admin SDK service account JSONs (from 1A.4): same handling
  — password manager only, uploaded as opaque secret values to GH
  Secrets / Vercel envs.
