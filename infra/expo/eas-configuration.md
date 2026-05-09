# EAS Configuration

Populated during Phase 3E of
[`../master-plan.md`](../master-plan.md).

## Build profiles

The `eas.json` will define three profiles:

| Profile | Used for | Distribution | App name (via `app.config.ts`) |
|---|---|---|---|
| development | Local dev (Igor's device) | internal (dev client) | `Oglasino (Dev)` |
| preview | Stage testing | internal (TestFlight Internal / Play Internal Testing) | `Oglasino (Stage)` |
| production | Prod release | store | `Oglasino` |

All three share `applicationId: com.oglasino`. Differentiation is
purely cosmetic (display name) and via env vars.

## Environment variables per profile

| Var | development | preview | production |
|---|---|---|---|
| EXPO_PUBLIC_API_URL | TBD | TBD | TBD |
| EXPO_PUBLIC_FIREBASE_PROJECT_ID | oglasino-stage | oglasino-stage | oglasino-prod |
| (more — populate Phase 3E.3) | | | |

## CI integration

GH Actions on `oglasino-expo`:

- Push to `stage` → trigger `eas build --profile preview` for both platforms
- Push to `main` → trigger `eas build --profile production` for both platforms
- Push to `dev` → no auto-build (manual `eas build` from local)

## Push notifications

- iOS APNs key (.p8) registered in Apple Developer, uploaded to
  Firebase per project (see
  [`../firebase/messaging.md`](../firebase/messaging.md))
- Android FCM credentials per project, downloaded as
  `google-services.json`, injected per profile via EAS Secrets
