# DigitalOcean Databases

## Production — Managed Postgres

Prod uses DigitalOcean's managed Postgres service. Backups, failover,
and point-in-time recovery are handled by DO. Specifics (cluster name,
size, region, connection string location) populated during Phase 1.

- **Cluster:** TBD
- **Size:** TBD
- **Region:** TBD
- **Backup policy:** managed (default)
- **Connection details:** stored as a GH Secret on `oglasino-backend`
  (see [`../overview/secret-inventory.md`](../overview/secret-inventory.md))

## Stage — Postgres in Docker

Stage runs Postgres inside the Docker Compose stack on the stage
droplet (see [`droplets.md`](droplets.md)). It is **not** a managed
service.

- **No automated backups by default.** Stage is treated as ephemeral.
- **Test data preservation:** if QA workflows ever require preserving
  stage data across restarts, schedule a Phase 5 followup to add a
  backup cron (`pg_dump` to R2 daily) and document recovery in
  [`../runbooks/recover-from-droplet-loss.md`](../runbooks/recover-from-droplet-loss.md).
