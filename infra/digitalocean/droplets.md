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

### oglasino-stage
- **Size:** 2 GB / 1 vCPU / 50 GB SSD ($12/mo)
- **Region:** Frankfurt 1 (FRA1)
- **OS:** Ubuntu 24.04 LTS
- **IP:** 142.93.106.90
- **Created:** 2026-05-09
- **What runs:** Spring Boot, Postgres, Redis, Elasticsearch (all in Docker Compose)
- **SSH:** port 2202, user `igor`, key-based only (no password auth)
- **Hostname:** oglasino-stage

**Original size was 1 GB; upgraded to 2 GB on 2026-05-09 after
Elasticsearch could not start within the 1 GB memory budget. The
upgrade was a pre-committed escape hatch documented before
provisioning. No further upgrades anticipated for stage.**

**Memory budget at steady state (data services only, before Spring lands):**

| Component | Used | Cap |
|---|---|---|
| Elasticsearch | ~745 MB | 900 MB |
| Postgres | ~18 MB | 200 MB |
| Redis | ~13 MB | 80 MB |
| Spring (when published) | ~450 MB expected | 600 MB |
| System (kernel, Docker daemon, SSH, fail2ban) | ~280 MB | n/a |
| **Total expected when fully running** | **~1.5 GB** | of 2 GB |

This leaves ~500 MB headroom, sufficient for the 4-tester / 40-product
workload of stage. Sustained 80%+ memory pressure or any swap usage
should trigger an upgrade to 4 GB.
