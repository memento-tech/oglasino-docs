# Firebase Cloud Messaging (FCM)

## Per-project configuration

Each Firebase project (`oglasino-stage`, `oglasino-prod`) has its own
FCM configuration. Records below are populated during Phase 1A.5 and
Phase 3E.5 of [`../master-plan.md`](../master-plan.md).

| Field | oglasino-stage | oglasino-prod |
|---|---|---|
| Sender ID | TBD | TBD |
| Web Push (VAPID) public key | TBD | TBD |
| Web Push (VAPID) private key | stored in GH Secrets (name TBD) | stored in GH Secrets (name TBD) |
| iOS APNs auth key (.p8) | stored in EAS / GH Secrets | stored in EAS / GH Secrets |

## Web

The service worker file lives at `oglasino-web/public/firebase-messaging-sw.js`
(or the equivalent path in the Next.js app). It is registered on first
permission grant. Specifics documented in
[`../vercel/deployments.md`](../vercel/deployments.md) and the web
repo's README.

## Mobile (Expo / React Native)

iOS push uses an APNs auth key registered in Apple Developer + uploaded
to Firebase. Android push uses FCM directly. Both are handled by
`expo-notifications`. EAS build profile picks up the right Firebase
config per environment — see
[`../expo/eas-configuration.md`](../expo/eas-configuration.md).
