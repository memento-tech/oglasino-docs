# Firebase Projects

## Current state (pre-cutover)

- **Oglasino Dev** — used by both dev and prod (incorrectly). Decommissioned in Phase 4.11.
- **Oglasino Prod** — exists but not wired correctly. Decommissioned in Phase 4.12.

## Target state (post-cutover)

### oglasino-stage
- **Project ID:** oglasino-stage-49abb
- **Used by:** stage backend, stage web, stage RN, **and local development**
- **Auth providers:** email, Google
- **Created:** TBD
- **Owner:** oglasino (oglasino@gmail.com)

### oglasino-prod
- **Project ID:** TBD
- **Used by:** prod backend, prod web, prod RN
- **Auth providers:** email, Google
- **Created:** TBD
- **Owner:** Igor

## Local development uses stage Firebase

Decision recorded: local dev points at the stage Firebase project for v1.
This is acceptable risk while there's only one developer (Igor). The trigger
for spinning up a separate `oglasino-dev` Firebase project is:

- A second developer joins, OR
- A QA tester joins (their stage data must not be polluted by dev work), OR
- Stage starts hosting real-world testing scenarios that local dev shouldn't
  disrupt.

When any of those triggers fire, create a third Firebase project, copy the
local-dev configuration over, point `eas.json development` profile at it.
