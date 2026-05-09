# DigitalOcean Droplets

## Production

### oglasino-prod-spring (existing)
- **Size:** TBD (record actual)
- **Region:** TBD
- **What runs:** Spring Boot, Redis
- **OS:** TBD
- **IP:** TBD (consider: should this be public?)

### oglasino-prod-elasticsearch (existing)
- **Size:** TBD
- **Region:** TBD
- **What runs:** Elasticsearch
- **IP:** TBD

## Stage

### oglasino-stage (to be provisioned in Phase 1B)
- **Size:** 1 GB / 1 vCPU (smallest)
- **Region:** TBD (same as prod for latency consistency)
- **What runs:** Spring + Postgres + Redis + Elasticsearch (all in Docker Compose)
- **OS:** Ubuntu LTS
- **IP:** TBD

**Note:** 1 GB is below recommended floors for Spring + Postgres + Redis +
Elasticsearch combined. Pre-committed upgrade trigger: first time stage
becomes unusably slow → upgrade to 2 GB ($12/mo more). Don't debug — upgrade.
