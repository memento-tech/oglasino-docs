# Firebase Projects

## Current state (pre-cutover)

- **Oglasino Dev** — used by both dev and prod (incorrectly). Decommissioned in Phase 4.11.
- **Oglasino Prod** — exists but not wired correctly. Decommissioned in Phase 4.12.

## Target state (post-cutover)

### oglasino-stage
- **Project ID:** oglasino-stage-49abb
- **Used by:** stage backend, stage web, stage RN, **and local development**
- **Auth providers:** email, Google (both enabled)
- **Authorized domains:** `localhost`, `oglasino-stage-49abb.firebaseapp.com`, `oglasino-stage-49abb.web.app`, `stage.oglasino.com`
- **Created:** 2026-05-09
- **Owner:** Oglasino (oglasino@gmail.com)

### oglasino-prod
- **Project ID:** oglasino-prod-7e5db
- **Used by:** prod backend, prod web, prod RN
- **Auth providers:** email, Google (both enabled)
- **Authorized domains:** `localhost`, `oglasino-prod-7e5db.firebaseapp.com`, `oglasino-prod-7e5db.web.app`, `oglasino.com`, `www.oglasino.com`
- **Created:** 2026-05-09
- **Owner:** Oglasino (oglasino@gmail.com)

## Local development uses stage Firebase

Decision recorded: local dev points at the stage Firebase project for v1.
This is acceptable risk while there's only one developer (Igor). The trigger
for spinning up a separate `oglasino-dev` Firebase project is:

- A second developer joins, OR
- A QA tester joins (their stage data must not be polluted by dev work), OR
- Stage starts hosting real-world testing scenarios that local dev shouldn't
  disrupt.

When any of those triggers fire, create a third Firebase project, copy the
local-dev configuration over, point `eas.json development` profile at it.

## Registered apps per project

Each project has three app registrations (web, iOS, Android). The
config blobs / files are NOT stored in this repo — they live in:
- Web config blobs → secret-inventory.md tracks them, values in GH
  Secrets (Phase 2 wiring) and Vercel env vars (Phase 1E.2)
- iOS GoogleService-Info.plist → EAS Secrets (Phase 3E.2)
- Android google-services.json → EAS Secrets (Phase 3E.2)

| Platform | Stage app nickname | Prod app nickname | Bundle/Package ID |
|---|---|---|---|
| Web | oglasino-web-stage | oglasino-web-prod | n/a |
| iOS | oglasino-ios-stage | oglasino-ios-prod | com.oglasino |
| Android | oglasino-android-stage | oglasino-android-prod | com.oglasino |

**Single applicationId note:** iOS and Android use `com.oglasino` for
both stage and prod. A device can only have one Firebase config
installed per platform at a time — installing the prod build over a
stage build overwrites the Firebase config (FCM re-registers, etc.).
This is acceptable for v1; if side-by-side install is ever needed,
switch to applicationId-per-env.

## Future ownership transfer

Both projects are owned by oglasino@gmail.com (a dedicated Google
account, not Igor's personal). When Memento Tech (or equivalent legal
entity) is formally registered:

1. The Google account itself can be transferred to a Google Workspace
   under the entity (preferred), OR
2. A new Google account under the entity can be made co-owner, then
   the original account demoted/removed.

Google Cloud project ownership transfer involves billing account
re-association — plan ahead.
