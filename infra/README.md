# Infrastructure

> **During the active build-out phase, start at
> [`master-plan.md`](master-plan.md) — it is the live checklist driving
> every other page in this directory.**

## Subdirectories

| Path | What it covers |
|---|---|
| [`overview/`](overview/) | System-wide architecture, environment matrix (dev/stage/prod), secret inventory. |
| [`digitalocean/`](digitalocean/) | Droplets, managed databases, deploy mechanism. |
| [`cloudflare/`](cloudflare/) | Workers (router, images, maintenance), R2 buckets, DNS records, access/bot rules. |
| [`firebase/`](firebase/) | Firebase projects (stage, prod), auth, Firestore rules, FCM. |
| [`vercel/`](vercel/) | Vercel project for `oglasino-web`, branch → environment mapping, env vars. |
| [`google-play/`](google-play/) | Google Play Console listing and release tracks. |
| [`apple/`](apple/) | Apple Developer Program enrollment + App Store Connect. |
| [`expo/`](expo/) | Expo project + EAS build profiles for the React Native app. |
| [`github/`](github/) | Repo inventory, GH Actions secrets, deploy workflows. |
| [`namecheap/`](namecheap/) | Domain registration and renewal records. |
| [`runbooks/`](runbooks/) | Operational procedures — deploy, rotate secrets, recover from droplet loss, debug stage vs prod, database-overload incident handling. |
