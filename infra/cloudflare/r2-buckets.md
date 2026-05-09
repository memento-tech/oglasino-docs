# Cloudflare R2 Buckets

| Bucket | Purpose | Bound to Worker |
|---|---|---|
| oglasino-images-prod | Production product/avatar/chat images | oglasino-images-prod |
| oglasino-images-stage | Stage product/avatar/chat images | oglasino-images-stage |

Each bucket is bound to its Worker via `[[r2_buckets]]` block in
`oglasino-image-worker/wrangler.toml`. Bucket access is gated through
JWT-signed view tokens issued by the Spring Boot backend; Workers
verify tokens on every read.

## Branch-to-bucket mapping

- Push to `main` on `oglasino-image-worker` → deploys
  `oglasino-images-prod` Worker → reads/writes `oglasino-images-prod` bucket
- Push to `stage` on `oglasino-image-worker` → deploys
  `oglasino-images-stage` Worker → reads/writes `oglasino-images-stage` bucket
