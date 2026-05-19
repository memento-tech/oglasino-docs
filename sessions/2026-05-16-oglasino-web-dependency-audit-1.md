# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-16
**Task:** Dependency audit (read-only) — produce a complete inventory of every direct dependency with current version, latest available within the current major, and upgrade-safety assessment, output to one markdown file.

## Implemented

- Read `package.json` and enumerated all direct dependencies: 43 in `dependencies`, 19 in `devDependencies` (62 total).
- Ran `npm outdated --long` and `npx --yes npm-check-updates --target minor`, captured both outputs as the dual inventory source. `npm-check-updates --target latest` deliberately not run, per brief Step 4.
- Used `npm list --depth=0` to resolve installed versions for the packages npm outdated omitted (which were absent because their installed version already matched the latest within range — npm outdated hides those rows).
- Scanned `../oglasino-docs/issues.md` for documented web-dep pins. None found. The only version-pin entries in `issues.md` are backend-only (`opentelemetry-semconv 1.41.1`).
- Scanned `package.json` for non-`^` constraints (brief Step 9). Two found, both exact pins: `react: 19.2.0` and `react-dom: 19.2.0`. No documented reason; flagged for Igor.
- Produced `.agent/audit-dependency-upgrade.md` with: per-bucket summary, two inventory tables (deps + devDeps) one row per direct dep, "Notable observations" section covering the React/Next cluster, non-`^` pins, the `@types/node` ↔ `engines.node` mismatch, and an eight-package range-tighten list.
- Ran `npm audit` (Step 10) — completed well within the 2-minute cap. Captured 3 moderate-severity findings, all chained off one outdated `postcss` bundled by `next`. Added a Security findings section with the chain, the unsafe `audit fix --force` warning (it would seven-major-downgrade `next`), and a recommended `package.json overrides` fix path.

### Bucket counts (62 direct deps)

| Bucket                  | Count |
| ----------------------- | ----- |
| `up-to-date` (added)    | 50    |
| `safe-patch`            | 4     |
| `safe-minor`            | 1     |
| `review-minor`          | 3     |
| `major-skipped`         | 2     |
| `major-flagged-as-safe` | 0     |
| `pinned`                | 2     |
| `unknown`               | 0     |

`up-to-date` is an informational bucket added on top of the brief's seven. Reasoning is documented inline in the audit file: ~80% of the tree has no available upgrade, and `safe-patch` is misleading for "no bump exists."

`major-flagged-as-safe` is intentionally empty — I was conservative per the brief's last paragraph; nothing was assessed without reading release notes, which was out of scope.

## Files touched

Two new files in `.agent/` (tracked but expected to live alongside other engineer artifacts):

