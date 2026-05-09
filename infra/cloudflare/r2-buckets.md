# Cloudflare R2 Buckets

| Bucket | Stores | Accessed via | Public? |
|---|---|---|---|
| oglasino-images-prod | User-uploaded images for prod (originals + transforms) | `oglasino-images-prod` Worker, gated by JWT | no — Worker mediates all reads/writes |
| oglasino-images-stage | User-uploaded images for stage | `oglasino-images-stage` Worker, gated by JWT | no — Worker mediates all reads/writes |

## Access model

R2 buckets are not publicly readable. Every read and write goes
through the corresponding Cloudflare Worker, which validates a JWT
issued by the backend. The JWT signing secret is shared between the
backend and the Worker (see `JWT_SIGNING_SECRET_*` in
[`../overview/secret-inventory.md`](../overview/secret-inventory.md)).

Worker → backend admin calls (e.g., for cleanup jobs) use a separate
shared secret (`BACKEND_SHARED_SECRET_*`).
