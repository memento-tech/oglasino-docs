# Infrastructure Master Plan

This is the live checklist for building Oglasino's stage and prod
environments from scratch. Each item links to the doc page that gets
populated when the item completes.

## Phase 0 — Documentation scaffold

- [ ] **0.1** Create empty `oglasino-docs` private GitHub repo *(manual)*
- [ ] **0.2** Bootstrap docs repo skeleton + master plan + secret inventory template *(agent task — this PR)*

## Phase 1 — Provision stage infrastructure

### 1A — Firebase
- [x] **1A.1** Create new Firebase project `oglasino-stage` *(manual)*
- [x] **1A.2** Create new Firebase project `oglasino-prod` *(manual)*
- [x] **1A.3** Configure auth providers (email, Google) on both *(manual)*
- [x] **1A.4** Generate Admin SDK service account JSON for each *(manual + record in secret-inventory.md)*
- [x] **1A.5** Generate FCM VAPID key, web push configuration on each *(manual + record)*
- [x] **1A.6** Document both projects in `firebase/projects.md`

### 1B — DigitalOcean stage droplet
- [x] **1B.1** Provision 1 GB droplet, Ubuntu LTS, same region as prod *(manual)*
- [x] **1B.2** Initial server hardening (SSH keys only, fail2ban, ufw firewall) *(agent task — bash script)*
- [x] **1B.3** Install Docker + Docker Compose *(agent task)*
- [x] **1B.4** Write `docker-compose.yml` for Spring + Postgres + Redis + ES with resource limits *(agent task)*
- [x] **1B.5** Test the compose stack boots within 1 GB RAM *(manual verification)*
- [x] **1B.6** Document droplet config in `digitalocean/droplets.md`

### 1C — Cloudflare
- [x] **1C.1** Worker rename PR: `oglasino-prod-router` → `oglasino-router-prod` + create `oglasino-router-stage` *(agent task)*
- [x] **1C.2** Configure DNS for `stage.oglasino.com` → Vercel *(manual)*
- [x] **1C.3** Configure DNS for `api-stage.oglasino.com` → stage droplet *(manual)*
- [x] **1C.4** Configure DNS for `api-origin-stage.oglasino.com` (api-origin pattern, details TBD with Igor) *(manual)*
- [x] **1C.5** Configure bot/SEO protection on stage subdomains (X-Robots-Tag, robots.txt) *(agent task in router Worker)*
- [x] **1C.6** Document all Workers, R2 buckets, DNS records in `cloudflare/*.md`

**Phase 1C status: substantively complete.** Stage backend reachable
via api-stage.oglasino.com. Stage frontend (stage.oglasino.com) waits
on Phase 1E (Vercel stage env). Bot/SEO protection: HTTP header done
in Worker; robots.txt deferred to Phase 3D in oglasino-web repo.

### 1D — Apple + Google
- [ ] **1D.1** Enroll in Apple Developer Program (Individual, $99/year) *(manual — IN PROGRESS)*
- [ ] **1D.2** Configure App Store Connect app listing *(manual, after 1D.1)*
- [ ] **1D.3** Configure TestFlight track for stage builds *(manual)*
- [ ] **1D.4** Create Internal Testing track on Google Play Console *(manual)*
- [ ] **1D.5** Document mobile app distribution in `google-play/*.md` and `apple/app-store.md`

### 1E — Vercel stage
- [x] **1E.1** Add stage environment to `oglasino-web` Vercel project, point at `stage` branch *(manual)*
- [x] **1E.2** Configure stage env vars (populate in Phase 2) *(manual)*
- [x] **1E.3** Document in `vercel/deployments.md`
- [~] **1E.4** Add deploy-stage.yml workflow + vercel.json update *(workflow file written; awaiting first push to stage branch to verify end-to-end)*

## Phase 2 — Secret rotation + inventory

