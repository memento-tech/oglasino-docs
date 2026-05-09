# Deployment

## Stage

Manual rsync + remote `docker compose up -d` from local laptop.
Workflow:

1. Edit `docker-compose.yml`, configs, etc. in `oglasino-stage-infra`
2. Commit + push
3. From laptop: `./scripts/deploy.sh`

The deploy script:
- rsyncs the working tree to droplet (excluding .git, .env)
- runs `docker compose pull` then `docker compose up -d` over SSH
- shows status afterward

See `oglasino-stage-infra/scripts/` for all helper scripts:
- `deploy.sh` — sync + restart
- `ssh.sh` — interactive SSH or single command
- `logs.sh <service>` — tail logs
- `exec.sh <service> [cmd]` — exec into container

Currently `deploy.sh` only deploys data services
(`postgres redis elasticsearch`). The Spring service in
`docker-compose.yml` references an image that doesn't exist yet
(`ghcr.io/memento-tech/oglasino-backend:stage`). Phase 3C of the
master plan publishes the first Spring image and adds Spring to the
`SERVICES` list in `deploy.sh`.

CI/CD via GitHub Actions deferred — explicit non-goal for now,
documented in `oglasino-stage-infra/.github/workflows/README.md`.

## Prod

Existing deploy mechanism (TBD — fill in when prod is reviewed
during cutover).

## SSH access

Both prod and stage droplets use SSH port 2202 (matching prod
convention). Igor's local `~/.ssh/config`:

```
Host oglasino-stage
    HostName 142.93.106.90
    User igor
    Port 2202
    IdentityFile ~/.ssh/oglasino
    IdentitiesOnly yes
    UseKeychain yes
    AddKeysToAgent yes
```

The key (`~/.ssh/oglasino`) is stored in macOS Keychain. Add
equivalent entries for prod droplets (TBD when prod is reviewed).
