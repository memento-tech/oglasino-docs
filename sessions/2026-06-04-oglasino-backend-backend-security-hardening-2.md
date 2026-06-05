# Session summary — Brief 2 (H2): admin via per-env SQL seed, remove the admin-email branch

**Repo:** oglasino-backend · **Branch:** dev · **Date:** 2026-06-04
**Feature:** backend-security-hardening §3 (H2) · **Brief:** Engineering Brief 2
**Order-is-law gate:** seed mechanism + files verified-correct **first**, then JSON admin seeding
switched off + email branch removed. Gate passed (see Part 1 verification); no STOP triggered.

---

## Task (one sentence)

Move the admin grant out of code: provision the admin as a per-environment SQL seed row (keyed on
that env's Firebase UID, idempotent on every boot), switch off the JSON admin seeding, and delete
the `admin@oglasino.com` email-literal branch so all registrations become `ROLE_BASIC`.

---

## Part 1 — mechanism + schema verification (read-only, done first)

All cited `file:line` opened with Read and cross-checked with grep. No phantom-read divergence hit.

1. **Existing SQL seed mechanism.** `spring.sql.init.mode: always` + `data-locations` globs in each
   of `application-{dev,stage,prod}.yaml`. **All three yamls list the IDENTICAL globs**
   (`classpath:data/core/*.sql`, `…/translations/*.sql`, `…/configuration/*.sql`, `…/location/*.sql`,
   `…/basesite/*.sql`). So today the seeds are **env-agnostic** — every file runs in every env.
   There is **no** `data-${profile}.sql` convention and no profile-specific paths. **Per-env
   selection is NOT supported by a shared convention** (operator believed it was — see "Brief vs
   reality"). It IS achievable because each env has its own yaml: per-env selection = each yaml
   listing an explicit (non-glob) admin file. This matches `decisions.md` 2026-06-03 (DB-overload
   finding #4: "no per-env Configuration seed exists — all envs load the identical SQL").

2. **JSON test-data admin seeding.** `data/TestDataImportService` is `@Profile({"!prod"})` (runs on
   **dev + stage, NOT prod**), `@EventListener(ApplicationReadyEvent)`. It seeds users
   (`testUsers.json`), products, reviews. `testUsers.json` carried the admin as its **first** entry
   PLUS **3 load-bearing non-admin test users**; products/reviews are also load-bearing. ⇒ **Could
   not rip the seeder out — surgical removal of only the admin object.** Note: prod never ran this
   path, so prod's admin previously came **only** from the email-literal code branch (the exact
   workaround the brief describes).

3. **User entity / columns.** Table `users`. `userRole` is `@Enumerated(EnumType.STRING)` →
   confirmed STRING by the DDL CHECK `users_user_role_check` (`'ROLE_ADMIN'`/`'ROLE_BASIC'`). Columns
   `firebase_uid`, `email`, `display_name`, `subscription_type` (STRING), `subscription_active`,
   `preferred_language_id` (NOT NULL FK), `rating` (NOT NULL), `id` (NOT NULL).

4. **Unique constraints.** Both `users_firebase_uid_key UNIQUE (firebase_uid)` **and**
   `users_email_key UNIQUE (email)` exist. ⇒ `ON CONFLICT (firebase_uid)` is a real, valid target
   (chosen — the per-env identity). Email is also unique but on a fresh DB it does not bite.

5. **Login role-read path.** `UserRepository.findAuthDataByFirebaseUid` looks up by `firebase_uid`,
   **INNER JOINs `preferredLanguage`** (⇒ seed MUST set `preferred_language_id`) and **LEFT JOINs
   `baseSite`** (⇒ null `base_site_id` is safe for auth). Returns `AuthenticatedUserDTO.userRole`,
   consumed by `AuthUtils.subscriptionToAuthorities`. So a seeded `ROLE_ADMIN` row with the right
   per-env UID is found and honored on login; a wrong UID ⇒ row never found ⇒ admin can't log in.

**Additional schema fact (load-bearing):** `id` has **no DB default** — all entities draw from a
single shared sequence `global_id_seq` (`@SequenceGenerator` in `BaseEntity`). A raw INSERT must
supply `id` via `nextval('global_id_seq')`. (nextval is evaluated before the ON CONFLICT check, so
it burns one sequence value per boot even on the no-op — harmless for a bigint global sequence.)

---

## Brief vs reality

1. **"Per-env SQL is supported" — not as a convention; achievable per-yaml.**
   - Brief/operator says: per-env SQL selection is already supported; extend the real pattern.
   - Code says: all three env yamls list the **same** globs (`application-{dev,stage,prod}.yaml`
     `spring.sql.init.data-locations`); seeds are env-agnostic. No `data-${profile}` convention.
   - Why it matters: if I'd assumed a profile-keyed convention existed, the admin seed would have
     run in every env (or none). The correct mechanism is: each env's own yaml lists an **explicit,
     non-glob** admin file. I implemented exactly that, so the brief's stated design ("stage profile
     runs the stage file… via spring.sql.init in the respective application-*.yaml") still holds.
   - Resolution: implemented per-yaml explicit wiring; **no STOP** (the design works). Flagged here
     so the "per-env SQL supported" belief is corrected in the record.

2. **JSON admin seeder runs on dev+stage, not prod; prod's admin came from the email branch.**
   - Implication: removing the email branch genuinely removes prod's *only* admin-provisioning path
     today. The new prod SQL seed is therefore net-new provisioning capability for prod, exactly the
     gap the brief wanted closed. (Not a contradiction — a confirmation; recorded for clarity.)

Neither finding blocked work; both were resolvable from the code. No guess made.

---

## Part 2 — per-env admin seed files (authored on disk, not run)

New dir `src/main/resources/data/admin/` with three files, each a single idempotent INSERT into
`users` (`ON CONFLICT (firebase_uid) DO NOTHING`), `id = nextval('global_id_seq')`,
`user_role='ROLE_ADMIN'`, `email='admin@oglasino.com'`, `display_name='Admin User'` (matches what the
removed code set), `subscription_type='SUBSCRIPTION_FREE'`, `subscription_active=true`,
`email_verified_external=true`, `verified_internally=true`, `preferred_language_id=1`, and the
NOT-NULL booleans/`rating`/`deletion_status`/`num_of_penalties` set to the same defaults
`createUserSynchronized` produces for a normal user. `base_site_id`/`region_id`/`city_id` left NULL
(matches the code-path admin; LEFT-JOINed in the auth read, so safe).

- `data-admin-dev.sql` — local/dev admin UID **inline** (the same value already committed in
  `testUsers.json`; no new secret), so local boots create a working admin out of the box.
- `data-admin-stage.sql` / `data-admin-prod.sql` — authored with `<STAGE_ADMIN_FIREBASE_UID>` /
  `<PROD_ADMIN_FIREBASE_UID>` placeholders + `-- operator: replace…` comment. **During this session
  the operator filled both** directly in the files (stage = the dev UID, prod = a distinct prod UID).
  Left as-is per "don't revert operator edits"; the seed tests accept placeholder or real UID.

Each file is wired into **only its own** env yaml's `data-locations`, appended **after**
`data/basesite/*.sql` (and thus after `data/core` where language id 1 is seeded — satisfies the
`preferred_language_id` FK and the auth read's INNER JOIN). Each file carries a header comment
explaining it replaces the removed code-literal grant + the JSON admin seed (so it isn't
"simplified" away).

---

## Part 3 — switch off JSON admin seeding + remove the email branch

1. **JSON admin seeding:** removed **only** the admin object (first entry) from
   `dataJSON/testUsers.json`. The 3 non-admin test users + `TestUsersImportService` /
   `TestProductsImportService` / `TestReviewsImportService` are untouched (load-bearing). This stops
   the dev+stage double-provisioning that would otherwise race the new SQL seed.
2. **Email branch:** in `DefaultFirebaseAuthService.createUserSynchronized` deleted the
   `if (token.getEmail().equals("admin@oglasino.com"))` block; collapsed to an unconditional
   `newUser.setUserRole(UserRole.ROLE_BASIC)` with a comment pointing at the per-env seed. Normal
   displayName resolution preserved (the admin override no longer forces `"Admin User"`).
3. **Grep for other email-derived privilege:** none. The only remaining `admin@oglasino.com` strings
   are the three seed files (the row's email + the explanatory header comments). Remaining `ROLE_ADMIN`
   references are legitimate (enum def, `DefaultCurrentUserService` authority check, `DefaultUsersService`
   query filter).

---

## Part 4 — verification

- **Code no longer grants admin by email** — `DefaultFirebaseAuthServiceUserRoleTest` (new):
  registering with `admin@oglasino.com` ⇒ `ROLE_BASIC` and displayName NOT forced to "Admin User";
  any other email ⇒ `ROLE_BASIC`. (Mockito service-layer, mirrors the existing DisplayName test.)
- **Seeded admin honored on login** — `AdminSeedTest.seededAdminRowShapeIsHonoredAsAdminAuthority`:
  an `AuthenticatedUserDTO` of the exact shape the seed writes (ROLE_ADMIN, active SUBSCRIPTION_FREE)
  yields a `ROLE_ADMIN` authority through `AuthUtils`. The actual SQL execution + DB read by
  `findAuthDataByFirebaseUid` is **not** unit-tested (no DB in tests, per Brief 1's finding) and is
  left to Igor's stage/prod smoke — stated explicitly here.
- **Seed-file sanity** — `AdminSeedTest` (new, file-parse, mirrors `ConfigurationSeedTest`, no DB):
  each of the 3 files has the admin-seed shape (role, email, subscription, `preferred_language_id`,
  `nextval('global_id_seq')`, `ON CONFLICT (firebase_uid) DO NOTHING`, a non-blank firebase_uid), and
  each env yaml wires only its own admin file. Actual once-per-boot no-op is Igor's smoke (no
  SQL-execution harness exists; per brief, none was built).
- `./mvnw spotless:check` → **EXIT 0**. `./mvnw test` for the security/auth packages +
  `ConfigurationSeedTest` → **73 run, 0 failures/errors**.

---

## Obsoleted by this session

- The `admin@oglasino.com` email-literal `ROLE_ADMIN` branch in
  `DefaultFirebaseAuthService.createUserSynchronized` — deleted (not commented out).
- The admin entry in `dataJSON/testUsers.json` — deleted (surgical; the 3 test users remain).
- The stale javadoc in `DefaultFirebaseAuthServiceDisplayNameTest` describing the admin override as
  "unchanged… remains visible in production code" — corrected to point at the new role test.

## Cleanup performed

- No commented-out code left behind (email branch fully removed; the new comment is explanatory, not
  dead code). No unused imports introduced (`UserRole` still used for `ROLE_BASIC`). No
  `System.out`/debug logging. No `TODO`/`FIXME` added.

## Conventions check

- **Part 4 (cleanliness):** clean — see above; spotless + tests green.
- **Part 4a (simplicity):** chose `nextval('global_id_seq')` over a hardcoded id (avoids the
  reference-seed id/sequence-overlap footgun); per-env wiring via existing per-yaml `data-locations`
  rather than a new mechanism; surgical JSON edit rather than a new conditional in the importer.
- **Part 4b (adjacent observations):** the existing reference seeds use hardcoded low ids (e.g.
  base_site 1000) while `global_id_seq` starts at 1000 — a pre-existing cross-table id/sequence
  overlap pattern. Not touched (out of scope, different tables, no live break). Noted only.
- **Part 6 (translation seeds):** N/A — no translation keys added.
- **Part 7 (HTTP error contract):** N/A — no controller/error changes.
- **Part 11 (trust boundary):** improved — admin-ness is now a server-side DB row keyed on the
  env-controlled Firebase UID, never derived from a client-presentable email literal.

## Config-file impact

- **Backend application config (authored by me here, in-scope):** `spring.sql.init.data-locations`
  additions in `application-{dev,stage,prod}.yaml`. Done.
- **oglasino-docs config files (NOT edited — Docs/QA only):** none required by mechanism, but two
  records should be batched at feature close (drafts in "For Mastermind"). No edit to
  `conventions.md`/`decisions.md`/`state.md`/`issues.md` made this session. Closure gate: confirmed
  no implicit config-file dependency beyond those drafts.

---

## For Mastermind

1. **Correct the "per-env SQL is supported" assumption (decisions/record).** It is supported only in
   the sense that each env has its own yaml; there is **no** profile-keyed convention — all envs
   currently load identical globs. The admin seed achieves per-env by explicit per-yaml file entries.
   Any future "per-env seed X" work should follow this same explicit-per-yaml pattern, not assume a
   `data-${profile}.sql` convention exists.

2. **Dev admin UID is inline (deviation-with-rationale from the "never commit a real UID" rule).**
   The brief's hard rule scopes placeholders to stage/prod; for dev the brief says "match current dev
   behavior; don't break local." The dev UID (`qngqVFVIw7Rb…`) was **already committed** in
   `testUsers.json`, so I moved it into `data-admin-dev.sql` inline (no new secret, local works out of
   the box). If you'd rather dev also use a placeholder, say so and I'll swap it (local would then
   need the operator to fill it before the admin works on a fresh local DB).

3. **Operator filled stage + prod UIDs during the session.** `data-admin-stage.sql` and
   `data-admin-prod.sql` now carry real UIDs (stage = dev UID; prod distinct). If the intent was to
   keep placeholders in git until ship, revert those two literals back to the placeholder tokens.

4. **state.md / decisions.md (batch at feature close, draft only — do not let me write these):**
   - state.md: flip the H2 (admin-by-email) task to done for the backend.
   - decisions.md note: *Admin is now a per-environment seeded DB row (`data/admin/data-admin-*.sql`)
     keyed on the env's Firebase UID, idempotent via `ON CONFLICT (firebase_uid) DO NOTHING`;
     registration is unconditionally `ROLE_BASIC`; the email-literal grant and the JSON admin seed are
     removed.*

5. **Smoke reminder (unchanged from brief):** the seed only works once the real per-env UIDs are in
   place (now done in-file) and the DBs are the intended fresh state; verify boot creates the admin
   row, login reaches `/api/secure/admin/**`, restart stays idempotent, and a throwaway non-admin
   registers as `ROLE_BASIC`.
