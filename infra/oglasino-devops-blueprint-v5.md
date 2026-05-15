# Oglasino — Production DevOps Blueprint (v5)

**Stack:** Spring Boot 4.0.5 (Java 21 + preview) · DigitalOcean Managed Postgres · Redis (Docker) · Elasticsearch 9.0.1 (Docker) · Next.js 15 App Router (Vercel) · Cloudflare (DNS / Workers / R2 / KV / WAF) · DigitalOcean droplets

**Audience:** Solo founder shipping a Serbia-first marketplace. Prod-only. Iterate fast on prod with maintenance gating + tested rollback.

**Philosophy:** Boring tech. Few moving parts. Security non-negotiable. Cost-disciplined. Optimize for fast iteration with strong safety nets (maintenance flag, image rollback, DB backups).

---

## What changed from v4

This revision captures the security & operational work executed in the May 15 session. Everything in this section is **deployed and verified working** as of the v5 publish date, not aspirational.

The big operational changes:

- **UFW Cloudflare-only lockdown active on both prod and stage.** Ports 80/443 are now Cloudflare-source-IP only on `oglasino-prod` and `oglasino-stage`. SSH on 2202 unchanged. Direct origin-IP scans from non-Cloudflare sources get TCP timeouts at the kernel. The "exposed origin" concern around `api-origin*.oglasino.com` records is now mitigated.
- **`update-ufw-cloudflare.sh` is patched.** The original v4 version of the script bails on first run because `grep 'Cloudflare'` exits non-zero when there are no existing CF rules to clean up, and `set -euo pipefail` kills the script before it adds any rules. The patched version wraps the cleanup pipeline in `{ ... } || true` and uses `xargs -r`. The v5 §4.9 listing is the patched version. **The patched script worked correctly on both prod and stage.**
- **Cron schedule changed from weekly to daily.** Originally Monday 04:00, now `0 4 * * *` (every day at 04:00 Europe/Belgrade). Cloudflare's IP list rarely changes but daily costs effectively nothing and gives faster recovery if it does. Log written to `/var/log/ufw-cf.log` on each droplet.
- **Reserved IPs assigned to both prod and stage.** `oglasino-prod` → `157.245.22.189`. `oglasino-stage` → `139.59.204.182`. Free while attached. Survive droplet rebuild/destroy. Detachable and movable in seconds — the disaster-recovery story is now "spin up replacement, move Reserved IP, done; no DNS change needed."
- **All Cloudflare DNS records migrated off anchor IPs.** `api-origin.oglasino.com` and `api-origin-stage.oglasino.com` point at Reserved IPs (gray cloud). The proxied "fallback" records (`api`, `oglasino.com`, `api-stage`, `stage`) — whose Content fields are only consulted if the Worker route fails — were also moved off anchor IPs onto the Reserved IPs. No public DNS record anywhere now references the old anchor IPs `46.101.166.120` (prod) or `68.183.211.5` (stage).
- **ES droplet UFW rule corrected.** The rule allowed `9200/tcp` from `10.114.0.2` — a leftover from a previous backend droplet that no longer exists. The current backend's VPC IP on the `default-fra1` VPC is `10.114.0.6`. Rule was deleted and recreated with the correct source. Note: this rule is largely decorative — see "Docker bypass" below.
- **Documented the dual-VPC reality.** Both `oglasino-prod` and `oglasino-elasticsearch` have **two** network interfaces. `eth0` carries the public IP plus a private IP on `10.19.0.0/16`. `eth1` carries the "real" project VPC (`10.114.0.0/20`, the `default-fra1` VPC visible in the DO panel). `eth1` is the interface that actually carries backend ↔ ES traffic. The blueprint and `.env` consistently use the `10.114.0.x` addresses for inter-droplet config. References to `10.19.0.x` are private interfaces that the application stack does not use.

The hard-won lessons added in this session:

- **Daily cron alone does not lock anything down.** The cron schedules the script; the script does the actual work. After installing the cron, you **must** run the script manually once or wait until 04:00 the next day for it to take effect. The first run is what flips state.
- **First-run bug in the v4 lockdown script.** Documented in detail above and patched in §4.9. The lesson generalizes: `set -e` + a leading `grep` over a list that's normally non-empty in steady state but empty on first run is a classic "works in testing, fails on bootstrap" pattern. Either guard with `|| true` or use a tolerant operation.
- **Docker bypasses UFW for published ports.** When a container publishes a port (e.g. `9200:9200` in `docker-compose.yml`), Docker inserts `ACCEPT` rules in iptables' `FORWARD` chain. These run *before* UFW's filters in the `INPUT` chain see anything, because traffic destined for a container IP doesn't traverse `INPUT`. Net result: **UFW rules for Docker-published ports are documentation, not enforcement.** Real protection comes from: (a) binding the published port to a specific interface (e.g. `"10.114.0.4:9200:9200"` instead of `"9200:9200"`) so Docker only listens where you want, (b) DO VPC isolation, (c) application-layer auth (e.g. `xpack.security` for ES). On `oglasino-elasticsearch`, all three are in place; the UFW rule is cosmetic.
- **DO droplets have a private IP visible in the DO panel that does not match `ip addr` output for `eth0`.** This bit us hard when diagnosing the ES connection. The DO panel "Private IP" field is the `eth1` address (the project VPC). Use the DO panel as the source of truth for inter-droplet IPs, not the host's `ip addr`.
- **Stale UFW source-IP rules from rebuilt droplets are a security audit landmine.** A rule referencing a former droplet's VPC IP looks correct in `ufw status` but enforces nothing (or worse, references an IP that gets recycled to a different tenant). Post-rebuild, audit every UFW rule that references a private IP.
- **Reserved IPs are essentially free.** Assign one to every droplet on day one. The cost is `$0` while attached, ~$4/mo only if unattached.
- **The proxied DNS "Content" field is a fallback origin.** If the Cloudflare Worker route is deleted or the Worker crashes, Cloudflare falls back to the DNS `Content` IP for proxied records. Anything stale there is a risk surface. Point fallback IPs at something you control (Reserved IP) rather than letting them rot.

The Vercel custom-domain observation (carried into the "What's left" section at the end):

- `oglasino.com`, `www.oglasino.com`, `stage.oglasino.com` are **not** configured as custom domains in Vercel. The Worker forwards via `env.FRONTEND_ORIGIN` → `https://oglasino-web-prod.vercel.app` (and the stage equivalent), so frontend serves correctly. But Vercel sees `Host: oglasino-web-prod.vercel.app`, not the user's actual hostname. Analytics, edge config, and any host-keyed behavior are attributed to the `.vercel.app` hostname. Cosmetic for now, worth fixing before launch.

---

## What changed from v3

The big architectural changes:

- **Frontend moved to Vercel.** Cloudflare Pages was assumed for v3; Next.js 15 App Router with SSR is better served by Vercel's native runtime. Vercel + Cloudflare Worker proxy gives you SSR with maintenance gating intact.
- **Cloudflare Worker pattern matured.** One Worker (`oglasino-prod-router`) handles all four hostnames, KV-driven maintenance flag, edge-cached backend health probe, admin bypass path, three-tier rate limiting bindings. No separate frontend router.
- **Branch model: `dev` → `main`.** No more pushes to master. Frontend has `dev`/`main`/`production` (production is a phantom branch for Vercel UI validation; never updated). Backend has the same `dev`/`main` split.
- **DNS structure crystallized.** Six records: apex, www, api, api-origin (gray cloud — Worker bypass), cdn (R2 connect-domain), www.api removed (multi-level cert issue).
- **Elasticsearch upgraded to 9.0.1.** Spring Boot 4.0.5 ships Spring Data ES 6.x targeting ES 9.x. ES 8.x produces `Expecting a response body, but none was sent` errors on `indices.exists`.
- **`MaxRAMPercentage=70`.** Bumped from 50 — managed Postgres freed memory. JVM heap is now ~1.05 GB on the 2 GB backend droplet.
- **Caddy serves `api-origin.oglasino.com`, not `api.oglasino.com`.** TLS cert needs to match the SNI hostname Cloudflare connects with, which is the gray-cloud subdomain.
- **No Caddy `trusted_proxies cloudflare` directive.** Stock Caddy doesn't include the `caddy-cloudflare-ip` module. Skip it; the UFW Cloudflare-only lockdown is the real security boundary.
- **Rate limiting via Worker bindings, not Cloudflare dashboard rules.** Free-tier dashboard rate limiting is capped at 10s windows — useless. Worker bindings give 60s windows for free.
- **Firebase service account: file-mounted secret, not classpath.** Service account JSON moves out of `src/main/resources` (compromised the moment it hits git history) into `/run/secrets/firebase.json` mounted from `/opt/oglasino/secrets/firebase.json`. Spring Boot reads via configurable path.

The hard-won lessons:

- **Ubuntu 22.10+ socket activation gotcha:** `sshd_config Port` directive is ignored when `ssh.socket` controls the port. Editing `sshd_config` to change SSH port has no effect; you must edit `ssh.socket` directly. Multiple hours lost in this session — see §1.4 for the safe sequence.
- **Pre-deploy SSH lockout safety net:** never close port 22 in UFW until the new port is verified working from a separate terminal. Running `ufw allow 2202/tcp` while SSH still listens on 22 means UFW rejects all SSH traffic — instant lockout.
- **`passwd <user>` for the recovery console:** DO recovery console requires a Unix password. Set one for your sudo user *before* you need recovery.
- **`spring.config.import` belongs in `application.yml`, not `application-local.yml`.** Profile-specific YAML loads after datasource bean construction; the import has to be in the base config to be processed in time.
- **`docker compose down -v` is required after Postgres password change.** `POSTGRES_PASSWORD` is read only on first init.
- **Rotation discipline for secrets pasted in chat tools.** Anything that leaves your laptop into a chat is compromised. Rotate the leaked secret, paste placeholders only, build the habit.

---

## TL;DR — Architecture

```
                            ┌────────────────────────────────────────┐
   Users ──────────────────▶│  Cloudflare (DNS + WAF + CDN)          │
                            │  Worker: oglasino-prod-router          │
                            │  KV: maintenance.active flag           │
                            │  Rate limiters: AUTH/WRITE/GLOBAL      │
                            └────────────┬───────────────────────────┘
                                         │
            ┌────────────────────────────┼─────────────────────────────┬────────────────┐
            │                            │                             │                │
            ▼                            ▼                             ▼                ▼
  oglasino.com  /  www         api.oglasino.com               cdn.oglasino.com    api-origin
  (Vercel,                     (Worker → api-origin           (R2 connect-         (gray cloud,
   Next.js 15 SSR,             → Caddy → Spring Boot)         domain to            DNS only,
   region fra1)                                                oglasino-images)    Worker bypass)
            │                            │
            │                            │ private VPC (10.x.x.x)
            │                            │
            │       ┌────────────────────┼─────────────────────┐
            │       │                    │                     │
            │       ▼                    ▼                     ▼
            │  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────┐
            │  │ Backend Droplet│  │ ES Droplet     │  │ Managed Postgres     │
            │  │ ─────────────  │  │ ────────────   │  │ ─────────────────    │
            │  │ Caddy (host)   │  │ ES 9.0.1       │  │ Basic 1 GB ($15/mo)  │
            │  │ Spring Boot    │  │  (Docker)      │  │ Port 25060, TLS-only │
            │  │  (Docker)      │  │ private VPC    │  │ Daily backups + PITR │
            │  │ Redis (Docker) │  │ only           │  │ oglasino_app user    │
            │  └────────────────┘  └────────────────┘  └──────────────────────┘
            │
            │  Backend deploy: GitHub → CI → flip maintenance ON → SSH deploy → flip OFF
            │  Frontend deploy: GitHub → CI → flip maintenance ON → vercel deploy --prod → flip OFF
            └────────────────────────────────────────────────────────────────────────────
```

**Infrastructure:**

| | Backend (`oglasino-prod`) | Elasticsearch (`oglasino-es`) | Managed Postgres |
|---|---|---|---|
| Region | FRA1 | FRA1 (same VPC) | FRA1 (same VPC) |
| Size | 2 vCPU / 2 GB / 60 GB | 2 vCPU / 2 GB / 60 GB | 1 vCPU / 1 GB / 10 GB |
| Cost | ~$18/mo | ~$18/mo | $15/mo |

**Frontend:** Vercel Hobby (free tier — generous for unlaunched MVP)

**Total infra: ~$55/mo** for backend + ES + Postgres. Plus DO weekly backups (~$4) + R2 (under $1 at low scale) + CF Free + Vercel Free = **~$60/mo all in**.

> **Honest 2 GB note.** Pulling Postgres off freed ~400 MB. JVM has ~1.05 GB heap with `MaxRAMPercentage=70`. Plan to upgrade backend to 4 GB ($24/mo total) the moment swap usage stays >200 MB or response P99 latency degrades.

> **Why managed Postgres at this stage.** $15/mo buys: automated daily backups, 7-day point-in-time recovery, no DB ops time, JVM headroom, future failover migration path. Highest-ROI cloud expense in the stack.

---

# PHASE 1 — Initial Droplet Hardening

> **Run on BOTH droplets.** ES droplet is on private VPC, but its public interface still gets scanned the moment it has a public IP.
>
> **Priority: CRITICAL NOW.** First hour after droplet creation.

## 1.1 First login

Add your SSH public key during droplet creation in DO panel. Never password auth.

```bash
ssh-keygen -t ed25519 -C "igor@oglasino"     # if you don't have one
cat ~/.ssh/id_ed25519.pub                       # paste into DO when creating droplet
ssh root@DROPLET_IP
```

## 1.2 Update everything

```bash
apt update && apt upgrade -y
apt autoremove -y
reboot
```

## 1.3 Sudo user (critical: set password)

```bash
adduser igor                                  # set a strong password — REMEMBER IT
usermod -aG sudo igor
mkdir -p /home/igor/.ssh
cp /root/.ssh/authorized_keys /home/igor/.ssh/
chown -R igor:igor /home/igor/.ssh
chmod 700 /home/igor/.ssh
chmod 600 /home/igor/.ssh/authorized_keys
```

> **Critical:** the password you set for `igor` is what you'll need at the **DigitalOcean recovery console** if you ever lock yourself out via SSH. Save it in your password manager **right now**, before you need it. The recovery console requires a Unix password — SSH keys don't help there.

Test from a **new terminal** before closing root:
```bash
ssh igor@DROPLET_IP
sudo whoami           # should print "root"
exit
```

## 1.4 SSH hardening — the safe sequence

> ⚠ **This is the section that locked us out for hours in v3.** Read all of it before changing anything. The mistake was assuming `sshd_config Port` directive controlled the listening port. On Ubuntu 22.10+, `ssh.socket` (systemd socket activation) controls it instead — `sshd_config` Port is silently ignored.

