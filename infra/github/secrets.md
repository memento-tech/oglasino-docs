# GitHub Actions Secrets

> Names only — never paste values. Cross-reference
> [`../overview/secret-inventory.md`](../overview/secret-inventory.md)
> for the canonical list with rotation dates.

Populated alongside Phase 2 of
[`../master-plan.md`](../master-plan.md).

## oglasino-backend

| Secret | Used by |
|---|---|
| (TBD — populate Phase 2.4) | |

## oglasino-web

| Secret | Used by |
|---|---|
| (TBD — populate Phase 2.5) | |

## oglasino-image-worker

| Secret | Used by |
|---|---|
| CLOUDFLARE_API_TOKEN | wrangler deploy |
| JWT_SIGNING_SECRET_STAGE | worker runtime |
| JWT_SIGNING_SECRET_PROD | worker runtime |
| BACKEND_SHARED_SECRET_STAGE | worker → backend admin calls |
| BACKEND_SHARED_SECRET_PROD | worker → backend admin calls |
| (more — populate Phase 2.6) | |

## oglasino-firestore-rules

| Secret | Used by |
|---|---|
| FIREBASE_TOKEN | firebase deploy |
| (more — populate Phase 2.7) | |

## oglasino-expo

| Secret | Used by |
|---|---|
| (TBD — populate Phase 2.8) | |
