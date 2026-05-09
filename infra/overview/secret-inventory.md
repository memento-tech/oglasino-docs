# Secret Inventory

> This file is populated in Phase 2.1. Every entry must include the
> rotation date — even `[never rotated]` is a valid value to start with.

Document **secret names only** — never paste secret values into this
repo. Values live in GH Secrets, Vercel env, Firebase, or wherever the
secret is consumed.

## Storage philosophy

Secret VALUES never live in this repo. Only NAMES, owners, and
metadata (rotation date, where the value lives) are tracked here.

Actual values live in:
- **GH Actions Secrets** — for CI/CD pipeline access
- **Vercel Environment Variables** — for Next.js runtime
- **EAS Secrets** — for Expo build profiles
- **Igor's password manager / encrypted local storage** — for files
  that need to be hand-installed (e.g., service account JSONs during
  initial droplet provisioning)
- **Droplet env files** (`/etc/oglasino/.env`) — for backend runtime,
  set at provisioning time, never committed

Whenever a secret's location moves OR its value rotates, update the
"Lives In" column and "Rotation Date" column simultaneously.

## Inventory

| Secret Name | Lives In | Used By | Rotation Date | Notes |
|---|---|---|---|---|
| FIREBASE_TOKEN | GH Secrets (oglasino-firestore-rules) | rules deploy workflow | TBD | CI token from `firebase login:ci` |
| CLOUDFLARE_API_TOKEN | GH Secrets (oglasino-image-worker) | worker deploy | TBD | scoped to Workers + R2 + Routes |
| JWT_SIGNING_SECRET_STAGE | GH Secrets (oglasino-image-worker) | worker runtime | TBD | byte-identical to backend's value |
| JWT_SIGNING_SECRET_PROD | GH Secrets (oglasino-image-worker) | worker runtime | TBD | byte-identical to backend's value |
| BACKEND_SHARED_SECRET_STAGE | GH Secrets (oglasino-image-worker) | worker → backend admin calls | TBD | |
| BACKEND_SHARED_SECRET_PROD | GH Secrets (oglasino-image-worker) | worker → backend admin calls | TBD | |
| FIREBASE_STAGE_ADMIN_SDK | will go into GH Secrets (oglasino-backend) + local dev (Igor's laptop ~/Secrets/) | Spring backend Firebase Admin SDK init | 2026-05-09 | Service account JSON for oglasino-stage-49abb. Rotate every 90 days. |
| FIREBASE_PROD_ADMIN_SDK | will go into GH Secrets (oglasino-backend) | Spring backend Firebase Admin SDK init | 2026-05-09 | Service account JSON for oglasino-prod-7e5db. Rotate every 90 days. NEVER store on a developer laptop. |
| NEXT_PUBLIC_FIREBASE_API_KEY (stage) | will go into Vercel env (stage) + GH Secrets (oglasino-web stage workflow) | Next.js Firebase client init | 2026-05-09 | From oglasino-stage-49abb web app config blob |
| NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN (stage) | Vercel env (stage) | Next.js Firebase client init | 2026-05-09 | oglasino-stage-49abb.firebaseapp.com |
| NEXT_PUBLIC_FIREBASE_PROJECT_ID (stage) | Vercel env (stage) | Next.js Firebase client init | 2026-05-09 | oglasino-stage-49abb |
| NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET (stage) | Vercel env (stage) | Next.js Firebase client init | 2026-05-09 | oglasino-stage-49abb.appspot.com |
| NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID (stage) | Vercel env (stage) | Next.js Firebase client init | 2026-05-09 | from web app config blob |
| NEXT_PUBLIC_FIREBASE_APP_ID (stage) | Vercel env (stage) | Next.js Firebase client init | 2026-05-09 | from web app config blob |
| NEXT_PUBLIC_FIREBASE_API_KEY (prod) | Vercel env (prod) | Next.js Firebase client init | 2026-05-09 | From oglasino-prod-7e5db web app config blob |
| NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN (prod) | Vercel env (prod) | Next.js Firebase client init | 2026-05-09 | oglasino-prod-7e5db.firebaseapp.com |
| NEXT_PUBLIC_FIREBASE_PROJECT_ID (prod) | Vercel env (prod) | Next.js Firebase client init | 2026-05-09 | oglasino-prod-7e5db |
| NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET (prod) | Vercel env (prod) | Next.js Firebase client init | 2026-05-09 | oglasino-prod-7e5db.appspot.com |
| NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID (prod) | Vercel env (prod) | Next.js Firebase client init | 2026-05-09 | from web app config blob |
| NEXT_PUBLIC_FIREBASE_APP_ID (prod) | Vercel env (prod) | Next.js Firebase client init | 2026-05-09 | from web app config blob |
| GoogleService-Info.plist (stage) | EAS Secrets (Phase 3E.2 wiring TBD) | Expo iOS Firebase init | 2026-05-09 | downloaded from oglasino-stage-49abb iOS app registration |
| GoogleService-Info.plist (prod) | EAS Secrets | Expo iOS Firebase init | 2026-05-09 | downloaded from oglasino-prod-7e5db iOS app registration |
| google-services.json (stage) | EAS Secrets | Expo Android Firebase init | 2026-05-09 | downloaded from oglasino-stage-49abb Android app registration |
| google-services.json (prod) | EAS Secrets | Expo Android Firebase init | 2026-05-09 | downloaded from oglasino-prod-7e5db Android app registration |
| NEXT_PUBLIC_FIREBASE_VAPID_KEY (stage) | Vercel env (stage), Igor's password manager | Web push registration | 2026-05-09 | Public VAPID key for oglasino-stage-49abb. Public exposure is by design. |
| NEXT_PUBLIC_FIREBASE_VAPID_KEY (prod) | Vercel env (prod), Igor's password manager | Web push registration | 2026-05-09 | Public VAPID key for oglasino-prod-7e5db. Public exposure is by design. |
| APNs Authentication Key (.p8) | Igor's password manager only | Uploaded to Firebase Console (both projects, both slots) | 2026-05-09 | Single key for entire Apple Team ID. Sandbox & Production. NEVER store in any repo, env var, or build artifact. |
| APNs Key ID | Igor's password manager | Reference only — needed for Firebase upload form | 2026-05-09 | 10 alphanumeric chars. Not secret per se but part of the credential context. |
| Apple Team ID | Igor's password manager | Reference only — needed for Firebase upload form, EAS configuration | n/a | 10 alphanumeric chars. Not secret. Used by EAS for code signing in Phase 3E. |
| (more rows added in Phase 2.1) | | | | |
