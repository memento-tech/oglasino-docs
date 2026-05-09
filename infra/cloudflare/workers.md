# Cloudflare Workers

| Worker | Repo | Branch → Env | Custom Domains | KV Bindings | R2 Bindings |
|---|---|---|---|---|---|
| oglasino-router-prod | oglasino-router | main → production | oglasino.com, www.oglasino.com, api.oglasino.com | CONFIG (prod) | none |
| oglasino-router-stage | oglasino-router | stage → stage | api-stage.oglasino.com, stage.oglasino.com (route claimed but no DNS yet) | CONFIG (stage) | none |
| oglasino-images-prod | oglasino-image-worker | main → production | cdn.oglasino.com | none | oglasino-images-prod |
| oglasino-images-stage | oglasino-image-worker | stage → stage | cdn-stage.oglasino.com | none | oglasino-images-stage |
| oglasino-maintenance | oglasino-maintenance | main only | (toggled on demand via KV flag) | none | none |

## Migration history

`oglasino-prod-router` (dashboard-edited Worker) was migrated to
`oglasino-router-prod` (deployed from oglasino-router repo) on
2026-05-09. Routes were moved one at a time. Old Worker deleted after
verification.

## Naming convention

`<purpose>-<env>` everywhere. Env values are `stage` or `prod`. The
Cloudflare dashboard env name is `production` (full word) but human
references use `prod`.
