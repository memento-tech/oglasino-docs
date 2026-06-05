# Expo Status

Quick at-a-glance overview of the `oglasino-expo` release-readiness program.
Derived dashboard — on any disagreement, the sources of truth win.

**Sources of truth:** [`features/expo-release-readiness.md`](features/expo-release-readiness.md) · [`state.md`](state.md) · [`issues.md`](issues.md)

**Updated:** 2026-06-02
**Active now:** Analytics GA v1 closed at `mobile-stable` (Igor 2026-06-02) — all 13 events verified on-device via Firebase DebugView (iOS + Android). Image-pipeline closed at `mobile-stable` (Igor 2026-06-01) — on-device smoke deferred. The A–I feature queue is complete. Remaining program work: the Ω structural sweep ([handoff ready](.agent/handoffs/expo-structural-sweep-omega.md)) and the deferred on-device Ψ smokes.
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
| **Image-pipeline** | [x] shipped — `mobile-stable`; on-device smoke deferred |
| **Analytics GA v1** | [x] shipped — `mobile-stable`; verified on-device (iOS + Android) |
| Structural sweep (Ω) | [ ] not started — handoff ready |
| Runtime verification (Ψ) | [ ] pending — deferred on-device smokes |

---

## What's left

- [x] **Image-pipeline** — closed at `mobile-stable` (Igor 2026-06-01); 14-case on-device smoke deferred, not skipped (failures, if any, are ordinary follow-ups). [Close-out handoff](.agent/handoffs/image-pipeline-closeout.md).
- [x] **Analytics GA v1** — closed at `mobile-stable` (Igor 2026-06-02); all 13 events mirrored from web, verified on-device via Firebase DebugView (iOS + Android).
- [ ] **Structural sweep (Ω)** — dead-code + parked low-severity issues. Next chat to open; [handoff ready](.agent/handoffs/expo-structural-sweep-omega.md). Note: the original dead-code list needs re-grep against HEAD — feature chats have been deleting dead code under the P4 rule.
- [ ] **Runtime verification (Ψ)** — deferred on-device smokes (Φ4 airplane-mode, F7 re-renders, F22 hydration, F9 image cache, image-pipeline 14-case). Last chat in the queue.
- [ ] **Pre-deploy checklist** — Google Sign-In validation + final checks. [Checklist](infra/expo/pre-deploy-checklist.md).

Open mobile bugs live in [`issues.md`](issues.md).