- [ ] **2.1** Inventory every existing secret across all repos and infra into `overview/secret-inventory.md` *(agent task — grep + Igor populates values)*
- [ ] **2.2** Generate new secret values for stage env *(agent task)*
- [ ] **2.3** Generate new secret values for prod env *(agent task)*
- [ ] **2.4** Update GH Actions secrets in `oglasino-backend` *(manual)*
- [ ] **2.5** Update GH Actions secrets in `oglasino-web` *(manual)*
- [ ] **2.6** Update GH Actions secrets in `oglasino-image-worker` *(manual)*
- [ ] **2.7** Update GH Actions secrets in `oglasino-firestore-rules` *(manual)*
- [ ] **2.8** Update GH Actions secrets in `oglasino-expo` *(manual)*
- [ ] **2.9** Update Vercel env vars for both environments *(manual)*
- [ ] **2.10** Update droplet env vars on prod droplet (rotate, currently using dev Firebase) *(manual + redeploy)*
- [ ] **2.11** Update secret inventory doc with rotation date for every secret

## Phase 3 — Repository plumbing

### 3A — oglasino-image-worker
- [ ] **3A.1** Send `dev → stage` rename prompt *(prompt already drafted, agent task)*

### 3B — oglasino-firestore-rules
- [ ] **3B.1** Send rules-repo bootstrap prompt *(prompt already drafted, agent task)*
- [ ] **3B.2** Phase B rule tightening *(deferred — separate work after stage env is up)*

### 3C — oglasino-backend
- [ ] **3C.1** Audit current branch state (does it have `dev`? `stage`?) *(manual check)*
- [ ] **3C.2** Add missing branches *(manual)*
- [ ] **3C.3** Add `stage-deploy.yml` workflow *(agent task)*
- [ ] **3C.4** Update `prod-deploy.yml` to use new prod credentials *(agent task)*
- [ ] **3C.5** Document branch model + deploy workflows in `github/workflows.md`

### 3D — oglasino-web
- [ ] **3D.1** Audit current branch state *(manual)*
- [ ] **3D.2** Add missing branches *(manual)*
- [ ] **3D.3** Add `stage-deploy.yml` workflow OR configure Vercel to auto-deploy from `stage` *(manual + agent task)*
- [ ] **3D.4** Update `prod-deploy.yml` *(agent task)*
- [ ] **3D.5** Document in `github/workflows.md`

### 3E — oglasino-expo
- [ ] **3E.1** Configure `eas.json` with development/preview/production profiles *(agent task)*
- [ ] **3E.2** Configure single applicationId `com.oglasino` with env-specific app names via `app.config.ts` *(agent task)*
- [ ] **3E.3** Configure environment variables per profile *(agent task)*
- [ ] **3E.4** Configure GH Actions to trigger EAS builds on push to `stage` and `main` *(agent task)*
- [ ] **3E.5** Configure expo-notifications + APNs key for iOS *(manual + agent task)*
- [ ] **3E.6** Document in `expo/cloud-setup.md`

## Phase 4 — Cutover

- [ ] **4.1** First deploy of stage backend from `stage` branch *(manual)*
- [ ] **4.2** First deploy of stage web from `stage` branch *(manual)*
- [ ] **4.3** First EAS build of stage RN app, install on test device *(manual)*
- [ ] **4.4** End-to-end smoke test on stage (user, image, chat, notification) *(manual)*
- [ ] **4.5** First deploy of stage Cloudflare Workers *(manual)*
- [ ] **4.6** First deploy of stage Firestore rules *(manual)*
- [ ] **4.7** Redeploy prod backend with new prod Firebase credentials *(manual)*
- [ ] **4.8** Redeploy prod web with new prod Firebase credentials *(manual)*
- [ ] **4.9** New EAS build of prod RN app with new credentials *(manual)*
- [ ] **4.10** Verify prod still works end-to-end *(manual)*
- [ ] **4.11** Decommission old "Oglasino Dev" Firebase project *(manual)*
- [ ] **4.12** Decommission old "Oglasino Prod" Firebase project *(manual)*

## Phase 5 — Documentation finalization

- [ ] **5.1** Walk through every doc file, fill in anything missing *(manual review)*
- [ ] **5.2** Write the runbooks *(agent task with Igor's input)*
- [ ] **5.3** Architecture diagram (Mermaid) showing how everything connects *(agent task)*
- [ ] **5.4** Final review — pretend you're a new hire reading this for the first time *(manual)*

## Status legend

- `[ ]` = not started
- `[~]` = in progress
- `[x]` = complete
- `[!]` = blocked (add a note explaining what's blocking)
