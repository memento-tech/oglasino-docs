# Vercel Deployments

## Project

- **Vercel project:** `oglasino-web` (TBD — confirm slug during Phase 1E)
- **Connected repo:** `oglasino-web` on GitHub
- **Framework:** Next.js (TBD — confirm)

## Branch → environment mapping

| Branch | Vercel environment | Domain |
|---|---|---|
| `main` | Production | oglasino.com, www.oglasino.com |
| `stage` | Stage (custom env) | stage.oglasino.com |
| (preview branches) | Preview | auto-generated `*.vercel.app` |

Stage environment is added during Phase 1E.1.

## Environment variables

Env vars per environment (production / stage / preview / development)
are populated during Phase 1E.2 and Phase 2.9. Names recorded in
[`../overview/secret-inventory.md`](../overview/secret-inventory.md);
values live in Vercel's dashboard.

| Env var name | Production | Stage | Notes |
|---|---|---|---|
| (TBD — populate Phase 2) | | | |