### 1.4.1 Open a second SSH session as a safety net

Before touching anything, open a **second terminal tab on your laptop** and SSH in:

```bash
ssh igor@DROPLET_IP        # in the new tab
# leave it idle, do not close it until SSH on new port is verified
```

This is your escape hatch. Existing TCP connections survive sshd restarts; if anything breaks, you have a working shell to fix it from.

### 1.4.2 Edit sshd_config

```bash
sudo nano /etc/ssh/sshd_config
```

Set/change:
```ini
Port 2202
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
AllowUsers igor deploy
Protocol 2
```

Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

**Verify the save took:**
```bash
sudo grep "^Port" /etc/ssh/sshd_config
# MUST print "Port 2202"
```

If it prints anything else, the save didn't work. Re-edit. **Do not proceed** until grep shows `Port 2202`.

```bash
sudo sshd -t
# Must print nothing (silence = success). If errors, fix.
```

### 1.4.3 Edit the socket — this is what actually changes the port

On Ubuntu 22.10+, `ssh.socket` controls the listening port. The `sshd_config` Port directive is ignored. You must override the socket:

```bash
sudo systemctl edit ssh.socket
```

In the editor that opens, paste exactly this:
```ini
[Socket]
ListenStream=
ListenStream=2202
```

The empty `ListenStream=` line is **mandatory** — it clears the inherited port 22 binding. Without it, the socket listens on both 22 and 2202.

Save. Verify the override file:
```bash
sudo cat /etc/systemd/system/ssh.socket.d/override.conf
# should show exactly what you pasted
```

### 1.4.4 Open port 2202 in UFW BEFORE closing 22

This is the safety-net rule. Open the new port while keeping the old one open:

```bash
sudo ufw allow 22/tcp comment 'SSH (temporary, will remove after 2202 verified)'
sudo ufw allow 2202/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP - Caddy'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw status verbose
# Should show BOTH 22/tcp ALLOW and 2202/tcp ALLOW
```

