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

| Component | Detail | Status |
|---|---|---|
| Backend | Spring Boot in Docker on oglasino-stage droplet | Image not yet published — Phase 3C |
| Database | Postgres 16 in Docker on oglasino-stage droplet | Running |
| Cache | Redis 7 in Docker on oglasino-stage droplet | Running |
| Search | Elasticsearch 8.13.4 in Docker on oglasino-stage droplet | Running |
| Frontend (Web) | Vercel — stage env | Phase 1E pending |
| Frontend (Mobile) | EAS profile `preview` | Phase 3E pending |
| CDN | Cloudflare oglasino-images-stage | Running (since pre-Phase 1) |
| Router | Cloudflare oglasino-router-stage | Running (Phase 1C) |
| Firebase | oglasino-stage-49abb | Configured (Phase 1A) |
| Domain — frontend | stage.oglasino.com | DNS deferred (Phase 1E) |
| Domain — API | api-stage.oglasino.com | Configured (Phase 1C) — Worker reachable, awaiting backend |
| Domain — origin (Worker→droplet) | api-origin-stage.oglasino.com | Configured (Phase 1C) — gray cloud |
| Domain — CDN | cdn-stage.oglasino.com | Configured |

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
