# Google Play — Release Tracks

## Track flow

```
Internal Testing  →  Closed Testing  →  Open Testing  →  Production
   (stage)             (optional)         (optional)       (prod)
```

For v1, only **Internal Testing** (stage) and **Production** (prod)
are wired up. Closed/Open testing tracks can be added later if a
broader beta group is needed.

## EAS Build → track mapping

| Branch | EAS profile | Track | Trigger |
|---|---|---|---|
| `stage` | preview | Internal Testing | GH Actions on push |
| `main` | production | Production | GH Actions on push |

Specifics (signing key alias, service account JSON for Play Console
upload) populated during Phase 3E.4 of
[`../master-plan.md`](../master-plan.md). Cross-reference
[`../expo/cloud-setup.md`](../expo/cloud-setup.md).

## Internal Testing setup

- **Tester list:** TBD (Igor + invited testers)
- **Service account:** TBD (Google Play Developer API, used by EAS to
  upload AABs)