> **Never run `ufw allow 2202/tcp` without first running `ufw allow 22/tcp`.** That's how we locked ourselves out. UFW's default policy is `deny incoming`; opening only 2202 means port 22 is silently rejected — and if SSH is still actually listening on 22 (which it is, because the socket override hasn't been activated yet), every existing and new SSH connection breaks.

### 1.4.5 Reload systemd, restart the socket

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

This will **not** kick you out — established TCP connections don't care what the listening socket does.

### 1.4.6 Verify port 2202 is listening

```bash
sudo ss -tlnp | grep ssh
# Should show:
# LISTEN 0  4096  0.0.0.0:2202  ...  users:(("systemd",pid=1,fd=N))
# LISTEN 0  4096  [::]:2202     ...  users:(("systemd",pid=1,fd=N))
```

Must show `:2202`, not `:22`. If it shows `:22`, the override didn't take effect. Re-check `/etc/systemd/system/ssh.socket.d/override.conf`.

### 1.4.7 Test from a THIRD terminal on your laptop

Don't close the existing two sessions. Open a fresh terminal tab:

```bash
ssh -v -p 2202 igor@DROPLET_IP
```

If this connects: you have three working sessions (two on 22, one on 2202). The 2202 session proves the new port works.

If it fails with "Connection refused": port 2202 isn't listening. Recheck §1.4.6.
If it fails with "Permission denied": port 2202 listening but auth wrong. Don't proceed — investigate.

### 1.4.8 Only after §1.4.7 succeeds, close port 22

```bash
sudo ufw delete allow 22/tcp
sudo ufw status verbose
# Should now show only 2202/tcp for SSH
```

The two old port-22 sessions remain alive (existing connections). They keep working until you log out — free safety net for a few more minutes.

### 1.4.9 Update your laptop's SSH config

So you never type `-p 2202` again. On your **laptop**:
```bash
nano ~/.ssh/config
```

Add:
```
Host oglasino-prod
    HostName 157.245.22.189
    User igor
    Port 2202
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host oglasino-stage
    HostName 139.59.204.182
    User igor
    Port 2202
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host oglasino-es
    HostName 10.114.0.4
    User igor
    Port 2202
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ProxyJump oglasino-prod
```

> **v5 note: hostnames use the Reserved IPs**, not the anchor IPs. The anchor IPs (`46.101.166.120` for prod, `68.183.211.5` for stage) still route to the droplets but are no longer the "canonical" address. Using the Reserved IP means: if you ever rebuild a droplet, you can move the Reserved IP to the new droplet in seconds and your SSH config still works without edits. The `oglasino-es` host uses the project VPC IP `10.114.0.4` (the value visible in the DO panel for the ES droplet, which is the `eth1` address — not the `10.19.0.7` you'd see in `ip addr show eth0`).

The `ProxyJump` for ES means "connect to the ES droplet through the backend droplet" — the ES droplet's UFW only allows SSH from the VPC, so you tunnel through.

```bash
ssh oglasino-prod      # works
ssh oglasino-es        # works (via jump)
```

### 1.4.10 Final cleanup

Close all old port-22 sessions. Open a fresh terminal:
```bash
ssh oglasino-prod
# should connect via port 2202 with no flags
```

You're done with SSH. Don't touch it again unless you need to.

## 1.5 Fail2Ban

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = 2202
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

## 1.6 Swap — critical at 2 GB

At 2 GB RAM, swap is mandatory. JVM GC, ES indexing, or a burst of traffic will OOM-kill without it.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

> **ES caveat:** Elasticsearch officially recommends `bootstrap.memory_lock=true` and disabling swap. At 2 GB this is impossible — ES needs the swap cushion. Leave swap on. Plan to upgrade ES to 4 GB before meaningful index size, then disable swap.

## 1.7 Timezone, NTP, monitoring agent, unattended upgrades

```bash
sudo timedatectl set-timezone Europe/Belgrade
sudo timedatectl set-ntp true

sudo apt install unattended-upgrades apt-listchanges -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

In `/etc/apt/apt.conf.d/50unattended-upgrades`:
```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

DO monitoring agent:
```bash
curl -sSL https://repos.insights.digitalocean.com/install.sh | sudo bash
```

In DO panel: enable weekly backups on backend ($3.60/mo), optional on ES (indexes rebuildable). Set CPU/RAM/disk alerts at 80%.

## 1.8 DigitalOcean VPC

When creating the second droplet, attach to **same VPC** as the first. If you forgot, move it via panel.

Verify private connectivity from backend → ES:
```bash
# On backend droplet
ping -c 3 10.114.0.4       # ES droplet's private VPC IP (eth1, default-fra1 VPC) — get from DO panel
```

> **v5 note on the dual-VPC reality.** DO droplets on the `default-fra1` VPC have two network interfaces. `eth0` carries the public IP and a private IP on a different VPC range (e.g. `10.19.0.0/16`). `eth1` carries the project VPC IP visible in the DO panel (e.g. `10.114.0.0/20`). **Always use the panel-visible IP** (the `eth1` value) for `.env` configs, UFW source-IP rules, and cross-droplet references. The `eth0` private IP is not the one your application stack uses, and using it leads to silent connection timeouts. Today (v5 publish day), prod backend's panel IP is `10.114.0.6`, ES is `10.114.0.4`.

VPC traffic is **free** (no bandwidth charges) and **never leaves DO's network**.

## 1.9 Provision DigitalOcean Managed Postgres

In DO panel → **Databases → Create Database Cluster**:

| Setting | Value |
|---|---|
| Engine | PostgreSQL 16 |
| Plan | **Basic** → 1 vCPU / 1 GB / 10 GB SSD — **$15/mo** |
| Datacenter | **FRA1** (must match droplets' region) |
| VPC Network | Same VPC as your droplets |

Wait ~5 minutes for provisioning.

Once ready, in **Settings**:

1. **Trusted sources** — add **only** the backend droplet's name (NOT public IP). This restricts cluster connections to that droplet via VPC.
2. **Connection details** — note the *Private network* connection string. Format:
   ```
   postgresql://doadmin:PASSWORD@private-cluster-do-user-XXXXX-0.b.db.ondigitalocean.com:25060/defaultdb?sslmode=require
   ```

3. **Download CA certificate** — click "Download CA certificate" (top right). You'll get `ca-certificate.crt`. Save to `/opt/oglasino/secrets/do-postgres-ca.crt` on the backend droplet (steps below).

### 1.9.1 Create app-specific user and database

Don't use `doadmin` for app connections — that's the cluster superuser. Create a scoped user:

```bash
# On the backend droplet
sudo apt install postgresql-client-16 -y

# Connect as doadmin (one-time, to create the app user)
psql "postgresql://doadmin:PASSWORD@private-cluster-do-user-XXXXX-0.b.db.ondigitalocean.com:25060/defaultdb?sslmode=require"
```

In psql:
```sql
CREATE DATABASE oglasino;
CREATE USER oglasino_app WITH PASSWORD 'GENERATE_STRONG_PASSWORD_HERE';
GRANT ALL PRIVILEGES ON DATABASE oglasino TO oglasino_app;
\c oglasino
GRANT ALL ON SCHEMA public TO oglasino_app;
\q
```

Generate the password with `openssl rand -base64 32` on your laptop, paste it into the `CREATE USER` command, save it to your password manager.

### 1.9.2 Place the CA cert on the droplet

```bash
# On the backend droplet
sudo mkdir -p /opt/oglasino/secrets
sudo chmod 700 /opt/oglasino/secrets
sudo chown root:root /opt/oglasino/secrets
```

From your laptop:
```bash
scp ~/Downloads/ca-certificate.crt igor@oglasino-prod:/tmp/
```

Back on the droplet:
```bash
sudo mv /tmp/ca-certificate.crt /opt/oglasino/secrets/do-postgres-ca.crt
sudo chown root:root /opt/oglasino/secrets/do-postgres-ca.crt
sudo chmod 600 /opt/oglasino/secrets/do-postgres-ca.crt
```

### 1.9.3 Verify connection from droplet

```bash
PGPASSWORD='YOUR_OGLASINO_APP_PASSWORD' psql \
  "host=private-cluster-do-user-XXXXX-0.b.db.ondigitalocean.com port=25060 dbname=oglasino user=oglasino_app sslmode=verify-full sslrootcert=/opt/oglasino/secrets/do-postgres-ca.crt" \
  -c "SELECT version();"
```

Should print PostgreSQL version. If "connection refused": check trusted sources in DO panel. If "TLS handshake failed": CA cert path is wrong or `sslmode` is wrong.

> **Why `sslmode=verify-full`?** It verifies the server's cert chain (using the CA you downloaded) AND verifies the hostname matches the cert. `sslmode=require` only encrypts but doesn't verify identity — vulnerable to man-in-the-middle. Always `verify-full` for production.

You're done with Phase 1. Both droplets hardened, Postgres provisioned, VPC verified.
---

# PHASE 2 — Foundation services

## 2.1 Install Docker (both droplets)

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker igor
# log out / log in for group to take effect

# Compose plugin is included in modern Docker. Verify:
docker compose version
```

## 2.2 Install Caddy (backend droplet only)

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy -y
```

> **Don't `caddy add-package github.com/WeidiDeng/caddy-cloudflare-ip`.** The `trusted_proxies cloudflare` directive that module enables is convenient for getting accurate client IPs in Caddy logs, but the UFW Cloudflare-only firewall (§4.5) is the actual security control. Stock Caddy is enough.

## 2.3 Caddyfile — serves `api-origin.oglasino.com`

This is critical: Caddy must serve `api-origin.oglasino.com`, not `api.oglasino.com`. The Worker fetches from `api-origin` (gray cloud bypass) — that's the SNI hostname Cloudflare presents to Caddy. Caddy must have a cert matching that hostname.

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.original
```

Replace contents:
```bash
sudo tee /etc/caddy/Caddyfile > /dev/null <<'EOF'
api-origin.oglasino.com {
    encode zstd gzip

    reverse_proxy localhost:8080 {
        header_up X-Real-IP {http.request.header.CF-Connecting-IP}
        header_up X-Forwarded-For {http.request.header.CF-Connecting-IP}
    }

    header {
        Strict-Transport-Security "max-age=2592000"
        X-Content-Type-Options nosniff
        Referrer-Policy strict-origin-when-cross-origin
        -Server
    }

    log {
        output file /var/log/caddy/api.log {
            roll_size 50mb
            roll_keep 5
        }
        format json
    }
}
EOF
```

Validate, prepare logs, start:
```bash
sudo caddy validate --config /etc/caddy/Caddyfile

sudo mkdir -p /var/log/caddy
sudo chown caddy:caddy /var/log/caddy

sudo systemctl enable --now caddy
sudo systemctl status caddy --no-pager
```

Watch cert provisioning:
```bash
sudo journalctl -u caddy -f --since "1 minute ago"
```

Within 30 seconds you should see:
```
obtain certificate
challenge type=http-01
certificate obtained successfully
```

> **For Let's Encrypt to succeed**, port 80 must be reachable from the internet (UFW allows it — §1.4.4) AND `api-origin.oglasino.com` DNS must resolve to your droplet's IP. We'll add that DNS record in §4.2. If you're hardening UFW to Cloudflare-only later (§4.5), do it AFTER cert provisioning works — Let's Encrypt connections come from arbitrary IPs.

Test directly:
```bash
curl -I https://api-origin.oglasino.com/
# Expect: 502 Bad Gateway (Spring Boot not running yet)
# That's fine — confirms TLS works, Caddy is serving HTTPS.
```

A 502 here is the desired result at this stage. It means: Cloudflare ↔ Caddy TLS works, Caddy is alive, only Spring Boot is missing.

## 2.4 Directory layout (backend droplet)

```bash
sudo mkdir -p /opt/oglasino/{secrets,scripts,backups/postgres}
sudo chown -R igor:igor /opt/oglasino
sudo chmod 700 /opt/oglasino/secrets
sudo chmod 700 /opt/oglasino/scripts
```

Layout:
```
/opt/oglasino/
├── docker-compose.yml      # backend + redis (no postgres — managed)
├── .env                     # main app env (mode 600)
├── .env.cf                  # Cloudflare API creds (mode 600)
├── secrets/
│   ├── do-postgres-ca.crt   # from §1.9.2 (mode 600)
│   └── firebase.json        # from §3.7 (mode 600)
├── scripts/
│   ├── maintenance-on.sh
│   ├── maintenance-off.sh
│   └── update-ufw-cloudflare.sh
└── backups/
    └── postgres/            # weekly belt-and-suspenders dumps (Phase 6)
```

Notably absent: `data/postgres/`, `data/redis/`, `postgres-init/`. Postgres is managed; Redis is cache-only with no persistence.

## 2.5 GHCR authentication (backend droplet)

CI publishes Docker images to GitHub Container Registry. The droplet pulls from there. Auth required for private repos:

On GitHub, create a Personal Access Token (classic):
- Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new
- Scopes: `read:packages`
- Save the token in your password manager

On the droplet:
```bash
echo "ghp_yourTokenHere" | docker login ghcr.io -u memento-tech --password-stdin
```

Stores creds in `~/.docker/config.json`. From now on, `docker compose pull` works against private images.

For public repos, skip this step.
---

# PHASE 3 — Application Deployment (Backend)

## 3.1 Local development setup (your laptop)

The right pattern: **deps in Docker, app in IntelliJ**. You hit the green play button daily; Docker just provides Postgres/Redis/ES.

### 3.1.1 docker-compose.local.yml

In your backend repo root:

```yaml
name: oglasino-local

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: oglasino_user
      POSTGRES_PASSWORD: ${LOCAL_PG_PASSWORD:?set LOCAL_PG_PASSWORD in .env or shell}
      POSTGRES_DB: oglasino
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U oglasino_user -d oglasino"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    ports:
      - "127.0.0.1:6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.0.1
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - cluster.routing.allocation.disk.threshold_enabled=false
    ports:
      - "127.0.0.1:9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fs http://localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  postgres_data:
  es_data:
```

> **`127.0.0.1:` prefix matters.** Without it, ports bind to all interfaces — anyone on your WiFi could connect to your local Postgres. Loopback-only is correct for local dev.

> **ES 9.0.1 (not 8.x).** Spring Boot 4.0.5 ships Spring Data ES 6.x targeting ES 9.x. ES 8.x produces `Expecting a response body, but none was sent` on `indices.exists`.

> **Local Postgres password:** even locally, don't use `password`. Generate with `openssl rand -base64 24` and set `LOCAL_PG_PASSWORD` in your shell or a gitignored `.env` file. Builds the right habit, costs nothing.

### 3.1.2 IntelliJ run config

Run → Edit Configurations → Spring Boot config:

- **Active profiles:** `local`
- **VM options:** `--enable-preview` (mandatory — your pom enables Java 21 preview features)
- **Working directory:** project root
- **JDK:** Java 21 / Project SDK

Without `--enable-preview`, the JVM rejects your compiled classes immediately at startup.

### 3.1.3 application.yml (base)

The `spring.config.import` line MUST be in `application.yml`, not a profile-specific YAML. Spring loads imports during base config processing — too late if it's in `application-local.yml`:

```yaml
spring:
  config:
    import: optional:file:.env[.properties]
  application:
    name: Oglasino
```

### 3.1.4 application-local.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/oglasino
    driver-class-name: org.postgresql.Driver
    username: oglasino_user
    password: ${LOCAL_PG_PASSWORD}
    hikari:
      minimum-idle: 2
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: create-drop      # local — wipes/recreates schema each restart
    show-sql: false
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
  elasticsearch:
    uris: http://localhost:9200
    connection-timeout: 5s
    socket-timeout: 30s
  cache:
    type: redis
    redis:
      cache-null-values: false
  devtools:
    restart:
      enabled: true

server:
  port: 8080

logging:
  level:
    com.memento.tech.oglasino: DEBUG
    org.hibernate.SQL: WARN

# Local Firebase admin (for dev work) — points at your dev Firebase project
app:
  firebase:
    credentials-path: ${FIREBASE_CREDENTIALS_PATH:./firebase-service-account.json}
```

### 3.1.5 Daily flow

```bash
# Start of day
docker compose -f docker-compose.local.yml up -d

# Hit play in IntelliJ — app runs natively on your Mac, connects to Docker deps

# End of day (optional — they idle at near-zero cost)
docker compose -f docker-compose.local.yml down
```

## 3.2 Dockerfile (committed in repo root)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn ./.mvn
RUN ./mvnw -B dependency:go-offline
COPY src ./src
RUN ./mvnw -B clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
RUN apk add --no-cache curl tini tzdata && \
    addgroup -S app && adduser -S app -G app
ENV TZ=Europe/Belgrade
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["/sbin/tini", "--"]
# --enable-preview required by pom.xml maven-compiler-plugin config.
# MaxRAMPercentage=70 leaves ~30% non-heap for metaspace, threads, direct buffers.
CMD ["java", \
     "--enable-preview", \
     "-XX:MaxRAMPercentage=70", \
     "-XX:+UseG1GC", \
     "-XX:+ExitOnOutOfMemoryError", \
     "-Djava.security.egd=file:/dev/./urandom", \
     "-jar", "/app/app.jar"]
```

> **`MaxRAMPercentage=70`.** With managed Postgres no longer competing for memory, 70% of 1500 MB container limit = ~1050 MB heap. If you ever see `OutOfMemoryError: Metaspace`, dial back to 65%. Unlikely with this stack.

## 3.3 application-prod.yml

```yaml
spring:
  application:
    name: Oglasino

  # No .env import in prod — env vars come from docker-compose environment block,
  # which reads /opt/oglasino/.env on the host.

  datasource:
    url: ${DATASOURCE_URL}
    driverClassName: ${DATASOURCE_DRIVER}
    username: ${DATASOURCE_USER}
    password: ${DATASOURCE_PASSWORD}
    hikari:
      minimum-idle: 2
      maximum-pool-size: 12         # tuned for DO Basic 1GB Postgres (25 conn limit)
      auto-commit: true
      idle-timeout: 300000
      max-lifetime: 1200000
      connection-timeout: 5000
      initialization-fail-timeout: 10000

  # NO sql.init block — Flyway owns schema in prod.

  jpa:
    hibernate:
      ddl-auto: ${JPA_DDL_AUTO}      # MUST be "validate" in prod env
    show-sql: ${JPA_SHOW_SQL}
    properties:
      hibernate.jdbc.lob.non_contextual_creation: true
      hibernate:
        dialect: ${JPA_HIBERNATE_DIALECT}
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true

  data:
    redis:
      host: ${DATA_REDIS_HOST}
      port: ${DATA_REDIS_PORT}
      password: ${DATA_REDIS_PASSWORD}
      timeout: ${DATA_REDIS_TIMEOUT}
      # No pool config — Lettuce uses shared multiplexed connection.
      # Pool config without commons-pool2 dependency is a no-op anyway.

  cache:
    type: redis
    redis:
      cache-null-values: false
      use-key-prefix: true
      key-prefix: "oglasino:"
      # No time-to-live: per-cache TTLs are set in code via RedisCacheConfiguration.

  elasticsearch:
    uris: ${ELASTICSEARCH_URIS}
    username: ${ELASTICSEARCH_USERNAME}
    password: ${ELASTICSEARCH_PASSWORD}
    connection-timeout: ${ELASTICSEARCH_CONNECTION_TIMEOUT}
    socket-timeout: ${ELASTICSEARCH_SOCKET_TIMEOUT}

  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  address: 0.0.0.0
  port: 8080
  shutdown: graceful
  forward-headers-strategy: framework    # honor X-Forwarded-* from Caddy
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
    min-response-size: 1024
  tomcat:
    max-threads: 100
    accept-count: 50
    connection-timeout: 20000

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: never
      probes:
        enabled: true
  health:
    livenessstate: { enabled: true }
    readinessstate: { enabled: true }
    db: { enabled: true }
    redis: { enabled: true }
    elasticsearch: { enabled: true }

cloudflare:
  r2:
    account-id: ${CLOUDFLARE_ACC_ID}
    access-key: ${CLOUDFLARE_ACCESS_KEY}
    secret-key: ${CLOUDFLARE_SECRET_KEY}
    bucket: ${CLOUDFLARE_BUCKET}
  api:
    token: ${CLOUDFLARE_API_TOKEN}
  account:
    id: ${CLOUDFLARE_ACC_ID}
  worker:
    domain: ${CLOUDFLARE_WORKER_DOMAIN}
    url: ${CLOUDFLARE_WORKER_URL}
  config:
    token: ${CLOUDFLARE_KV_CONFIG_TOKEN}
    namespace-id: ${CLOUDFLARE_NAMESPACE_ID}

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  level:
    root: INFO
    com:
      zaxxer:
        hikari: WARN
      memento:
        tech:
          oglasino: INFO
    org:
      springframework:
        web:
          _self: INFO
          filter: WARN
        servlet:
          DispatcherServlet: INFO
          mvc:
            method:
              annotation:
                RequestMappingHandlerMapping: INFO
        data:
          redis: WARN
          elasticsearch: INFO

app:
  firebase:
    credentials-path: /run/secrets/firebase.json
  internal:
    token: ${APP_INTERNAL_TOKEN}
  openai:
    api-key: ${OPENAI_API_KEY}
    api-url: ${OPENAI_API_URL}
    model: ${OPENAI_MODEL}
  views:
    flush-delay-ms: 20000
  images:
    chat:
      removal: "0 0 3 * * SUN"
  exchange:
    api-key: ${EXCHANGE_API_KEY}
  security:
    recaptcha:
      url: ${RECAPTCHA_URL}
      secret: ${RECAPTCHA_SECRET}
    allowed:
      cors: ${ALLOWED_CORS}
```

## 3.4 docker-compose.yml on backend droplet

`/opt/oglasino/docker-compose.yml`:

```yaml
name: oglasino

services:
  backend:
    image: ghcr.io/memento-tech/oglasino-backend:${IMAGE_TAG:-latest}
    restart: unless-stopped
    pull_policy: always
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DATASOURCE_URL: ${DATASOURCE_URL}
      DATASOURCE_DRIVER: ${DATASOURCE_DRIVER}
      DATASOURCE_USER: ${DATASOURCE_USER}
      DATASOURCE_PASSWORD: ${DATASOURCE_PASSWORD}
      JPA_DDL_AUTO: ${JPA_DDL_AUTO}
      JPA_SHOW_SQL: ${JPA_SHOW_SQL}
      JPA_HIBERNATE_DIALECT: ${JPA_HIBERNATE_DIALECT}
      DATA_REDIS_HOST: redis
      DATA_REDIS_PORT: "6379"
      DATA_REDIS_PASSWORD: ${DATA_REDIS_PASSWORD}
      DATA_REDIS_TIMEOUT: ${DATA_REDIS_TIMEOUT:-2000ms}
      ELASTICSEARCH_URIS: ${ELASTICSEARCH_URIS}
      ELASTICSEARCH_USERNAME: ${ELASTICSEARCH_USERNAME}
      ELASTICSEARCH_PASSWORD: ${ELASTICSEARCH_PASSWORD}
      ELASTICSEARCH_CONNECTION_TIMEOUT: ${ELASTICSEARCH_CONNECTION_TIMEOUT:-5s}
      ELASTICSEARCH_SOCKET_TIMEOUT: ${ELASTICSEARCH_SOCKET_TIMEOUT:-30s}
      CLOUDFLARE_ACC_ID: ${CLOUDFLARE_ACC_ID}
      CLOUDFLARE_ACCESS_KEY: ${CLOUDFLARE_ACCESS_KEY}
      CLOUDFLARE_SECRET_KEY: ${CLOUDFLARE_SECRET_KEY}
      CLOUDFLARE_BUCKET: ${CLOUDFLARE_BUCKET}
      CLOUDFLARE_API_TOKEN: ${CLOUDFLARE_API_TOKEN}
      CLOUDFLARE_WORKER_DOMAIN: ${CLOUDFLARE_WORKER_DOMAIN}
      CLOUDFLARE_WORKER_URL: ${CLOUDFLARE_WORKER_URL}
      CLOUDFLARE_KV_CONFIG_TOKEN: ${CLOUDFLARE_KV_CONFIG_TOKEN}
      CLOUDFLARE_NAMESPACE_ID: ${CLOUDFLARE_NAMESPACE_ID}
      APP_INTERNAL_TOKEN: ${APP_INTERNAL_TOKEN}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OPENAI_API_URL: ${OPENAI_API_URL}
      OPENAI_MODEL: ${OPENAI_MODEL}
      RECAPTCHA_URL: ${RECAPTCHA_URL}
      RECAPTCHA_SECRET: ${RECAPTCHA_SECRET}
      EXCHANGE_API_KEY: ${EXCHANGE_API_KEY}
      ALLOWED_CORS: ${ALLOWED_CORS}
    volumes:
      - /opt/oglasino/secrets/do-postgres-ca.crt:/run/secrets/do-postgres-ca.crt:ro
      - /opt/oglasino/secrets/firebase.json:/run/secrets/firebase.json:ro
    ports:
      - "127.0.0.1:8080:8080"          # Caddy reaches via host loopback
    networks: [internal]
    depends_on:
      redis: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health/liveness || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 90s
    deploy:
      resources:
        limits:
          memory: 1500M
          cpus: "1.5"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${DATA_REDIS_PASSWORD}
      --maxmemory 100mb
      --maxmemory-policy allkeys-lru
      --appendonly no
      --save ""
    networks: [internal]
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a ${DATA_REDIS_PASSWORD} --no-auth-warning ping | grep PONG"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 130M

networks:
  internal:
    driver: bridge
```

> **No postgres service.** Managed.
> **No redis volume.** Cache-only — data loss on restart is intentional.
> **`pull_policy: always`** — Compose forces registry check on every `up`, prevents stale-image bugs.

## 3.5 /opt/oglasino/.env

Generated by hand on the droplet. **Never paste this file anywhere** (chat, screenshots, support tickets). Rotate any value that ever leaves your password manager.

```bash
# === Image / deploy metadata ===
IMAGE_TAG=latest                      # CI overwrites this on each deploy

# === Managed Postgres ===
DATASOURCE_URL=jdbc:postgresql://private-cluster-do-user-XXXXX-0.b.db.ondigitalocean.com:25060/oglasino?sslmode=verify-full&sslrootcert=/run/secrets/do-postgres-ca.crt
DATASOURCE_DRIVER=org.postgresql.Driver
DATASOURCE_USER=oglasino_app
DATASOURCE_PASSWORD=<from §1.9.1>

JPA_DDL_AUTO=validate                 # NEVER "create-drop" or "update" in prod
JPA_SHOW_SQL=false
JPA_HIBERNATE_DIALECT=org.hibernate.dialect.PostgreSQLDialect

# === Redis ===
DATA_REDIS_PASSWORD=<openssl rand -base64 32>
DATA_REDIS_TIMEOUT=2000ms

# === Elasticsearch (private VPC IP of ES droplet) ===
# Use the panel-visible IP (eth1, default-fra1 VPC), NOT the eth0 private IP.
# In v5 deployment this is 10.114.0.4 (confirmed working).
ELASTICSEARCH_URIS=http://10.114.0.4:9200
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=<same as on ES droplet, set in §3.6>

# === Cloudflare R2 (images bucket) ===
CLOUDFLARE_ACC_ID=<your account ID — not a secret>
CLOUDFLARE_ACCESS_KEY=<token scoped to oglasino-images bucket only>
CLOUDFLARE_SECRET_KEY=<R2 secret>
CLOUDFLARE_BUCKET=oglasino-images-prod
CLOUDFLARE_WORKER_DOMAIN=cdn.oglasino.com
CLOUDFLARE_WORKER_URL=https://cdn.oglasino.com/

# === Cloudflare API + KV ===
CLOUDFLARE_API_TOKEN=<scoped: workers:kv edit>
CLOUDFLARE_NAMESPACE_ID=<your CONFIG namespace ID>
CLOUDFLARE_KV_CONFIG_TOKEN=<...>

# === App-internal ===
APP_INTERNAL_TOKEN=<openssl rand -base64 48>

# === OpenAI ===
OPENAI_API_KEY=<sk-proj-...>
OPENAI_API_URL=https://api.openai.com/v1/responses
OPENAI_MODEL=<your model — e.g. gpt-4o-mini>

# === reCAPTCHA ===
RECAPTCHA_URL=https://www.google.com/recaptcha/api/siteverify
RECAPTCHA_SECRET=<from Google reCAPTCHA admin>

# === Exchange rates (was hardcoded — now env) ===
EXCHANGE_API_KEY=<your key>

# === CORS ===
ALLOWED_CORS=https://oglasino.com,https://www.oglasino.com
```

```bash
chmod 600 /opt/oglasino/.env
chown igor:igor /opt/oglasino/.env
```

## 3.6 Elasticsearch droplet

On `oglasino-es`:

```bash
sudo mkdir -p /opt/oglasino-es
sudo chown igor:igor /opt/oglasino-es
cd /opt/oglasino-es
```

`/opt/oglasino-es/.env`:
```bash
ELASTIC_PASSWORD=<openssl rand -base64 32>
```

`/opt/oglasino-es/docker-compose.yml`:
```yaml
name: oglasino-es

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.0.1
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - cluster.routing.allocation.disk.threshold_enabled=false
    ports:
      - "9200:9200"        # Bound to all interfaces, but UFW restricts to backend's VPC IP
    volumes:
      - es_data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fs -u elastic:$ELASTIC_PASSWORD http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 1700M

volumes:
  es_data:
```

```bash
chmod 600 /opt/oglasino-es/.env
docker compose up -d
docker compose logs -f elasticsearch
```

After ~60 s, verify from the ES droplet:
```bash
curl -u "elastic:$(grep ELASTIC_PASSWORD .env | cut -d= -f2)" http://localhost:9200/_cluster/health
# Expect JSON with "status": "green" or "yellow"
```

From the **backend** droplet:
```bash
ES_PRIVATE_IP=10.114.0.4   # ES droplet's private VPC IP (eth1, from DO panel)
curl -u "elastic:<password>" http://$ES_PRIVATE_IP:9200/_cluster/health
```

If "Connection refused" → check ES droplet UFW (see §1.4.4 — same allowlist pattern but for port 9200). Note: as documented in v5, the UFW rule for 9200 is largely cosmetic because Docker bypasses UFW for published ports; the real protection is Docker's bind to the specific VPC interface (`10.114.0.4:9200:9200` in the ES compose file). UFW still gets the right source IP (`10.114.0.6`, the backend's `eth1`) for clean documentation.

If "Connection timed out" → likely the UFW source-IP is wrong or Docker is binding to the wrong interface. Verify backend's actual `eth1` IP with `ip -4 addr show eth1` and the ES Docker binding with `sudo ss -tlnp | grep 9200` on the ES droplet.

> **Why `ELASTICSEARCH_PASSWORD` matters in the backend `.env`:** ES 8+/9 ships with `xpack.security.enabled=true` by default. Without the username/password in Spring Boot config, `indices.exists` and other client calls fail with 401.

## 3.7 Firebase service account — secrets pattern

> **Critical:** if `firebase-service-account.json` is in `src/main/resources` of your repo, it's compromised. Rotate the key first, then move the file out, then purge git history.

### 3.7.1 Rotate the key

Firebase Console → Project Settings → Service Accounts → "Generate new private key". Download the new JSON. The old one is now revocable; revoke it once the new one works.

### 3.7.2 Move the file out of repo

```bash
# In your backend repo
mkdir -p secrets-local
mv src/main/resources/firebase-service-account.json secrets-local/
ln -s secrets-local/firebase-service-account.json firebase-service-account.json
```

### 3.7.3 .gitignore

```
firebase-service-account.json
secrets-local/
```

### 3.7.4 Purge git history

```bash
# Destructive — coordinate with anyone holding clones
pip install git-filter-repo  # or brew install
git filter-repo --path src/main/resources/firebase-service-account.json --invert-paths
git push --force-with-lease origin master
```

### 3.7.5 Update `FirebaseConfig.java`

```java
package com.memento.tech.oglasino.security.config;

import com.google.auth.oauth2.GoogleCredentials;
import com.google.firebase.FirebaseApp;
import com.google.firebase.FirebaseOptions;
import jakarta.annotation.PostConstruct;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FirebaseConfig {

  private static final Logger log = LoggerFactory.getLogger(FirebaseConfig.class);

  @Value("${app.firebase.credentials-path:}")
  private String credentialsPath;

  @PostConstruct
  public void init() throws IOException {
    if (!FirebaseApp.getApps().isEmpty()) return;

    if (credentialsPath == null || credentialsPath.isBlank()) {
      throw new IllegalStateException(
          "app.firebase.credentials-path is not configured");
    }

    Path path = Path.of(credentialsPath);
    if (!Files.exists(path) || !Files.isReadable(path)) {
      throw new IllegalStateException(
          "Firebase credentials not readable at: " + path.toAbsolutePath());
    }

    try (InputStream is = new FileInputStream(path.toFile())) {
      FirebaseOptions options = FirebaseOptions.builder()
          .setCredentials(GoogleCredentials.fromStream(is))
          .build();
      FirebaseApp.initializeApp(options);
      log.info("Firebase Admin SDK initialized from: {}", path);
    }
  }
}
```

### 3.7.6 Place the file on the droplet

```bash
# Laptop
scp secrets-local/firebase-service-account.json igor@oglasino-prod:/tmp/firebase.json

# Droplet
sudo mv /tmp/firebase.json /opt/oglasino/secrets/firebase.json
sudo chown root:root /opt/oglasino/secrets/firebase.json
sudo chmod 600 /opt/oglasino/secrets/firebase.json
```

The compose file already mounts it at `/run/secrets/firebase.json`. `application-prod.yml` references that path. Done.
---

# PHASE 4 — Cloudflare

This phase has the most moving pieces. Goal: edge-protected, maintenance-gated, rate-limited routing for both the Vercel frontend and the Caddy/Spring Boot backend, with R2 serving images via a public CDN domain.

## 4.1 DNS — six records

In Cloudflare dashboard → DNS → Records:

| # | Name | Type | Content | Proxy | Purpose |
|---|------|------|---------|-------|---------|
| 1 | `oglasino.com` | A | `<backend droplet IP>` | 🟠 Proxied | Apex — Worker routes to Vercel |
| 2 | `www` | CNAME | `oglasino.com` | 🟠 Proxied | Worker 301 → apex |
| 3 | `api` | A | `<backend droplet IP>` | 🟠 Proxied | API surface — Worker routes to Caddy |
| 4 | `api-origin` | A | `<backend droplet IP>` | ⚪ DNS only | **Gray cloud — Worker bypass to Caddy** |
| 5 | `cdn` | R2 | `oglasino-images-prod` | 🟠 Proxied | R2 connect-domain (set up via R2 UI, not DNS) |

**Do NOT add `www.api`** — multi-level subdomain wildcard certs aren't covered by Cloudflare's free SSL. The hostname has no real use case anyway.

The asymmetry that matters: `api.oglasino.com` (orange) and `api-origin.oglasino.com` (gray) point at the same droplet IP via different paths. Public traffic uses the proxied `api`; the Worker uses the gray `api-origin` to bypass its own route and avoid an infinite loop.

### 4.1.1 R2 CDN setup

The `cdn` record isn't a manual DNS entry. Go to:
- Cloudflare dashboard → R2 → click your bucket (`oglasino-images-prod`) → Settings → Public access → "Connect Domain"
- Enter `cdn.oglasino.com`, save
- Cloudflare creates the right CNAME record automatically

Verify:
```bash
curl -I https://cdn.oglasino.com/<some-test-file-key>
# Expect: 200 with image content-type, after uploading any test file
```

### 4.1.2 Verify DNS propagation

```bash
# All proxied → return Cloudflare IPs (104.x.x.x, 172.x.x.x)
dig +short oglasino.com
dig +short www.oglasino.com
dig +short api.oglasino.com
dig +short cdn.oglasino.com

# Gray-cloud → returns YOUR droplet IP directly
dig +short api-origin.oglasino.com
# Should return e.g. 157.245.22.189 (the Reserved IP — v5)
```

If `api-origin` returns Cloudflare IPs instead of your droplet IP: you forgot to gray-cloud it. Click the orange cloud icon on that record until it goes gray.

## 4.2 KV namespace

Cloudflare dashboard → Workers & Pages → KV → Create namespace:
- Name: `oglasino-config` (binding will be `CONFIG`)
- Save the namespace ID — you'll need it for the Worker and CI

Add initial keys:
- `maintenance.active` = `false`
- `use.backend.check` = `false`

## 4.3 Cloudflare Worker — `oglasino-prod-router`

Create the Worker via Cloudflare dashboard → Workers & Pages → Create → "Hello World" template. Name it `oglasino-prod-router`.

Replace the code with:

```javascript
// Oglasino production router
// Handles: oglasino.com, www.oglasino.com, api.oglasino.com
// Routes frontend to Vercel, API to Caddy via api-origin gray-cloud bypass.
// Rate-limits API traffic. Maintenance-gates everything via KV flag.

const FRONTEND_ORIGIN = "https://oglasino-frontend.vercel.app";
const BACKEND_ORIGIN  = "https://api-origin.oglasino.com";
const MAINTENANCE     = "https://oglasino-maintenance.pages.dev";

const MAINTENANCE_JSON = JSON.stringify({
  status: "maintenance",
  message: "Oglasino is undergoing maintenance. Please try again in a few minutes.",
  retryAfter: 120
});

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const host = url.hostname.toLowerCase();
    const path = url.pathname;

    // www → apex 301
    if (host === "www.oglasino.com") {
      return Response.redirect(`https://oglasino.com${path}${url.search}`, 301);
    }

    const isApi = host === "api.oglasino.com";
    const isFrontend = host === "oglasino.com";

    if (!isApi && !isFrontend) {
      return new Response("Not found", { status: 404 });
    }

    // Rate limiting — API only, before any forwarding
    if (isApi) {
      const rateLimitResp = await applyRateLimits(request, env, path);
      if (rateLimitResp) return rateLimitResp;
    }

    // Admin path bypasses maintenance (used to verify deploys before flipping flag off)
    const isAdminBypass = path.startsWith("/admin");

    // Read maintenance flag (60s edge cache to limit KV reads)
    let maintenanceActive = false;
    try {
      const v = await env.CONFIG.get("maintenance.active", { cacheTtl: 60 });
      maintenanceActive = v === "true";
    } catch (_) {
      maintenanceActive = false;  // fail open on KV errors
    }

    // Optional: backend liveness probe
    if (!maintenanceActive) {
      const useBackendCheck = await env.CONFIG.get("use.backend.check", { cacheTtl: 60 });
      if (useBackendCheck === "true") {
        try {
          const probe = await fetch(`${BACKEND_ORIGIN}/health`, {
            cf: { cacheTtl: 30, cacheEverything: true }
          });
          if (!probe.ok) maintenanceActive = true;
        } catch (_) {
          maintenanceActive = true;
        }
      }
    }

    if (maintenanceActive && !isAdminBypass) {
      return maintenanceResponse(isApi, path, url.search);
    }

    if (isApi) {
      return forwardToOrigin(BACKEND_ORIGIN, path, url.search, request);
    }
    return forwardToOrigin(FRONTEND_ORIGIN, path, url.search, request);
  }
};

async function applyRateLimits(request, env, path) {
  const clientIp = request.headers.get("CF-Connecting-IP") || "unknown";
  const method = request.method;

  // Tier 1: auth endpoints
  const isAuthBurst =
    method === "POST" &&
    (path === "/api/auth/login" || path === "/api/auth/forgot-password");
  if (isAuthBurst) {
    const { success } = await env.AUTH_LIMITER.limit({ key: clientIp });
    if (!success) {
      return rateLimitedResponse("Too many authentication attempts.", 60);
    }
  }

  // Tier 2: sensitive writes
  const isSensitiveWrite =
    method === "POST" &&
    (path === "/api/auth/register" || path === "/api/listings");
  if (isSensitiveWrite) {
    const { success } = await env.WRITE_LIMITER.limit({ key: clientIp });
    if (!success) {
      return rateLimitedResponse("Too many requests. Please slow down.", 60);
    }
  }

  // Tier 3: global API ceiling
  const { success: g } = await env.GLOBAL_LIMITER.limit({ key: clientIp });
  if (!g) {
    return rateLimitedResponse("Too many requests. Slow down.", 60);
  }

  return null;
}

function rateLimitedResponse(message, retryAfterSeconds) {
  return new Response(
    JSON.stringify({ error: "rate_limit_exceeded", message, retryAfter: retryAfterSeconds }),
    {
      status: 429,
      headers: {
        "Content-Type": "application/json; charset=utf-8",
        "Retry-After": String(retryAfterSeconds),
        "Cache-Control": "no-store"
      }
    }
  );
}

function maintenanceResponse(isApi, path, search) {
  if (isApi) {
    return new Response(MAINTENANCE_JSON, {
      status: 503,
      headers: {
        "Content-Type": "application/json; charset=utf-8",
        "Retry-After": "120",
        "Cache-Control": "no-store",
        "X-Oglasino-Maintenance": "true"
      }
    });
  }

  return fetch(`${MAINTENANCE}${path}${search}`).then(upstream => {
    const headers = new Headers(upstream.headers);
    headers.set("X-Oglasino-Maintenance", "true");
    headers.set("Cache-Control", "no-store");
    headers.set("Retry-After", "120");
    return new Response(upstream.body, {
      status: 503,
      statusText: "Service Unavailable",
      headers
    });
  });
}

async function forwardToOrigin(originBase, path, search, originalRequest) {
  const targetUrl = `${originBase}${path}${search}`;

  // Preserve original headers; forward Host so Next.js generates correct canonical URLs
  const newHeaders = new Headers(originalRequest.headers);
  const originalHost = originalRequest.headers.get("Host") || "";
  if (originalHost) {
    newHeaders.set("X-Forwarded-Host", originalHost);
    newHeaders.set("X-Forwarded-Proto", "https");
  }

  const forwarded = new Request(targetUrl, {
    method: originalRequest.method,
    headers: newHeaders,
    body: originalRequest.body,
    redirect: "manual"
  });

  return fetch(forwarded);
}
```

Save and Deploy.

## 4.4 Worker bindings (in dashboard)

Settings → Bindings → Add:

**KV namespace binding:**
- Variable name: `CONFIG`
- Namespace: select your `oglasino-config` namespace

**Rate limiting bindings (3):**

| Variable | Limit | Period |
|---|---|---|
| `AUTH_LIMITER` | 5 | 60s |
| `WRITE_LIMITER` | 10 | 60s |
| `GLOBAL_LIMITER` | 600 | 60s |

> Free tier caps periods at 60s — that's why we don't have hourly windows. For longer windows, Bucket4j in Spring Boot (Phase 6 task) handles per-user-id limits.

## 4.5 Worker routes

Settings → Triggers → Routes → Add:

```
oglasino.com/*           Zone: oglasino.com
www.oglasino.com/*       Zone: oglasino.com
api.oglasino.com/*       Zone: oglasino.com
```

> **Don't add `api-origin.oglasino.com/*`.** That's the gray-cloud bypass — the Worker must NOT intercept its own backend calls.

## 4.6 Test the Worker

Manual maintenance flip via Cloudflare API:

```bash
# Flip ON
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/storage/kv/namespaces/<KV_NAMESPACE_ID>/values/maintenance.active" \
  -H "Authorization: Bearer <CF_API_TOKEN>" \
  -H "Content-Type: text/plain" \
  --data "true"

sleep 60

curl -I https://api.oglasino.com/health
# Expect: HTTP/2 503, X-Oglasino-Maintenance: true

curl -I https://oglasino.com/
# Expect: HTTP/2 503, served from oglasino-maintenance.pages.dev with X-Oglasino-Maintenance: true

# Flip OFF
curl -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/storage/kv/namespaces/<KV_NAMESPACE_ID>/values/maintenance.active" \
  -H "Authorization: Bearer <CF_API_TOKEN>" \
  -H "Content-Type: text/plain" \
  --data "false"

sleep 60
curl -I https://api.oglasino.com/health
# Expect: not 503 anymore (502 if Spring Boot not deployed yet, that's fine)
```

Rate limit test:
```bash
for i in 1 2 3 4 5 6 7; do
  CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST https://api.oglasino.com/api/auth/login \
    -H "Content-Type: application/json" -d '{"email":"a","password":"b"}')
  echo "Request $i: HTTP $CODE"
done
# Expect: ~5 of (400/401/whatever auth returns), then 429s
```

## 4.7 Maintenance toggle scripts (on droplet)

`/opt/oglasino/.env.cf`:
```bash
CF_API_TOKEN=<same as in GitHub secrets — scoped to KV edit>
CF_ACCOUNT_ID=<your account ID>
CF_KV_NAMESPACE_ID=<your CONFIG namespace ID>
```

```bash
chmod 600 /opt/oglasino/.env.cf
```

`/opt/oglasino/scripts/maintenance-on.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
source /opt/oglasino/.env.cf

curl -fsSL -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: text/plain" \
  --data "true"

echo "✅ Maintenance ON. Allow ~60s for global propagation."
```

`/opt/oglasino/scripts/maintenance-off.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
source /opt/oglasino/.env.cf

curl -fsSL -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: text/plain" \
  --data "false"

echo "✅ Maintenance OFF. Allow ~60s for traffic to flow normally."
```

```bash
chmod +x /opt/oglasino/scripts/maintenance-*.sh
```

## 4.8 Cloudflare security settings (dashboard)

These are one-click toggles. Each takes <1 minute. Together they're significant security upgrades.

### SSL/TLS → Overview
- **Encryption mode:** Full (strict)

### SSL/TLS → Edge Certificates
- **Always Use HTTPS:** ON
- **Minimum TLS Version:** TLS 1.2
- **TLS 1.3:** ON
- **Automatic HTTPS Rewrites:** ON

### SSL/TLS → Edge Certificates → HSTS
Conservative initial setup:
- **Enable HSTS:** ON
- **Max Age:** 1 month
- **Apply to subdomains:** OFF (turn on after 6+ months stable)
- **Preload:** OFF (never until after subdomains is on for 6+ months)
- **No-Sniff Header:** ON

> HSTS rollout: 1 month now → 6 months after 3 months stable → 12 months → enable subdomains → enable preload. Each step is harder to reverse than the last. Don't skip ahead.

### Security → WAF → Managed Rules
- Enable **Cloudflare Managed Ruleset**
- **Action: Log** initially (one week observation)
- Switch to **Block** after a week of clean events

### Security → Bots
- **Bot Fight Mode:** ON

### Security → Settings
- **Security Level:** Medium
- **Browser Integrity Check:** ON
- **Challenge Passage:** 30 minutes

### Caching → Cache Rules
Two rules:

**Rule 1 — API never cached:**
```
When: Hostname equals "api.oglasino.com"
Then: Cache eligibility: Bypass cache
```

**Rule 2 — CDN images aggressively cached:**
```
When: Hostname equals "cdn.oglasino.com"
Then:
  Cache eligibility: Eligible for cache
  Edge TTL: Override origin: 1 month
  Browser TTL: Override origin: 1 month
```

> **Don't add a maintenance page rule.** The Worker sets `Cache-Control: no-store` directly, which overrides any cache rule.

## 4.9 UFW Cloudflare-only lockdown — DO LAST

> **Don't run this until everything else is end-to-end working.** The moment you tighten UFW, anything reaching 443 from outside Cloudflare gets dropped — including manual curl tests. Verify: Worker → Caddy → Spring Boot path through Cloudflare returns 200 BEFORE locking down.

> **v5 note: this section was executed on both `oglasino-prod` (May 15, 2026) and `oglasino-stage` (same day).** Both droplets are now Cloudflare-source-IP-only on 80/443. The script below is the **patched** version that ships in v5 — it fixes a first-run bug in the v4 script where `set -e` caused premature exit on the leading `grep 'Cloudflare'` when no Cloudflare-tagged rules existed yet. The v4 script silently did nothing on a fresh box.

`/opt/oglasino/scripts/update-ufw-cloudflare.sh` (v5 patched version):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Remove existing CF-tagged rules (tolerate "no matches" on first run).
# The { ... } || true wrapper prevents set -e from exiting when grep
# finds nothing (first run has no Cloudflare-tagged rules to clean).
# xargs -r is --no-run-if-empty: don't invoke ufw delete with no args.
{
  sudo ufw status numbered | grep 'Cloudflare' | \
    awk -F'[][]' '{print $2}' | sort -rn | \
    xargs -r -I {} sudo ufw --force delete {}
} || true

# Allow only Cloudflare IP ranges on 443 (and 80 for cert renewals)
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  sudo ufw allow from "$ip" to any port 443 proto tcp comment 'Cloudflare'
  sudo ufw allow from "$ip" to any port 80  proto tcp comment 'Cloudflare'
done
for ip in $(curl -s https://www.cloudflare.com/ips-v6); do
  sudo ufw allow from "$ip" to any port 443 proto tcp comment 'Cloudflare'
  sudo ufw allow from "$ip" to any port 80  proto tcp comment 'Cloudflare'
done

# Remove broad allow-all
sudo ufw delete allow 80/tcp 2>/dev/null || true
sudo ufw delete allow 443/tcp 2>/dev/null || true
sudo ufw reload
echo "UFW updated to Cloudflare-only on 80/443."
```

```bash
chmod +x /opt/oglasino/scripts/update-ufw-cloudflare.sh
```

**Deployment sequence (mandatory order):**

1. **Pre-flight: verify the proxied path works through Cloudflare.** From your laptop: `curl -I https://api.oglasino.com/health` must return 200 *before* you touch UFW. Caddy and Spring Boot must be healthy. Otherwise you'll lock down a broken setup and waste time blaming the lockdown.
2. **Open a second SSH session as a safety net.** Different terminal tab. Leave it idle.
3. **Run the script manually first** — `sudo /opt/oglasino/scripts/update-ufw-cloudflare.sh`. Expect ~44 `Rule added` lines + 4 `Rule deleted` + `Firewall reloaded` + the final success message. If you get a clean prompt with no output, you're running the v4 script and the first-run bug bit you. Re-check the script contents.
4. **Verify UFW state**: `sudo ufw status verbose`. Should show ~22 Cloudflare CIDR rules on 80 and 443, SSH on 2202 still `ALLOW Anywhere`, and *no* broad `80/tcp ALLOW Anywhere` or `443/tcp ALLOW Anywhere` rules.
5. **Verify normal traffic still works**: `curl -I https://api.oglasino.com/health` → 200.
6. **Verify direct-origin is blocked**: `curl --max-time 5 -k https://YOUR_DROPLET_IP` from your laptop → connection timed out (correct).
7. **Only then schedule daily via cron.** Cron alone does nothing on a fresh box — the script has to run at least once to install the rules, and the cron keeps them fresh.

Schedule daily via cron:
```bash
sudo crontab -e
# Add:
0 4 * * * /opt/oglasino/scripts/update-ufw-cloudflare.sh >> /var/log/ufw-cf.log 2>&1
```

> **v5 note: schedule is daily, not weekly.** v4 originally specified Monday-only (`0 4 * * 1`). Cloudflare's IP list rarely changes but daily costs nothing measurable and reduces drift window. Both prod and stage are on daily.

This is the **most important** network-layer security control. Without it, your origin IP is reachable directly, bypassing all Cloudflare protection. With it, only Cloudflare can reach 443 — every WAF rule, rate limit, and bot protection becomes the only path.

> **The `api-origin*.oglasino.com` "exposed origin" warning** in your DNS records is correct: those hostnames leak your origin IP (they're gray-cloud, DNS-only — they have to expose the real IP for the Worker to reach the origin). The mitigation is exactly this UFW rule. Anyone who connects to your droplet's IP from a non-Cloudflare source gets dropped. The Worker's connection to `api-origin` works because Workers run on Cloudflare's network — their source IPs are in the allowlist. The v5 Reserved IP migration means even if the anchor IP leaks via some old certificate transparency log or DNS history, the Reserved IP can be detached and replaced without rebuilding the droplet.

### Live infrastructure IPs (post-v5 migration)

| Droplet | Public IP (current) | Anchor IP (historical, no longer in DNS) | VPC IP (`eth1`, default-fra1) |
|---|---|---|---|
| `oglasino-prod` | `157.245.22.189` (Reserved) | `46.101.166.120` | `10.114.0.6` |
| `oglasino-stage` | `139.59.204.182` (Reserved) | `68.183.211.5` | `10.114.0.x` |
| `oglasino-elasticsearch` | `167.172.106.33` (anchor — no Reserved IP, ES is not directly public anyway) | — | `10.114.0.4` |

> The anchor IPs still resolve to the same droplets at the OS level (they're permanent NIC addresses), but **no Cloudflare DNS record references them anymore**. Bots using leaked-anchor-IP databases get UFW-dropped on connect.

---

# PHASE 5 — Frontend (Next.js 15 App Router on Vercel)

The frontend lives in a separate repo (e.g., `oglasino-frontend` or `oglasino-web`). Deployed to Vercel, fronted by your Cloudflare Worker.

## 5.1 Why Vercel

Next.js 15 SSR with App Router needs full Node.js runtime at request time. Cloudflare Pages forces Edge Runtime (limited Node API surface). Vercel runs Node natively. For a Serbia-first marketplace where SSR + SEO matter and global edge isn't critical, Vercel's free tier is the right fit.

When you might revisit:
- 100GB bandwidth/month is the free tier ceiling — you'll know if/when you hit it
- If you're committed to a single-vendor (Cloudflare) stack ideologically, Cloudflare Pages with `@cloudflare/next-on-pages` is the alternative

For now: Vercel.

## 5.2 Vercel project setup

- Sign up at vercel.com with the GitHub account that owns the frontend repo
- Add New → Project → import your Next.js repo
- Framework auto-detected: Next.js
- **Don't deploy yet** — set env vars first (next section)
- Click Deploy

After first build (~3-4 min), you have a `.vercel.app` URL.

### Production branch — phantom branch trick

Vercel's UI requires the production branch to exist. Create a `production` branch that you'll never push to:

```bash
# Laptop, in frontend repo
git checkout main
git pull origin main
git checkout -b production
git push -u origin production
```

Then in Vercel project → Settings → Environments → Production:
- Branch tracking: change `main` to `production`
- Save

This stops Vercel auto-deploying when you merge to `main`. CI handles production deploys instead. Vercel still auto-deploys `dev` and feature branches as preview URLs (good — preview is useful).

### vercel.json

In the frontend repo root:

```json
{
  "framework": "nextjs",
  "regions": ["fra1"],
  "git": {
    "deploymentEnabled": {
      "main": false
    }
  }
}
```

`fra1` (Frankfurt) is closer to Belgrade than Vercel's default `iad1` — lower latency for Serbian users.

`deploymentEnabled.main: false` is belt-and-suspenders with the production branch trick: even if the UI setting changes, this JSON declaratively prevents auto-deploy.

### Add domain

Vercel project → Settings → Domains → Add `oglasino.com`. Vercel will show "Invalid Configuration" — that's expected. You're not pointing DNS at Vercel; the Worker proxies traffic. The domain just needs to be associated with the project so Next.js generates correct canonical URLs.

## 5.3 Environment variables (Vercel)

Per-environment in Vercel project → Settings → Environment Variables. Production:

```bash
# Public — bundled into client JS
NEXT_PUBLIC_API_URL=https://api.oglasino.com/api
NEXT_PUBLIC_CDN_URL=https://cdn.oglasino.com
NEXT_PUBLIC_RECAPTCHA_SITE_KEY=<prod reCAPTCHA site key>
NEXT_PUBLIC_FIREBASE_API_KEY=<from Firebase console>
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=<your-project>.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=<your prod project id>
NEXT_PUBLIC_FIREBASE_APP_ID=1:xxx:web:xxx
NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID=G-XXXXXXXXXX     # NOT MEASURMENT (typo)
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=<your-project>.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=<numeric>
NEXT_PUBLIC_FIREBASE_VAPID_KEY=<from Firebase Cloud Messaging > Web Push certs>

# Server-only — never sent to browser
FIREBASE_PROJECT_ID=<prod project id>
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxx@<your-project>.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY=-----BEGIN PRIVATE KEY-----\nMIIEvQ...\n-----END PRIVATE KEY-----\n

# Reduce telemetry noise
NEXT_TELEMETRY_DISABLED=1
```

> **The `\n` in `FIREBASE_PRIVATE_KEY` is literal backslash-n, not a real newline.** Vercel's input is single-line; the runtime code does `replace(/\\n/g, '\n')` to convert. Don't try to paste real multi-line content.

> **All `NEXT_PUBLIC_FIREBASE_*` values are intentionally public** — Firebase's security model relies on Security Rules, not key secrecy. The service account JSON (`FIREBASE_PRIVATE_KEY`) is what's actually secret.

## 5.4 next.config.ts

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.oglasino.com',
        pathname: '/**',
      },
    ],
  },

  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
        ],
      },
    ];
  },
};

export default nextConfig;
```

The `images.remotePatterns` is required for Next's `<Image>` component to load from `cdn.oglasino.com`. Without it, image rendering errors with "hostname not configured."

## 5.5 Branch model (frontend)

```
feature/* ──▶ dev ──▶ main
                │      │
                │      └─▶ CI runs deploy-prod.yml → Vercel production
                │
                └──▶ Vercel auto-deploys preview URL (test here before merging to main)

production branch:
  Created once, never updated. Exists only for Vercel UI validation.
```

Three protected branches:
- `dev` — integration; all features merge here first
- `main` — production code; only updated via PR from dev
- `production` — phantom; nothing touches it

## 5.6 GitHub branch protection (frontend repo)

Settings → Branches → Add ruleset:

**Protect `main`:**
- Require PR before merging
- Required approvals: 1
- Require status checks to pass: `lint`, `typecheck`, `format-check`, `build`
- Require linear history: ON
- Do not allow bypassing: ON

**Protect `dev`:**
- Require status checks to pass for any PRs

**Protect `production`:**
- Restrict pushes that create matching files
- Do not allow bypassing: ON
- (Branch is effectively write-locked)

---

# PHASE 6 — CI/CD

Three workflows total: backend deploy on push to main, frontend CI on push to dev, frontend deploy on push to main.

## 6.1 GitHub secrets (both repos)

In each repo → Settings → Secrets and variables → Actions:

**Backend repo:**
| Secret | Value |
|---|---|
| `PROD_HOST` | `157.245.22.189` (the Reserved IP) |
| `PROD_SSH_PORT` | `2202` |
| `DEPLOY_SSH_KEY` | Private SSH key for the `deploy` user (see §6.5) |
| `CF_API_TOKEN` | Cloudflare API token, scoped to KV edit |
| `CF_ACCOUNT_ID` | From Cloudflare dashboard sidebar |
| `CF_KV_NAMESPACE_ID` | Your `oglasino-config` namespace ID |

**Frontend repo:**
| Secret | Value |
|---|---|
| `VERCEL_TOKEN` | From Vercel → Account Settings → Tokens |
| `CF_API_TOKEN` | Same as backend |
| `CF_ACCOUNT_ID` | Same as backend |
| `CF_KV_NAMESPACE_ID` | Same as backend |

## 6.2 Backend CI/CD workflow

`.github/workflows/deploy-backend.yml`:

```yaml
name: Deploy Backend

on:
  push:
    branches: [main]
  workflow_dispatch: {}

concurrency:
  group: deploy-backend
  cancel-in-progress: false

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4

      - id: meta
        run: |
          TAG="$(date -u +%Y%m%dT%H%M)-${GITHUB_SHA::7}"
          echo "tag=${TAG}" >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}
            ${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  maintenance-on:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Set maintenance.active = true
        env:
          CF_API_TOKEN:        ${{ secrets.CF_API_TOKEN }}
          CF_ACCOUNT_ID:       ${{ secrets.CF_ACCOUNT_ID }}
          CF_KV_NAMESPACE_ID:  ${{ secrets.CF_KV_NAMESPACE_ID }}
        run: |
          set -euo pipefail
          curl -fsSL -X PUT \
            "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active" \
            -H "Authorization: Bearer ${CF_API_TOKEN}" \
            -H "Content-Type: text/plain" \
            --data "true"

      - name: Wait for KV propagation
        run: sleep 90

      - name: Verify maintenance is live
        run: |
          set -e
          for attempt in 1 2 3 4 5; do
            HDR=$(curl -fsSI https://api.oglasino.com/health 2>/dev/null \
                  | tr -d '\r' \
                  | awk -F': *' 'tolower($1)=="x-oglasino-maintenance"{print $2}')
            if [ "$HDR" = "true" ]; then
              echo "✅ Confirmed at edge"
              exit 0
            fi
            sleep 30
          done
          echo "::warning::Maintenance not visible after 5 attempts — proceeding."

  deploy:
    needs: [build, maintenance-on]
    runs-on: ubuntu-latest
    steps:
      - name: SSH deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host:     ${{ secrets.PROD_HOST }}
          port:     ${{ secrets.PROD_SSH_PORT }}
          username: deploy
          key:      ${{ secrets.DEPLOY_SSH_KEY }}
          envs:     IMAGE_TAG
          script: |
            set -euo pipefail
            cd /opt/oglasino
            sed -i "s|^IMAGE_TAG=.*|IMAGE_TAG=${IMAGE_TAG}|" .env
            docker compose pull backend
            docker compose up -d --no-deps --wait backend
            docker image prune -f
            echo "Backend ${IMAGE_TAG} deployed."
        env:
          IMAGE_TAG: ${{ needs.build.outputs.image_tag }}

  notify:
    needs: deploy
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          if [ "${{ needs.deploy.result }}" = "success" ]; then
            echo "::notice::✅ Backend deployed. Maintenance STILL ON."
            echo "::notice::Verify, then run /opt/oglasino/scripts/maintenance-off.sh"
          else
            echo "::error::❌ Deploy FAILED. Maintenance still ON."
          fi
```

Maintenance stays ON after deploy. You verify the new build is healthy, then manually flip OFF via the script on the droplet.

## 6.3 Frontend CI workflow (dev)

`.github/workflows/ci-dev.yml`:

```yaml
name: CI - Dev Branch

on:
  push:
    branches: [dev]
  pull_request:
    branches: [dev, main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npx tsc --noEmit

  format-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npm run format:check

  build:
    runs-on: ubuntu-latest
    env:
      # Placeholder values — real ones come from Vercel at deploy time.
      # Build needs them so Next doesn't fail on missing env.
      NEXT_PUBLIC_API_URL: https://api.oglasino.com/api
      NEXT_PUBLIC_CDN_URL: https://cdn.oglasino.com
      NEXT_PUBLIC_RECAPTCHA_SITE_KEY: dummy
      NEXT_PUBLIC_FIREBASE_API_KEY: dummy
      NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN: dummy.firebaseapp.com
      NEXT_PUBLIC_FIREBASE_PROJECT_ID: dummy
      NEXT_PUBLIC_FIREBASE_APP_ID: dummy
      NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID: dummy
      NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET: dummy.appspot.com
      NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID: dummy
      NEXT_PUBLIC_FIREBASE_VAPID_KEY: dummy
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

> **Next.js 15 removed `next lint`.** Use `eslint .` directly. `package.json` lint script:
> ```json
> "scripts": {
>   "lint": "eslint .",
>   "format:check": "prettier --check .",
>   "format": "prettier --write ."
> }
> ```

## 6.4 Frontend deploy workflow (main)

`.github/workflows/deploy-frontend.yml`:

```yaml
name: Deploy Frontend (Production)

on:
  push:
    branches: [main]
  workflow_dispatch: {}

concurrency:
  group: deploy-frontend
  cancel-in-progress: false

jobs:
  pre-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit

  maintenance-on:
    needs: pre-checks
    runs-on: ubuntu-latest
    steps:
      - name: Set maintenance.active = true
        env:
          CF_API_TOKEN:        ${{ secrets.CF_API_TOKEN }}
          CF_ACCOUNT_ID:       ${{ secrets.CF_ACCOUNT_ID }}
          CF_KV_NAMESPACE_ID:  ${{ secrets.CF_KV_NAMESPACE_ID }}
        run: |
          set -euo pipefail
          curl -fsSL -X PUT \
            "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active" \
            -H "Authorization: Bearer ${CF_API_TOKEN}" \
            -H "Content-Type: text/plain" \
            --data "true"

      - name: Wait for KV propagation
        run: sleep 90

      - name: Verify maintenance is live
        run: |
          set -e
          for attempt in 1 2 3 4 5; do
            HDR=$(curl -fsSI https://oglasino.com/ 2>/dev/null \
                  | tr -d '\r' \
                  | awk -F': *' 'tolower($1)=="x-oglasino-maintenance"{print $2}')
            if [ "$HDR" = "true" ]; then
              echo "✅ Confirmed"
              exit 0
            fi
            sleep 30
          done
          echo "::warning::Not visible after 5 attempts — proceeding."

  deploy:
    needs: maintenance-on
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - name: Install Vercel CLI
        run: npm install -g vercel@latest

      - name: Pull Vercel production env
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}

      - name: Build for production
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}

      - name: Deploy to Vercel production
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}

  notify:
    needs: deploy
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          if [ "${{ needs.deploy.result }}" = "success" ]; then
            echo "::notice::✅ Frontend deployed. Maintenance STILL ON."
            echo "::notice::Verify, then flip OFF via maintenance-off.sh on droplet."
          else
            echo "::error::❌ Deploy FAILED. Maintenance still ON."
          fi
```

Same maintenance-stays-on-after-deploy pattern as backend. Same KV flag, same script flips both back.

## 6.5 Deploy user on droplet

Create a non-interactive deploy user that CI uses for SSH:

```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy

# Set up SSH access
sudo mkdir -p /home/deploy/.ssh
sudo chown deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh

# Generate a dedicated keypair on your laptop:
# ssh-keygen -t ed25519 -f ~/oglasino-ci-key -N ""
# Then paste the public key (~/oglasino-ci-key.pub) below:
sudo bash -c 'cat >> /home/deploy/.ssh/authorized_keys' <<'EOF'
ssh-ed25519 AAAAC3... deploy@oglasino-ci
EOF
sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys

# Give deploy user ownership of /opt/oglasino so it can edit .env and run compose
sudo chown -R deploy:deploy /opt/oglasino
```

The private key (`~/oglasino-ci-key`) goes into the GitHub `DEPLOY_SSH_KEY` secret in the backend repo. Save the file in your password manager too.

Test from your laptop:
```bash
ssh -p 2202 -i ~/oglasino-ci-key deploy@<droplet-ip>
docker ps                # should work, no permission error
exit
```

Add `deploy` to `AllowUsers` in `/etc/ssh/sshd_config` if you haven't already (§1.4.2).

## 6.6 First deploy — verification chain

Before merging to `main` for the first real deploy, sanity-check:

```bash
# 1. Can deploy SSH?
ssh -p 2202 -i ~/oglasino-ci-key deploy@<droplet-ip>

# 2. Can deploy run docker?
docker ps

# 3. Can deploy write to /opt/oglasino?
touch /opt/oglasino/test && rm /opt/oglasino/test

# 4. Has GHCR auth been done?
docker pull ghcr.io/memento-tech/oglasino-backend:latest
# Either succeeds (after first CI build) or fails with "manifest unknown"
# (no images pushed yet — fine on bootstrap)

# 5. Can the droplet reach the maintenance Worker?
curl -I https://api.oglasino.com/health
# Some response — confirms DNS+proxy work

# 6. Manual maintenance flip works?
/opt/oglasino/scripts/maintenance-on.sh
sleep 60
curl -I https://api.oglasino.com/health  # → 503
/opt/oglasino/scripts/maintenance-off.sh
sleep 60
curl -I https://api.oglasino.com/health  # → not 503
```

If all six pass: push to `main`, watch the workflow run, you have CI/CD.
---

# PHASE 7 — Database, Scaling, and Backups

## 7.1 Flyway migrations

Drop `spring.sql.init` from `application-local.yml` (the SQL files in `data/system/`, `data/core/` etc.). In prod, Flyway owns schema. Reference data becomes Flyway migrations.

Add to `pom.xml`:
```xml
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

Move SQL files to `src/main/resources/db/migration/` with versioned names:
- `V1__init_schema.sql`
- `V2__add_translations.sql`
- `V3__add_locations.sql`
- etc.

Spring Boot auto-runs Flyway on startup. With `JPA_DDL_AUTO=validate`, Hibernate verifies the schema matches your entity definitions but never modifies it. Flyway is the only thing that touches schema.

For your existing seed data (translations, base sites, etc.), convert to idempotent inserts using `INSERT ... ON CONFLICT DO NOTHING`. That way re-running the migration is safe.

## 7.2 Postgres tuning (managed cluster)

DO Basic 1GB plan defaults are reasonable. Things to know:

- **25 connection limit.** Hikari pool maxes at 12; one Spring Boot instance is fine.
- **No knobs for `shared_buffers`, `work_mem`** etc. on Basic. Auto-managed by DO.
- **PgBouncer:** available on $25+/mo plans. Worth it once you have 2+ backend instances. For now, direct connections.

## 7.3 When to upgrade Postgres

| Trigger | Action |
|---|---|
| First paying customer arrives | Upgrade to Pro HA (~$50/mo) — gives standby replica + auto-failover |
| Slow queries appearing in logs | Upgrade plan size; check for missing indexes first |
| Connection pool exhaustion | Add PgBouncer (Pro plan) before adding more app instances |

## 7.4 Backup strategy

**DO already provides** (Basic plan):
- Daily automatic backups (7-day retention)
- Point-in-time recovery within 7 days

**That's enough for an unlaunched app.**

Add the belt-and-suspenders R2 backup once you have real users. Skip until then. Section 3.5 from v3 covered this; here's the short version:

When ready:
1. Create separate R2 bucket: `oglasino-db-backups` (private, no public access)
2. Create a scoped R2 token with read+write to backups bucket only
3. Generate `age` keypair locally; store private key in password manager + paper backup
4. Backup script encrypts dumps with `age` public key before upload to R2
5. Cron weekly Sunday 03:00 UTC
6. Test restore quarterly

Trigger to set this up: first real user data hits prod. Before then, DO's automatic backups cover you.

## 7.5 Monitoring (post-launch priorities)

Order to add, when:

1. **Sentry** (week 1 post-launch). Free tier covers solo dev. Catch errors with stack traces and breadcrumbs.
2. **UptimeRobot or Pingdom** (week 1). Hit `/health` every 60s from external network. Alerts to your email/SMS if down.
3. **Cloudflare Web Analytics** (free, no cookies). Pageview metrics without GDPR overhead.
4. **DigitalOcean monitoring alerts** (already configured in §1.7) — CPU, RAM, disk at 80%.
5. **Grafana + Prometheus** (eventually). Spring Boot Actuator exposes `/actuator/prometheus`. Add when you need real dashboards. Not before.

---

# PHASE 8 — Day-to-Day Operations

## 8.1 Daily flow

**Backend dev:**
```bash
# Sit down to code
docker compose -f docker-compose.local.yml up -d
# IntelliJ → green play → app runs natively, connects to Docker deps

# Done for the day
docker compose -f docker-compose.local.yml down   # optional
```

**Frontend dev:**
```bash
cd ~/path/to/oglasino-frontend
npm run dev
# http://localhost:3000
```

**Branch hygiene (both repos):**
```bash
# Always work on a feature branch off dev
git checkout dev
git pull origin dev
git checkout -b feature/something

# Push, open PR to dev, CI runs
git push origin feature/something

# After PR merges to dev → Vercel auto-deploys preview URL (frontend)
# Test at preview URL

# When ready to ship:
# Open PR: dev → main
# CI runs again as defense
# Merge → production deploy workflow triggers
```

## 8.2 Production deploy

After your CI workflow finishes a deploy:
1. **Maintenance is still ON** (intentional)
2. Verify the new build via `/admin` path bypass or by checking droplet health directly
3. Flip OFF on the droplet:

```bash
ssh oglasino-prod
/opt/oglasino/scripts/maintenance-off.sh
```

Wait ~60s for KV propagation, traffic flows.

## 8.3 Rollback

If a deploy is broken, rollback is a 30-second operation:

```bash
ssh oglasino-prod
cd /opt/oglasino

# Find a previous tag in GHCR
docker images ghcr.io/memento-tech/oglasino-backend --format "{{.Tag}}\t{{.CreatedAt}}" | head -10

# Set that tag as the deployed version
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=20260504T1430-abc1234/' .env
docker compose pull backend
docker compose up -d --no-deps --wait backend

# Once verified working:
/opt/oglasino/scripts/maintenance-off.sh
```

GHCR retains all your past tagged images. The previous version is always one command away.

## 8.4 Common operations

**Restart backend (trigger reload of `.env` changes):**
```bash
cd /opt/oglasino
docker compose up -d --no-deps --force-recreate backend
```

**Tail backend logs:**
```bash
docker compose logs -f --tail=100 backend
```

**Get into Spring Boot container:**
```bash
docker compose exec backend sh
```

**Run a one-off psql against managed Postgres:**
```bash
PGPASSWORD='<password>' psql \
  "host=<private-host> port=25060 dbname=oglasino user=oglasino_app sslmode=verify-full sslrootcert=/opt/oglasino/secrets/do-postgres-ca.crt"
```

**Check disk usage:**
```bash
df -h
docker system df
```

Periodic cleanup:
```bash
docker system prune -af --filter "until=72h"   # remove images > 3 days old
```

## 8.5 Disaster recovery scenarios

**Scenario: backend droplet hard-fails.**

1. DO panel → Droplets → spin up new droplet from latest weekly backup (~10 min)
2. Update DNS A records for `oglasino.com`, `api.oglasino.com`, `api-origin.oglasino.com` to new IP
3. Wait for DNS propagation (~5 min with low TTL)
4. SSH in, verify `/opt/oglasino/.env` is intact, run `docker compose up -d`
5. Site comes back

**Scenario: managed Postgres data corruption.**

1. DO panel → Databases → your cluster → Backups
2. Choose timestamp → "Restore from backup"
3. DO creates a new cluster from the backup (~15 min)
4. Update `DATASOURCE_URL` in `/opt/oglasino/.env` to point to new cluster's private host
5. `docker compose up -d --force-recreate backend`

**Scenario: ES indexes corrupted.**

Cheapest possible recovery: rebuild from Postgres source of truth.

```bash
ssh oglasino-es
cd /opt/oglasino-es
docker compose down -v       # wipes data
docker compose up -d
```

In your Spring Boot app, trigger a reindex via `/admin/reindex` or whatever endpoint you wire up. Indexes regenerated from Postgres rows.

**Scenario: Cloudflare account compromised or locked.**

This is what UFW Cloudflare-only protects against — but only for inbound traffic. If your CF account itself is compromised, attacker could redirect DNS or alter the Worker. Mitigation:

- **2FA on Cloudflare account** (mandatory, do today)
- **CF account in separate password manager vault** with strong unique password
- **No automation tokens with full account access** — every token scoped to specific resources

For now, prevention only. Recovery is manual: contact CF support, prove ownership, regain access.

## 8.6 What you've explicitly deferred (and when to revisit)

| Item | Defer to |
|---|---|
| Belt-and-suspenders DB backups to R2 | Day after first real user data |
| UFW Cloudflare-only lockdown | Once Worker → Caddy → Spring Boot path verified end-to-end |
| Bucket4j per-user-id rate limits | When abuse patterns from Cloudflare-layer rate limiting prove insufficient |
| Sentry integration | Week 1 post-launch |
| Migrate Worker to Wrangler (git-versioned) | Post-launch when you have time for tooling |
| Upgrade backend droplet 2GB → 4GB | When swap usage stays >200 MB |
| Upgrade Postgres Basic → Pro HA | Day first paying customer arrives |
| WAF Managed Rules: Log → Block | After 1 week of clean logs |
| HSTS subdomain inclusion | After 6 months of stable HTTPS-only operation |
| HSTS preload | Never, until subdomain inclusion has been on 6+ months |
| Cloudflare Pro upgrade ($20/mo) | When free tier rate limits and bandwidth become limiting |
| Reintroduce a staging environment | When you have paying users (broken prod stops being acceptable) |

## 8.7 Pre-launch checklist

When you're ready to flip from "testing on prod with maintenance gate" to "real users":

- [ ] All env secrets rotated (any value that's ever been in chat or git history)
- [ ] Firebase service account file rotated and out of git
- [ ] WAF Managed Rules switched from Log to Block
- [ ] UFW Cloudflare-only rule active
- [ ] Maintenance flow tested end-to-end (ON → off via script → ON via script → off)
- [ ] Rollback procedure tested (manually deploy old tag, verify it works, redeploy current)
- [ ] DB restore tested (DO backup → new cluster → app connects)
- [ ] HSTS active at 1-month max-age
- [ ] Bot Fight Mode ON
- [ ] R2 image bucket has token scoped to that bucket only (not full account)
- [ ] Frontend `<Image>` patterns include `cdn.oglasino.com`
- [ ] CORS in backend allows only `oglasino.com` and `www.oglasino.com` (no Vercel preview URLs in prod)
- [ ] Sentry installed and capturing errors
- [ ] UptimeRobot/Pingdom hitting `/health` every 60s
- [ ] Backup retention dates documented somewhere you can find
- [ ] Recovery console password set for `igor` user on both droplets
- [ ] SSH key + Cloudflare API token + Vercel token all in password manager with strong unique passwords

Don't launch with any of these red.

---

# Appendix A — Spring Boot health endpoint

The Worker references `/health` for the optional backend probe. Add a lightweight controller (separate from actuator):

```java
package com.memento.tech.oglasino.health;

import java.util.Map;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {

  @GetMapping("/health")
  public ResponseEntity<Map<String, String>> health() {
    return ResponseEntity.ok(Map.of("status", "ok"));
  }
}
```

Lightweight by design — doesn't touch DB/Redis/ES. Different concern from `/actuator/health/readiness` (which checks dependencies and is used by Docker healthcheck).

> **Don't expose actuator publicly.** Add to your Caddyfile if you haven't:
> ```caddyfile
> @actuator path /actuator/*
> respond @actuator 404
> ```
> Insert before `reverse_proxy localhost:8080`. Internal monitoring uses actuator within the container; the Worker uses `/health` from outside.

# Appendix B — Memory budget reference

Backend droplet (2 GB total RAM):

| Component | Memory |
|---|---|
| Linux + Docker daemon | ~250 MB |
| Caddy | ~30 MB |
| Redis (Docker, capped at 100 MB cache) | ~150 MB |
| Spring Boot (Docker, 1500 MB limit, JVM heap ~1050 MB with `MaxRAMPercentage=70`) | ~1500 MB |
| Headroom / OS buffers | ~70 MB |
| **Total** | **~2000 MB** |

Tight but workable. Watch for:
- Sustained swap usage > 200 MB → upgrade to 4 GB ($24/mo)
- JVM OOMKilled in `docker ps` → bump container memory limit (rare; usually means heap leak)
- Redis hitting maxmemory often → not a problem, that's LRU eviction working

# Appendix C — Common gotchas (from this iteration)

A growing list of things that wasted hours:

**1. Ubuntu socket activation breaking SSH port changes.** §1.4 covers the safe sequence. Never assume `sshd_config Port` is enough.

**2. Spring Boot `.env` not being loaded.** Move `spring.config.import` from `application-local.yml` to `application.yml` (base config).

**3. ES 8.x with Spring Boot 4.** Use ES 9.0.1. ES 8 fails on `indices.exists` with "Expecting a response body."

**4. Caddy missing `caddy-cloudflare-ip` module.** Don't reference `trusted_proxies cloudflare`. Stock Caddy + UFW lockdown is enough.

**5. Caddy serving the wrong hostname.** Cert must match SNI — Worker connects via `api-origin.oglasino.com`, so that's the cert. Not `api.oglasino.com`.

**6. Java preview features missing.** Add `--enable-preview` to Dockerfile CMD AND IntelliJ VM options. Without both, runtime fails immediately.

**7. Rotating leaked credentials.** Anything that has left your password manager (chat tools, git, screenshots, support tickets) is compromised. Rotate. Build the habit of placeholders only.

**8. R2 `cdn.oglasino.com` won't work as a manual DNS record.** Use R2's "Connect Domain" flow instead.

**9. Cloudflare free tier rate limiting is 10s windows only.** Use Worker bindings for 60s windows. Don't waste time on dashboard rules at this tier.

**10. `www.api.oglasino.com` cert issue.** Multi-level subdomain wildcards aren't covered by free SSL. Delete the record; nobody types it anyway.

**11. `next lint` removed in Next 15.** Use `eslint .` directly.

**12. Vercel UI requires production branch to exist.** Create a phantom `production` branch that's never updated. Set Vercel to track it.

**13. Postgres password-only first init.** After changing `POSTGRES_PASSWORD` in compose: `docker compose down -v && up -d`. `-v` wipes the volume; password is read fresh.

**14. Redis Lettuce pool config without commons-pool2.** Pool settings in YAML are silently ignored. Either add the dep or remove the config.

**15. Spring Cache `time-to-live: 0` means forever.** If you want a 1-hour TTL, set `3600000` (ms). Or use `RedisCacheConfiguration` per-cache.

**16. UFW Cloudflare lockdown script bails silently on first run (v4 bug, fixed in v5).** Original script uses `set -euo pipefail` and starts with `sudo ufw status numbered | grep 'Cloudflare' | …`. On a fresh box there are no Cloudflare-tagged rules yet; `grep` exits 1; `set -e` kills the script before it adds any rules. You see a clean prompt and a totally unchanged UFW. Fix: wrap the cleanup block in `{ … } || true` and use `xargs -r`. The v5 §4.9 has the patched version.

**17. Cron alone does nothing on a fresh box.** The cron line `0 4 * * * /opt/oglasino/scripts/update-ufw-cloudflare.sh` schedules the script; it doesn't execute it. You must run the script manually at least once to apply the lockdown. Cron just keeps the IP list fresh from then on.

**18. Docker bypasses UFW for published ports.** When `docker-compose.yml` publishes `9200:9200`, Docker inserts `ACCEPT` rules in iptables' `FORWARD` chain that run before UFW's `INPUT` rules ever see the traffic. UFW rules for Docker-published ports are documentation, not enforcement. Real protection: bind to a specific IP in compose (`"10.114.0.4:9200:9200"`), use VPC isolation, and rely on application-layer auth (`xpack.security` for ES). The UFW rule should still be kept accurate for audit clarity.

**19. DO droplets have two network interfaces; the panel IP ≠ `ip addr show eth0` private IP.** `eth0` carries the public IP plus a private IP on a "legacy" VPC (e.g. `10.19.0.0/16`). `eth1` carries the project VPC IP visible in the DO panel (`10.114.0.0/20` on `default-fra1`). Always use the panel IP for `.env` configs and UFW source rules. The `eth0` private IP isn't the one the app stack uses — we wasted real time debugging an ES "connection timeout" that turned out to be the `.env` using `10.114.0.4` correctly while the UFW rule allowed a stale `10.114.0.2` from a previous droplet.

**20. Stale UFW source-IP rules survive droplet rebuilds.** Rebuilt the backend droplet → got a new VPC IP → ES UFW still allows the *old* backend IP → backend can't reach ES → app silently fails on search. The UFW rule looks correct in `ufw status` but enforces nothing useful. Post-rebuild checklist: audit every UFW rule on every droplet that references a private IP.

**21. Reserved IPs are free while attached and survive rebuilds.** Assign one to every droplet on day one. The disaster-recovery story becomes "spin up replacement, move Reserved IP, done." No Cloudflare DNS edit needed.

**22. Proxied DNS records' "Content" field is a load-bearing fallback if the Worker fails.** A proxied `oglasino.com A 46.101.166.120` looks irrelevant because the Worker handles everything, until one day the Worker route is deleted or the Worker errors out. Cloudflare then falls back to the Content IP. If that IP no longer belongs to you (because you destroyed the droplet without a Reserved IP), strangers receive your traffic. Always point fallback Content fields at the Reserved IP, not the anchor IP. Cleaned up in v5 for all proxied records.

**23. Vercel "Add custom domain" is independent of Cloudflare DNS records.** Even with all Cloudflare DNS records pointing the Worker at Vercel, Vercel itself doesn't know about `oglasino.com` unless added in the Vercel project's Domains settings. Currently (v5 publish): the Worker forwards via `env.FRONTEND_ORIGIN` so frontend serves correctly, but Vercel sees `Host: oglasino-web-prod.vercel.app` and attributes all traffic and analytics to that hostname. Add custom domains in Vercel before launch (see "What's left").

# Appendix D — Glossary of records, names, and conventions

**Hostnames:**
- `oglasino.com` — apex, frontend (Worker → Vercel)
- `www.oglasino.com` — 301s to apex (Worker → Vercel)
- `api.oglasino.com` — public API endpoint (Worker → Caddy → Spring Boot)
- `api-origin.oglasino.com` — Worker bypass to backend (gray cloud, points at Reserved IP `157.245.22.189`)
- `stage.oglasino.com` — stage frontend (Worker → Vercel stage project)
- `api-stage.oglasino.com` — stage API (Worker → stage Caddy → stage Spring Boot)
- `api-origin-stage.oglasino.com` — Worker bypass to stage backend (gray cloud, points at Reserved IP `139.59.204.182`)
- `cdn.oglasino.com` — R2 image bucket public domain (prod)
- `cdn-stage.oglasino.com` — R2 image bucket public domain (stage)

**Droplet hostnames and IPs (v5 current state):**
- `oglasino-prod` — backend droplet (Caddy + Spring Boot + Redis). Public: `157.245.22.189` (Reserved). Anchor (no DNS refs): `46.101.166.120`. VPC `eth1`: `10.114.0.6`.
- `oglasino-stage` — stage all-in-one droplet (Caddy + Spring Boot + Redis + Postgres + ES, all dockerized). Public: `139.59.204.182` (Reserved). Anchor: `68.183.211.5`. VPC `eth1`: `10.114.0.x` (verify from panel).
- `oglasino-elasticsearch` (alias `oglasino-es`) — prod-only Elasticsearch droplet. Public: `167.172.106.33`. VPC `eth1`: `10.114.0.4`. No Reserved IP (not directly public; ES bound to `10.114.0.4:9200` only).

**Repos:**
- `memento-tech/oglasino-backend` — Spring Boot
- `memento-tech/oglasino-web` (or `oglasino-frontend`) — Next.js

**Branches (both repos):**
- `dev` — integration
- `main` — production
- `feature/*` — your work
- `production` — frontend only; phantom branch for Vercel UI

**Cloudflare:**
- Worker: `oglasino-prod-router` (handles routes: `oglasino.com/*`, `www.oglasino.com/*`, `api.oglasino.com/*`, plus stage equivalents)
- KV namespace: `oglasino-config` (binding `CONFIG`)
- KV keys: `maintenance.active`, `use.backend.check`
- R2 buckets: `oglasino-images-prod`, `oglasino-images-dev`, `oglasino-images-stage`, (later) `oglasino-db-backups`
- Pages: `oglasino-maintenance` (the 503 page when maintenance is on)

**Vercel:**
- Prod project: `oglasino-web-prod` (Vercel default URL: `oglasino-web-prod.vercel.app`)
- Stage project: separate Vercel project for stage frontend
- Region: `fra1`
- ⚠️ Custom domains (`oglasino.com`, `www.oglasino.com`, `stage.oglasino.com`) NOT yet added in Vercel project settings — see "What's left".

**Local dev tools (your laptop):**
- `docker-compose.local.yml` — Postgres + Redis + ES, Spring Boot runs natively in IntelliJ
- IntelliJ run config: profile=`local`, VM options=`--enable-preview`

**Users on droplets:**
- `igor` — primary sudo user (formerly `nicolas` in v4 docs; same person, account renamed in v5 reality)
- `deploy` — non-interactive deploy user, owns `/opt/oglasino/scripts/*`

---

# Appendix E — What's left to do (v5 post-session security & ops gap list)

This appendix lists every known gap as of the v5 publish date. Items are grouped by severity. The 🔴 set is what would most plausibly cause an incident in the next 6 months. The 🟡 set are things to close before public launch. The 🟢 set are good hygiene that can wait.

The work executed in the May 15 session (UFW lockdown, Reserved IPs, DNS cleanup, ES rule fix) is **not** repeated here — it's already in §4.9 and the "What changed from v4" section. This is everything that was **not** done.

---

### 🔴 High concern — address soon

**1. Verify Cloudflare WAF and Bot Fight Mode are active and in Block mode.**
The whole point of the UFW lockdown is to funnel all traffic through Cloudflare so its protections become the only path. If those protections aren't ON or are in Log-only mode, the lockdown has just rerouted the bot pressure, not eliminated it. To verify:
- Cloudflare dashboard → Security → Bots → confirm "Bot Fight Mode" is ON for the zone.
- Cloudflare dashboard → Security → WAF → Managed Rules → confirm Action is **Block**, not Log.
- Cloudflare dashboard → Security → Events → review last 24h. If bot/malicious requests are still high after the UFW lockdown, the Cloudflare layer isn't doing its job.

**2. Confirm Worker rate-limiter bindings (AUTH/WRITE/GLOBAL) are deployed and firing.**
Blueprint §3 references three rate limiters bound to the Worker. Verify in the Worker's dashboard → Settings → Bindings, AND test by hammering an endpoint (e.g. `for i in {1..200}; do curl -s -o /dev/null -w "%{http_code}\n" https://api.oglasino.com/some-endpoint; done`). If you never see 429s, the limiters aren't firing.

**3. Add application-level rate limiting in Spring Boot (Bucket4j).**
Cloudflare's Worker limiters protect against bulk abuse but anything that gets past them (authenticated abuse, slow trickle attacks, login brute force) hits Spring Boot uncapped. Login endpoints in particular. Bucket4j with a Redis-backed bucket gives per-user/per-IP limits at the application layer. Already in your blueprint roadmap; finish it.

**4. Reduce ES privileges from the `elastic` superuser to a scoped application user.**
The backend `.env` uses `ELASTICSEARCH_USERNAME=elastic` — that's the cluster root. Anyone who obtains that password (via a backend droplet compromise, a `.env` leak, a memory dump) has full ES control: drop indices, exfiltrate everything, modify documents. Create a dedicated user (e.g. `oglasino_app`) with only the index permissions the app actually needs, rotate the `elastic` superuser password into a sealed envelope.

**5. Add fail2ban to both prod and stage droplets.**
Blueprint §1.5 specifies fail2ban with `sshd` jail enabled. Verify it's actually installed and running on both droplets (`sudo systemctl status fail2ban`, `sudo fail2ban-client status sshd`). SSH on port 2202 is security through obscurity — a scanner *will* find it; fail2ban turns brute-force attempts into temporary bans.

**6. Rotate any secret that has ever been pasted into a chat tool, screenshot, support ticket, or git history.**
Per the blueprint's own discipline: anything that has left your password manager is compromised. This is the highest-ROI security action because it costs only time. Inventory: ES `elastic` password, Postgres `oglasino_app` password, Firebase service account JSON, Cloudflare API token, Vercel deploy token, GitHub deploy SSH key, R2 secrets, any third-party API keys.

---

### 🟡 Medium concern — close before public launch

**7. Add `oglasino.com`, `www.oglasino.com`, and `stage.oglasino.com` as Vercel custom domains.**
Right now the Worker forwards via `env.FRONTEND_ORIGIN`, so frontend serves correctly, but Vercel sees `Host: oglasino-web-prod.vercel.app` and attributes all traffic to that hostname. Vercel Analytics, Web Vitals, edge config, geo routing — all miscategorized. Adding the custom domain in Vercel project settings does NOT require changing the Worker.

**8. Verify Postgres backups actually restore.**
Daily automated backups + 7-day PITR are configured (per blueprint §1.9), but untested backups aren't backups. Do a restore drill: spin up a temporary cluster from the latest backup, point a copy of the app at it, verify a known recent row exists. Time how long it takes (this becomes your DR RTO baseline). Document the procedure.

**9. Add Postgres weekly belt-and-suspenders dumps.**
The DO managed Postgres backups are good, but if your DO account is ever compromised or you lose access to the project, those backups go with it. A weekly `pg_dump` from the backend droplet to R2 (encrypted at rest) gives you an independent recovery path. The `backups/postgres/` directory in `/opt/oglasino` is already there from §2.4.

**10. Tighten Cloudflare account security.**
- 2FA enabled on the Cloudflare account (verify).
- Cloudflare API tokens scoped to minimum permissions (no "All Resources" tokens).
- Audit log review enabled.
- Recovery codes saved in password manager.
- Email alerts on dashboard logins from new IPs.

**11. Pin Docker image tags by SHA256 digest.**
Currently `docker-compose.yml` uses mutable tags (`redis:7-alpine`, `postgres:16-alpine`, `elasticsearch:9.0.1`). Tag-mutation attacks are rare but real; pinning by digest (`redis@sha256:abc…`) closes this. Renovate or Dependabot can keep digests fresh.

**12. Enable TLS for ES traffic between backend and ES.**
Currently `ELASTICSEARCH_URIS=http://10.114.0.4:9200` — plaintext over the private VPC. DO VPCs are isolated, but "encrypted in transit always" is the standard. ES supports TLS; configuring it involves generating a self-signed CA, deploying certs to ES, and updating the backend's connection config. Non-trivial. Worth doing before launch.

**13. Move secrets out of `/opt/oglasino/.env` into a secret store.**
Plaintext `.env` on disk is acceptable for solo MVP but doesn't scale (rotation overhead, audit story, multi-user access). Options: Doppler, Bitwarden Secrets Manager, HashiCorp Vault, AWS Secrets Manager, or simpler: SOPS + age + git for declarative encrypted secrets in a config repo. Pick one before the team grows beyond solo.

**14. Add structured security audit logging.**
Spring Boot default access logs catch requests but not security-relevant events (failed logins, password changes, admin actions, permission grants, data exports). Add explicit audit logging at the application layer — failed auth, suspicious payloads, repeated 4xx from same IP. Ship to a separate sink (different retention from app logs) so an attacker can't trivially scrub the trail.

**15. Switch HSTS from 1-month to 1-year `max-age`.**
Caddy currently sets `Strict-Transport-Security "max-age=2592000"` (30 days). Once you're confident in TLS forever, bump to `max-age=31536000` (1 year) and add `includeSubDomains; preload` to be eligible for [hstspreload.org](https://hstspreload.org/) submission. Note: once preloaded, you cannot back out for years — only do this when TLS is stable.

**16. Audit and tighten CORS.**
Backend CORS allows certain origins. Verify the allowlist contains *only* `https://oglasino.com` and `https://www.oglasino.com` for prod — no Vercel preview URLs, no wildcard subdomains, no `*`. Same exercise for stage. CORS misconfig is one of the most common XSS-adjacent vulnerabilities.

**17. Test the maintenance flow end-to-end.**
Per blueprint launch checklist: "Maintenance flow tested end-to-end (ON → off via script → ON via script → off)." Has this actually been done since the v4 → v5 changes? Run it.

**18. Test the rollback procedure.**
Manually deploy a previous image tag, verify it works, redeploy current. Document the procedure as a runbook in the repo.

**19. Set up real uptime monitoring.**
Blueprint mentions UptimeRobot or Pingdom hitting `/health` every 60s. If this isn't live yet, set it up. Free tiers cover this. Without it, you'll learn about downtime from users.

**20. GDPR / Serbian LPDP compliance review.**
Serbia isn't EU but has a GDPR-equivalent law (LPDP — *Law on Personal Data Protection*). A marketplace dealing with user data needs: a privacy policy, a terms of service, a cookie banner (if any cookies set beyond strictly-necessary), a way for users to export and delete their data, a breach notification process. Not a one-evening task; budget time for this before launch.

---

### 🟢 Lower concern — good hygiene, can wait

**21. Add fail-closed restrictions on outbound traffic from droplets.**
UFW default `allow outgoing` lets a compromised backend exfiltrate data to anywhere. Restricting outbound to known destinations (Cloudflare IPs, Postgres host, ES VPC IP, R2 endpoints, GHCR, OS update repos) is the gold standard. Overkill for MVP, real protection later.

**22. File integrity monitoring on droplets (AIDE or similar).**
Detect if `/etc/passwd`, sshd config, Caddyfile, or cron get modified. Daily report. Free, low-overhead, catches a real class of post-compromise persistence.

**23. Frontend CSP, SRI, X-Frame, X-Content-Type-Options strict review.**
Caddy already sets `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, basic referrer policy. Add a Content Security Policy that allowlists only your own domains + Vercel + Cloudflare for scripts/images/fonts. Add Subresource Integrity (`integrity=`) on any external scripts.

**24. Image upload validation hardening.**
Files uploaded to R2 — verify: magic-byte file type check (not just MIME or extension), max file size enforced server-side, EXIF stripping, no executable formats accepted, no zip-bomb–style payloads, no SVG with embedded scripts.

**25. Document recovery console procedures.**
The blueprint mentions setting a Unix password for `igor` so the DO recovery console works. Document where that password lives in the password manager, and rehearse opening the recovery console once so you know the interface in advance of an emergency.

**26. Restore retention dates documented somewhere.**
Per launch checklist: "Backup retention dates documented somewhere you can find." Where? Make sure this exists in writing.

**27. Cleanup: the v4 user rename.**
`AllowUsers nicolas deploy` in §1.4.2's `sshd_config` example was renamed to `igor deploy` for v5 doc consistency. On droplets where the `nicolas` user has been renamed to `igor`, verify `sshd_config` actually allows `igor`. Also: any cron jobs, deploy scripts, GitHub Actions secrets, or systemd units referencing `nicolas` need updating. Audit with `grep -r nicolas /etc /opt /var/spool/cron`.

**28. The weird permissions on `/opt/oglasino/scripts` on stage.**
Observed during v5 session: `drwx-w---- deploy docker`. Group has write-only (no read/execute). Probably a setup typo. Should be `drwxr-xr-x` or similar. Doesn't break anything but odd.

**29. Clean up commented-out Postgres lines in prod `.env`.**
The first 3 lines of prod's `.env` are commented-out `POSTGRES_DB/USER/PASSWORD` entries, leftover from before managed Postgres. Harmless but noise.

**30. Cloudflare "Always Use HTTPS" page rule.**
Verify it's on (Cloudflare dashboard → Rules → Page Rules, or Single Redirects). Should redirect any http:// request to https://. Often defaults on, but verify.

---

### Snapshot of where we are at v5 publish

Things that **are** in place and working:
- ✅ Network perimeter: UFW Cloudflare-only on prod and stage, daily refresh cron
- ✅ Reserved IPs on both droplets, no DNS references to anchor IPs
- ✅ ES on `10.114.0.4:9200`, bound to private VPC interface only, requires auth, UFW rule clean
- ✅ Managed Postgres on private VPC, TLS-only, daily backups + PITR
- ✅ SSH on non-standard port, key-only auth, separate sudo user, recovery-console password set
- ✅ Cloudflare in front of all public traffic
- ✅ Worker routing maintenance flag, hostnames, KV-driven config
- ✅ Frontend on Vercel with Worker forwarding

Things that are **not** in place — the appendix above is your TODO list.

The single highest-ROI next action: **verify Cloudflare WAF + Bot Fight Mode are ON and in Block mode** (🔴 #1). Everything else can wait a few days. That one can't, because the lockdown work only matters if Cloudflare itself is filtering at the edge.

---

*End of v5 blueprint. Reflects state as of May 15, 2026. Next iteration (v6) likely after: WAF/Bot-Fight verification + Bucket4j integration + secrets-store migration + first real users.*
