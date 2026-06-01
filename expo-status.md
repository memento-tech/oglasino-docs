# Expo Status

Quick at-a-glance overview of the `oglasino-expo` release-readiness program.
Derived dashboard — on any disagreement, the sources of truth win.

**Sources of truth:** [`features/expo-release-readiness.md`](features/expo-release-readiness.md) · [`state.md`](state.md) · [`issues.md`](issues.md)

**Updated:** 2026-06-01
**Active now:** Image-pipeline close-out — fixing device bugs
**Branch:** `new-expo-dev` (committed)
**Native rebuild:** done — device verification unblocked

---

## At a glance

| Work | Status |
|------|--------|
| Foundations (auth · nav · performance · services) | [x] shipped |
| Product validation | [x] shipped |
| Messaging | [x] shipped |
| Filtering & search | [x] shipped |
| User-deletion | [x] shipped |
| Consent UI v1 | [x] shipped |
| Backend-calls optimization | [x] shipped |
| Admin removal · maintenance-split · version checksums | [x] shipped |
| **Image-pipeline** | [~] closing — device bugs being fixed |
| Analytics GA v1 | [ ] not started |
| Structural sweep (Ω) | [ ] not started |

---

## What's left

- [~] **Image-pipeline** — fix device bugs, then mark shipped. [Close-out handoff](.agent/handoffs/image-pipeline-closeout.md).
- [ ] **Analytics GA v1** — whole chat; consent gate is ready.
- [ ] **Structural sweep (Ω)** — dead-code + parked low-severity issues.
- [ ] **Pre-deploy checklist** — Google Sign-In validation + final checks. [Checklist](infra/expo/pre-deploy-checklist.md).

Open mobile bugs live in [`issues.md`](issues.md).
