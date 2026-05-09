# Environments

Three environments: **Dev** (Igor's local machine), **Stage** (shared
testing environment), **Prod** (live customer-facing system). Each
section below enumerates the components and where they run. Sections
are populated during Phase 1 of [`../master-plan.md`](../master-plan.md).

---

## Dev (local development)

Local dev uses the oglasino-stage-49abb Firebase project; backend runs
locally against laptop's local Postgres + Redis + ES (NOT against stage
droplet). Using the stage Firebase project from local dev is intentional
for v1 — see [`../firebase/projects.md`](../firebase/projects.md) for
the "if/when we need a separate dev project" trigger conditions.

### Backend
TBD — populated during Phase 1.

### Database
TBD — populated during Phase 1.

### Cache (Redis)
TBD — populated during Phase 1.

### Search (Elasticsearch)
TBD — populated during Phase 1.

### Frontend (Web)
TBD — populated during Phase 1.

### Frontend (Mobile)
TBD — populated during Phase 1.

### CDN/Workers
TBD — populated during Phase 1.

### Firebase
Uses oglasino-stage-49abb (intentional for v1 — see
[`../firebase/projects.md`](../firebase/projects.md) for the if/when
we need a separate dev project trigger conditions).

### Domains
TBD — populated during Phase 1.

---

## Stage

| Component | Detail |
|---|---|
| Backend | Spring Boot in Docker on oglasino-stage droplet (image not yet published — Phase 3C) |
| Database | Postgres 16 in Docker on oglasino-stage droplet |
| Cache | Redis 7 in Docker on oglasino-stage droplet |
| Search | Elasticsearch 8.13.4 in Docker on oglasino-stage droplet (single-node, security disabled) |
| Frontend (Web) | Vercel — `oglasino-web-stage` env (Phase 1E pending) |
| Frontend (Mobile) | EAS profile `preview` for Internal Testing / TestFlight (Phase 3E pending) |
| CDN/Workers | Cloudflare oglasino-images-stage + oglasino-router-stage (Phase 1C pending) |
| Firebase | oglasino-stage-49abb |
| Domains | api-stage.oglasino.com, stage.oglasino.com, cdn-stage.oglasino.com, api-origin-stage.oglasino.com (DNS in Phase 1C) |

---

## Prod

### Backend
TBD — populated during Phase 1.

### Database
TBD — populated during Phase 1.

### Cache (Redis)
TBD — populated during Phase 1.

### Search (Elasticsearch)
TBD — populated during Phase 1.

### Frontend (Web)
TBD — populated during Phase 1.

### Frontend (Mobile)
TBD — populated during Phase 1.

### CDN/Workers
TBD — populated during Phase 1C.

### Firebase
Firebase project: oglasino-prod-7e5db. See
[`../firebase/projects.md`](../firebase/projects.md).

### Domains
TBD — populated during Phase 1.
