# Cloudflare Workers

| Worker | Repo | Purpose | Custom Domain | R2 Binding | Branch → Env |
|---|---|---|---|---|---|
| oglasino-router-prod | TBD | Router for prod traffic | oglasino.com | none | main → prod |
| oglasino-router-stage | TBD | Router for stage traffic | stage.oglasino.com | none | stage → stage |
| oglasino-images-prod | oglasino-image-worker | Image upload + serve | cdn.oglasino.com | oglasino-images-prod | main → prod |
| oglasino-images-stage | oglasino-image-worker | Image upload + serve | cdn-stage.oglasino.com | oglasino-images-stage | stage → stage |
| oglasino-maintenance | oglasino-maintenance | Maintenance page | (toggled on demand) | none | main only |

## Rename plan

The existing prod router is currently named `oglasino-prod-router`. It
will be renamed to `oglasino-router-prod` during Phase 1C.1 of
[`../master-plan.md`](../master-plan.md), to match the
`<service>-<env>` convention used everywhere else (images, web, etc.).
A new `oglasino-router-stage` Worker is created at the same time.