- `.agent/audit-dependency-upgrade.md` (new)
- `.agent/2026-05-16-oglasino-web-dependency-audit-1.md` (this file, new)
- `.agent/last-session.md` (overwritten with this file's content per conventions Part 5)

No source files, `package.json`, or `package-lock.json` were touched. Confirmed via `git status` mental model — pre-existing modifications listed at session start are unrelated to this audit.

## Tests

- N/A. Read-only audit per brief. No source code changed, so `npm run lint`, `npx tsc --noEmit`, and `npm test` were not run.

## Cleanup performed

- None needed. No code edits.

## Known gaps / TODOs

- The two `major-skipped` packages (`eslint` 9→10, `typescript` 5→6) and `lucide-react` 0.x→1.x were **not** evaluated against release notes for `major-flagged-as-safe`. The brief told me to be conservative; doing the release-note read for each is a separate, larger scope. If Mastermind wants a deeper assessment, that's a follow-up brief.
- `npm audit` surfaced the postcss vulnerability but Step 10's scope was reporting only — no fix applied. The recommended `overrides` block in `package.json` is a one-line change but out of scope for a read-only audit.
- `@tailwindcss/postcss`, `@types/node`, `@types/react`, `@types/react-dom`, `tailwindcss`, `typescript` all use very loose `^N` (major-only) constraints. Not formally flagged by Step 9 (those use `^`), but noted in the audit because the package.json floor is not a meaningful pin under those constraints — the lockfile carries the actual version. Igor's call whether to tighten or accept the drift.

## Obsoleted by this session

- Nothing. Read-only audit; nothing was made dead.

## Conventions check

- **Part 4 (cleanliness):** confirmed. No code edits, no lint/test runs needed. Two new `.agent/` files, both referenced by the audit workflow.
- **Part 4a (simplicity) / Part 4b (adjacent observations):** N/A for cleanliness; adjacent observations are routed via the "For Mastermind" section below.
- **Part 5 (session summary):** confirmed. This summary is written to both `.agent/2026-05-16-oglasino-web-dependency-audit-1.md` and `.agent/last-session.md`. The `-1` sequence number was determined by listing `.agent/` for `*-dependency-audit-*.md` files — none existed, so this is the first.
- **Part 6 (translations):** N/A this session.
- **Hard rules (no commit / push / branch switch / deploy / cross-repo / new `docs/` files):** confirmed — none invoked. `package.json` and `package-lock.json` deliberately untouched per brief.

## For Mastermind

1. **`react` and `react-dom` are exact-pinned at `19.2.0` with no documented reason.** No matching entry in `issues.md`. The likely reason is to keep them in lockstep, but that can be achieved with `^19.2.0` on both. **Decision needed:** keep exact pins (deliberate), or relax to `^19.2.0` and let the 19.2.6 patch flow in. Severity: low (patch is small). The audit file documents both options.

2. **`@types/node` ↔ `engines.node` mismatch.** `package.json` `engines.node` is `>=22`, but `@types/node` is constrained to `^20`. The TypeScript types are two majors behind the declared runtime. Severity: low-medium — the type signatures for any new Node 22+ API are missing or wrong. Recommended action: bump `@types/node` constraint to `^22` (or `^24`); cross-check whether anything in the source uses Node 22+-only APIs. Out of scope for this audit.

3. **`npm audit` surfaced a moderate postcss XSS chained through `next` → `next-intl`.** Fix sits upstream in `next` or in a `package.json overrides` block forcing `postcss >= 8.5.10`. The auto-fix npm suggests (`audit fix --force`) is unsafe — it would seven-major-downgrade `next`. Severity: low practical exposure (Tailwind/built-in CSS pipeline, no user-controlled CSS), but the fix is cheap. Recommended as a follow-up brief.

4. **Loose `^N` (major-only) constraints on six packages** — `@tailwindcss/postcss ^4`, `@types/node ^20`, `@types/react ^19`, `@types/react-dom ^19`, `tailwindcss ^4`, `typescript ^5`. Not Step 9-flagged (they do use `^`), but worth surfacing because the floor is unenforceable — the install drifts forward freely as soon as a new minor lands. Severity: low. Decision is whether to keep this drift-permitting style or tighten as part of a normalization pass.

5. **Eight packages have a range-tighten opportunity** (installed > declared floor): `@tanstack/react-virtual`, `@next/bundle-analyzer`, `eslint-config-next`, `firebase`, `next`, `prettier`, `react-syntax-highlighter`, `zustand`. Pure cosmetic — listed in the audit file. Severity: low. If Igor wants the `package.json` floor to reflect what's actually deployed, this is one batch edit.

6. **Three packages need `review-minor` attention before upgrade:** `lucide-react` (pre-1.0 minors can rename/remove icons), `next-intl` 4.11.1 → 4.12.0 (translation hook behaviors and TS types), `prettier-plugin-tailwindcss` 0.7.4 → 0.8.0 (pre-1.0 minor; class-ordering output may shift, producing a large cosmetic diff). All three have concrete verification steps documented in the audit's Notes column.

7. **Adjacent observation, low severity.** Out-of-scope for this brief, but worth flagging: the four working-tree modifications listed at session start (`src/lib/utils/filtersHelper.ts`, `src/lib/validators/productSchemas.ts`, three `service/*` files, two `(public)`/`(protected)` page files, `src/components/client/initializers/FilterHydrationSSRInit.tsx`, `src/lib/data/emptyOverviews.ts` untracked) appear unrelated to this audit and are uncommitted on `dev`. Did not investigate; just noting. Severity: low — this is Igor's working tree, not a defect.
