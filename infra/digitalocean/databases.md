# Databases

## Stage — Postgres in Docker on droplet

Stage runs Postgres 16 (alpine variant) inside the Docker Compose stack
on the oglasino-stage droplet. Configuration in
`oglasino-stage-infra/docker-compose.yml`.

- **Image:** `postgres:16-alpine`
- **Memory cap:** 200 MB (uses ~18 MB at steady state)
- **Persistence:** Docker named volume `postgres_data` — survives
  container restart but lost on `docker compose down -v`
- **No backups configured** — acceptable for stage. JSON seed data
  in backend's stage profile re-populates 40 test products on app
  startup. If real test data needs preservation, set up `pg_dump` cron
  to a local directory or external storage.
- **Network:** internal only (`oglasino_internal` Docker network).
  Not exposed externally. Spring connects via hostname `postgres:5432`.
- **Credentials:** `POSTGRES_USER`, `POSTGRES_PASSWORD`,
  `POSTGRES_DB` set in `.env` on droplet (not in git). See
  `secret-inventory.md`.

## Prod — Managed Postgres (existing)

Already provisioned. Details to be filled in when prod is reviewed
during cutover.
