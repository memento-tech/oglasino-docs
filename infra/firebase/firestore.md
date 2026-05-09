# Firestore

## Rules deployment

Firestore security rules live in the dedicated repo
**`oglasino-firestore-rules`**. The repo's CI runs `firebase deploy
--only firestore:rules` on push, using `FIREBASE_TOKEN` from GH
Secrets (see
[`../overview/secret-inventory.md`](../overview/secret-inventory.md)).

Branch model: `stage → prod`. A push to `stage` deploys to
`oglasino-stage`; a push to `prod` deploys to `oglasino-prod`.

## Cutover note

Firestore wipes happen during Phase 4 cutover when the old "Oglasino
Dev" and "Oglasino Prod" projects are decommissioned (4.11, 4.12).
There is no data preservation concern — we are pre-prod, no real users.

## Indexes

Firestore composite indexes are defined in the rules repo alongside
the rules file. Documented there, not here.
