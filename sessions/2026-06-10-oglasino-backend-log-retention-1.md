# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-10
**Task:** 90-day application-log retention (infra) — guarantee application-log retention ≤ 90 days, enforced automatically, documented in-repo.

## Implemented

- Added an explicit `logging:` block (json-file driver + `max-size`/`max-file`)
  to every service in both compose files: backend (PII target, 20m × 80 ≈ 1.6 GB),
  redis (10m × 3 = 30 MB), elasticsearch (20m × 5 = 100 MB). This is the disk
  ceiling and, under load, purges early (compliant) and removes the prior
  unbounded-growth / disk-fill risk (Docker's default json-file has no rotation).
- Added `infra/prune-container-logs.sh` — the time ceiling. A host cron deletes
  the backend container's **rotated** json-file segments whose mtime is older than
  88 days (headroom under the 90-day promise for segment span + cron cadence). The
  active segment is never touched; scoped to the backend (only PII-bearing) container.
- Added `infra/LOG-RETENTION.md` — mechanism, full sizing math, the json-file+cron
  vs journald/logrotate justification, exact host-install steps for Igor, and the
  container-**recreate** warning (logging changes need `up -d --force-recreate`, not
  a plain restart — a scheduled backend blip).
- Re-verified all audit citations before working (see below). No Java changes.

## Files touched

- infra/docker-compose.yml (+16 / -0) — logging blocks on backend + redis
- infra/docker-compose.es.yml (+6 / -0) — logging block on elasticsearch
- infra/prune-container-logs.sh (new, +~90) — 88-day rotated-segment pruner
- infra/LOG-RETENTION.md (new) — operator doc

## Tests

- No Java touched → no `./mvnw test` / `spotless` relevant to this change.
- `bash -n infra/prune-container-logs.sh` → OK (shellcheck not installed on host).
- `docker compose -f docker-compose.yml config -q` and `... .es.yml config -q` →
  parse clean (only expected "variable not set" warnings, no schema errors).
- Rendered config confirms options applied: backend `max-file:"80" max-size:20m`,
  redis `"3"/10m`, es `"5"/20m`.

## Re-verified citations (direct read + rg, both)

- `logging/RequestLoggingFilter.java` lives at
  `src/main/java/com/memento/tech/oglasino/logging/RequestLoggingFilter.java`
  (brief's package-relative path; actual root package is `com.memento.tech.oglasino`).
  `clientIp` resolved l.57 / MDC l.63; `userId` MDC l.71–76; ACCESS line emitted
  l.84 (warn ≥500) / l.86 (info). Brief's "55–99" range confirmed.
- Console pattern with `usr=%X{userId}` + `ip=%X{clientIp}` at
  `application-prod.yaml:171`. Confirmed.
- No `logging:` blocks existed anywhere under `infra/` before this session
  (`rg "logging:" infra/` → empty). Confirmed.
- NOT FOUND: nothing the brief cited was missing.

## Cleanup performed

- none needed (net-new infra artifacts + additive compose blocks; no dead code).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change required, but a candidate entry is drafted in
  "For Mastermind" (json-file+cron retention design, the 88-day cut rationale, and
  the journald rejection) for Mastermind to decide whether it warrants logging.
- state.md: no change strictly required. A one-line note that the 90-day
  log-retention publication blocker is now closed in-repo (pending Igor's host
  apply) is drafted in "For Mastermind" if Docs/QA wants to reflect it.
- issues.md: no change. (The audit Q7a finding that prompted this is addressed;
  Q7b `incident_log` follow-up remains out of scope and already tracked separately.)

## Obsoleted by this session

- nothing. The change is purely additive — no prior retention mechanism existed to
  supersede, no tests or docs contradicted.

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no debug logging, no
  unreferenced files (script is referenced by LOG-RETENTION.md + intended cron;
  doc is the deliverable).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one low-severity note flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 1 (doc style — new `infra/LOG-RETENTION.md`, kebab-ish
  caps filename matches the existing `infra/` convention e.g. `dc.sh`; it is an
  in-repo ops doc, not an `oglasino-docs/` feature spec, so Part 3 "no new docs in
  repo docs/" does not apply — this is `infra/`, not `docs/`).

## Known gaps / TODOs

- The 90-day ceiling is only live once Igor applies it on the droplet: ship files,
  `up -d --force-recreate` the affected containers, install the root cron. All
  steps are in `infra/LOG-RETENTION.md`. No code TODO left in the tree.
- Sizing assumes 50,000 requests/day. If real launch traffic is materially higher,
  the backend size cap (not the cron) should be revisited — documented in the doc.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): (1) the host cron script — earns its place because
    size caps provably cannot guarantee the time ceiling on a quiet system, which
    is the exact compliance gap; (2) the 88-vs-90 headroom constant — earns it
    because deleting by segment mtime overshoots true line-age by up to one segment
    span, and the promise is a hard ceiling. (3) A driver guard in the script
    (refuses to act if not json-file/local) — one cheap branch that prevents a
    future driver change from silently deleting the wrong files.
  - Considered and rejected: journald + system-wide `SystemMaxRetentionSec`
    (rejected: system-wide blast radius, loses the per-container size caps,
    changes operator log-reading); logrotate on json-file segments (rejected:
    races Docker's own rotation); extending the cron to redis/ES (rejected: no PII,
    no 90-day mandate — size caps suffice); a config flag for the retention window
    (rejected: it's one value with no second setting — left as a script constant /
    env override, per Part 4a "config is for values that vary").
  - Simplified or removed: nothing (additive change).

- **Adjacent observation (Part 4b):** `application-prod.yaml:166-171` logs the
  client IP and user id on **every** ACCESS line via the console pattern, but
  `RequestLoggingFilter` only emits one ACCESS line per request at INFO. This is
  intended, but worth noting that any future bump of a chatty logger to DEBUG in
  prod would multiply PII-bearing lines and the sizing math. **File:**
  `src/main/resources/application-prod.yaml`. **Severity:** low (informational; the
  same file already warns against DEBUG in prod at l.173-176). I did not change
  anything — out of scope.

- **Drafted decisions.md candidate** (Mastermind to decide if it lands; Docs/QA
  applies if so), target section: a new dated entry under 2026-06-10:

  > **Application-log 90-day retention — json-file caps + host cron.**
  > Backend ACCESS logs carry client IP + user id (Privacy Policy §2.12/§8.2 promises
  > 90-day purge). Enforced in two layers: per-service json-file `max-size`/`max-file`
  > caps in both compose files (disk ceiling; purges early under load = compliant),
  > plus `infra/prune-container-logs.sh` run daily from root cron, deleting the
  > backend container's rotated json-file segments older than 88 days (headroom below
  > 90 for segment-span + cron cadence). Scoped to the backend container only —
  > redis/ES carry no PII and get size caps alone. journald + system-wide
  > `SystemMaxRetentionSec` was rejected (system-wide blast radius, loses per-container
  > size caps); logrotate rejected (races Docker's rotation). Sizing assumes
  > ~50k req/day → ~15 MB/day → backend cap 20m×80 ≈ 1.6 GB. Applying it requires a
  > container **recreate** (`up -d --force-recreate`), a scheduled backend blip.
  > Mechanism + install steps: `oglasino-backend/infra/LOG-RETENTION.md`.

- **Drafted state.md candidate** (optional), target: wherever the publication /
  privacy-policy blockers are tracked:

  > Backend 90-day application-log retention: built in-repo 2026-06-10
  > (compose logging caps + `infra/prune-container-logs.sh` + `infra/LOG-RETENTION.md`),
  > **pending Igor's host apply** (ship files, `up -d --force-recreate`, install cron).
  > Closes audit Q7a. Q7b (`incident_log`) remains a separate optional follow-up.

- Nothing else flagged.
