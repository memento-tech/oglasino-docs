# Cross-Repo Docs Consolidation (Deferred)

**Status:** deferred
**Revisit:** post-launch

Three cross-cutting docs currently live in code repos and overlap with feature specs that live (or will live) in [`../features/`](../features/). They should migrate here. **Not now** — the move is deferred until after launch to avoid churn during the most fragile period.

## Files to migrate

| Source | Overlaps with | Notes |
|---|---|---|
| `oglasino-backend/docs/16-image-pipeline.md` | future `features/image-pipeline.md` | Cross-platform contract (backend issues R2 push token, web + mobile upload direct). |
| `oglasino-web/docs/product-validation.md` | [`../features/product-validation.md`](../features/product-validation.md) | Web-side description of the same flow. |

## Source of truth, today

Until the migration happens, the **feature spec in [`../features/`](../features/) is authoritative**. If the code-repo doc and the feature spec disagree, the feature spec wins. Engineer agents read `oglasino-docs/features/<slug>.md`, not the sibling-repo doc.

## Why deferred

Per [`../meta/conventions.md`](../meta/conventions.md) Part 1: existing files in `<repo>/docs/` can be edited or deleted, but new cross-cutting work goes into `oglasino-docs/`. The three files above pre-date that rule. Moving them now adds churn without product value; doing it after launch lets us:

- Avoid touching live reference material engineers are reading mid-feature.
- Collapse the two `product-validation.md` files into the canonical `features/product-validation.md` only after the validation refactor ships, so we don't rewrite the same doc twice.
- Treat the migration as a single dedicated cleanup pass rather than spreading it across feature work.

## Migration sketch (when we do it)

1. Diff each code-repo doc against the matching `features/<slug>.md`. Promote any unique facts into the feature spec.
2. Delete the code-repo doc.
3. Leave a one-line breadcrumb in the code repo's `docs/README.md` index pointing to `oglasino-docs/features/<slug>.md`.
4. Confirm no other code-repo doc links to the deleted file (grep across `oglasino-backend`, `oglasino-web`, `oglasino-expo`).

## Conditions for unparking

- The validation refactor ([`../features/product-validation.md`](../features/product-validation.md)) is `shipped`.
- A dedicated Docs/QA brief is approved to do the consolidation pass.
