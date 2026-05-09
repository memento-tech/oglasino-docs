# GitHub Actions Workflows

Populated during Phase 3 of
[`../master-plan.md`](../master-plan.md).

## oglasino-backend

| Workflow file | Trigger | Purpose |
|---|---|---|
| `stage-deploy.yml` | push to `stage` | Build image, push, SSH-deploy to stage droplet |
| `prod-deploy.yml` | push to `main` | Build image, push, SSH-deploy to prod droplet |

Phase 3C details.

## oglasino-web

| Workflow file | Trigger | Purpose |
|---|---|---|
| (Vercel handles deploys natively, OR `stage-deploy.yml` / `prod-deploy.yml` if explicit control needed) | | |

Phase 3D — decision recorded once made.

## oglasino-expo

| Workflow file | Trigger | Purpose |
|---|---|---|
| `eas-build-stage.yml` | push to `stage` | Trigger `eas build --profile preview` |
| `eas-build-prod.yml` | push to `main` | Trigger `eas build --profile production` |

Phase 3E.4 details.

## oglasino-image-worker, oglasino-firestore-rules, oglasino-maintenance

Each repo has a single deploy workflow per branch (stage / main or
stage / prod). Documented in those repos; cross-link here when
finalized.
