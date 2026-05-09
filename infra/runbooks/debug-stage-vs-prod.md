# Runbook — Debug Stage vs Prod Divergence

TBD — written during Phase 5.2 of
[`../master-plan.md`](../master-plan.md).

Will cover: common gotchas when stage and prod diverge — different
Firebase project IDs, different DB sizes (managed vs in-Docker
Postgres), different resource limits, different secret values, missing
indexes, env var typos. Includes a checklist for "works on stage,
breaks on prod" investigations.
