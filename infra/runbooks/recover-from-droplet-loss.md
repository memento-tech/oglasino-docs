# Runbook — Recover from Droplet Loss

TBD — written during Phase 5.2 of
[`../master-plan.md`](../master-plan.md).

Will cover: what to do if the stage or prod droplet dies (hardware
failure, accidental destroy, region outage). Includes provisioning a
replacement droplet, restoring database (managed Postgres restore for
prod; rebuild-from-empty for stage unless backup cron added),
re-running compose stack, updating DNS A record, verifying recovery.
