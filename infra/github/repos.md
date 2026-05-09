# GitHub Repos

| Repo | Purpose | Branch model | CI/CD trigger |
|---|---|---|---|
| oglasino-backend | Spring Boot API | dev → stage → main | branch push triggers deploy.yml |
| oglasino-web | Next.js web frontend | dev → stage → main | branch push triggers deploy.yml + Vercel |
| oglasino-expo | Expo React Native app | dev → stage → main (dev no auto-deploy) | stage/main trigger EAS build |
| oglasino-image-worker | Cloudflare Worker for images | stage → main | branch push triggers wrangler deploy |
| oglasino-firestore-rules | Firestore security rules | stage → prod | branch push triggers firebase deploy |
| oglasino-maintenance | Static HTML maintenance page | main only | branch push triggers wrangler deploy |
| oglasino-docs | This repo (all documentation) | main only | no deploy |
