# Secret Inventory

> This file is populated in Phase 2.1. Every entry must include the
> rotation date — even `[never rotated]` is a valid value to start with.

Document **secret names only** — never paste secret values into this
repo. Values live in GH Secrets, Vercel env, Firebase, or wherever the
secret is consumed.

| Secret Name | Lives In | Used By | Rotation Date | Notes |
|---|---|---|---|---|
| FIREBASE_TOKEN | GH Secrets (oglasino-firestore-rules) | rules deploy workflow | TBD | CI token from `firebase login:ci` |
| CLOUDFLARE_API_TOKEN | GH Secrets (oglasino-image-worker) | worker deploy | TBD | scoped to Workers + R2 + Routes |
| JWT_SIGNING_SECRET_STAGE | GH Secrets (oglasino-image-worker) | worker runtime | TBD | byte-identical to backend's value |
| JWT_SIGNING_SECRET_PROD | GH Secrets (oglasino-image-worker) | worker runtime | TBD | byte-identical to backend's value |
| BACKEND_SHARED_SECRET_STAGE | GH Secrets (oglasino-image-worker) | worker → backend admin calls | TBD | |
| BACKEND_SHARED_SECRET_PROD | GH Secrets (oglasino-image-worker) | worker → backend admin calls | TBD | |
| (more rows added in Phase 2.1) | | | | |
