# Vercel Deployments

## Projects

| Project | Repo | Production Branch | URL | Notes |
|---|---|---|---|---|
| oglasino-web | memento-tech/oglasino-web | main (workflow-driven, vercel.json disables auto-deploy) | https://oglasino-web.vercel.app | prod web |
| oglasino-web-stage | memento-tech/oglasino-web | stage (workflow-driven, vercel.json disables auto-deploy) | https://oglasino-web-stage.vercel.app | stage web |

## Deployment model

Both projects share a `vercel.json` in the `oglasino-web` repo:

```json
{
  "git": {
    "deploymentEnabled": {
      "main": false,
      "stage": false
    }
  }
}
```

Vercel does NOT auto-deploy. All deploys are driven by GH Actions
workflows:

- **Prod:** `.github/workflows/deploy-prod.yml` triggers on push to
  `main`. Maintenance gating is manual (deploy turns ON, must be
  manually flipped OFF after verification).
- **Stage:** `.github/workflows/deploy-stage.yml` triggers on push to
  `stage`. Maintenance gating is auto-restore (deploy turns ON, then
  automatically OFF on success).

Both workflows:
1. Run pre-checks (lint, typecheck)
2. Flip Cloudflare KV maintenance flag ON
3. Pull Vercel env (production scope) for the project
4. Build with Vercel CLI
5. Deploy to Vercel via `vercel deploy --prebuilt --prod`
6. (Stage only) Flip Cloudflare KV maintenance flag OFF on success

## Why workflow-driven instead of Vercel auto-deploy

- Maintenance gating during deploys (zero apparent downtime via Worker
  503 page)
- Pre-deploy lint + typecheck gate (Vercel auto-deploy doesn't run
  these as gates — they're just info)
- Single source of truth for deploy logic (the workflow file in git)
- Easier rollback strategy (re-run a previous workflow run on a
  previous commit)

## Routing

Vercel does NOT have a custom domain configured for either project.
The Vercel projects only know themselves as `*.vercel.app` URLs.

Public traffic flow:
```
user → oglasino.com / stage.oglasino.com
  → Cloudflare DNS (proxied)
  → oglasino-router-prod / oglasino-router-stage Worker
  → fetch(FRONTEND_ORIGIN = the *.vercel.app URL)
  → Vercel serves the build
```

Vercel sees all requests as hitting the `.vercel.app` URL (because
that's how the Worker forwards). The Cloudflare-side hostname is
preserved for the user but invisible to Vercel.

## Environment variables

Each project has its own env vars scoped to the "Production"
environment in Vercel. Names match between projects (e.g., both have
`NEXT_PUBLIC_FIREBASE_PROJECT_ID`); values differ:

- prod project: `NEXT_PUBLIC_FIREBASE_PROJECT_ID = oglasino-prod-7e5db`
- stage project: `NEXT_PUBLIC_FIREBASE_PROJECT_ID = oglasino-stage-49abb`

For full env var inventory, see `overview/secret-inventory.md`.

## Required GH Secrets (oglasino-web repo)

Shared between prod and stage workflows:
- `VERCEL_TOKEN` — user token
- `VERCEL_ORG_ID` — Vercel team/org ID
- `CF_API_TOKEN` — Cloudflare API token
- `CF_ACCOUNT_ID` — Cloudflare account ID

Per-environment:
- `VERCEL_PROJECT_ID` (prod), `VERCEL_PROJECT_ID_STAGE`
- `CF_KV_NAMESPACE_ID` (prod), `CF_KV_NAMESPACE_ID_STAGE`
