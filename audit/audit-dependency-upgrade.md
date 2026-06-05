# Dependency audit — `oglasino-backend`

**Date:** 2026-05-16
**Branch:** `dev` (no changes; read-only audit)
**Inputs:** `pom.xml`, captured output of `./mvnw versions:display-dependency-updates -DprocessDependencyManagement=true`, `oglasino-docs/issues.md`.

This audit covers **direct dependencies only** — every `<dependency>` in `<dependencies>` and every pinned `<dependency>` in `<dependencyManagement>`. Transitive deps are out of scope per the brief.

Plugin upgrade audit was **not completed** — `versions:display-plugin-updates` did not return in a reasonable time and was skipped. See the [Plugin updates — not audited](#plugin-updates--not-audited) section at the end.

---

## Summary

### Counts per bucket

| Bucket | Count |
| --- | --- |
| `safe-patch` (includes "no upgrade available") | 13 |
| `safe-minor` | 0 |
| `review-minor` | 18 |
| `major-skipped` | 2 |
| `major-flagged-as-safe` | 0 |
| `pinned` | 1 |
| `unknown` | 0 |
| **Total distinct rows audited** | **34** |

### Entries needing Igor's per-case decision

**`major-flagged-as-safe`:** none. No major bump was confidently classed as drop-in safe by this audit. Three majors are visible (Flyway 11 → 12 × 2 artifacts, Spring Boot 4 → 4.1.0-RC1 is **minor**, not major) — see `major-skipped` rows.

### Notable patterns

1. **Spring Boot 4.0.5 → 4.1.0-RC1** is the dominant pending upgrade. It pulls 13 starters / managed artifacts forward in lockstep. Currently classed `review-minor` on every row because the available version is a **release candidate, not GA**. If 4.1.0 GA ships before the upgrade brief is scoped, this collapses to a single coordinated `safe-minor` upgrade.
2. **AWS SDK v2** has a two-minor jump (2.42 → 2.44, 28 patch versions). Both s3 and url-connection-client move together — keep versions synchronized in any upgrade.
3. **Flyway 12.x** is a major bump from the parent-BOM-managed 11.14.1. Skipped per brief. Note: this audit did **not** capture the latest 11.x within-major target (only the absolute latest 12.6.1) — see "Known gaps" below.
4. **One documented pin** (`opentelemetry-semconv` @ 1.41.1) is intentional. Reason captured under [Documented pins](#documented-pins).

### Known gaps in this audit

- Only one column was captured for "available version" per dep: the latest absolute. For rows with a major bump on the table (the Flyway pair), the "latest within current major" cell is best-effort (`unknown — only absolute captured`). A follow-up `./mvnw versions:display-dependency-updates -DallowMajorUpdates=false` would close that gap cheaply.
- `versions:display-plugin-updates` did not complete. Plugin section is informational only.

---

## Documented pins

Source: `oglasino-docs/issues.md` (2026-05-15 entry "Backend fails to start: missing OpenTelemetry semconv class") and the comment in `pom.xml` `<dependencyManagement>`.

| Pinned dep | Pinned version | Reason | Source |
| --- | --- | --- | --- |
| `io.opentelemetry.semconv:opentelemetry-semconv` | `1.41.1` | Transitive resolution selects `1.29.0-alpha` via `firebase-admin → google-cloud-storage`, which lacks `io.opentelemetry.semconv.DbAttributes`. The Elasticsearch 9.2.6 Java client's built-in OpenTelemetry instrumentation throws `NoClassDefFoundError` at first ES request → startup fails. | `pom.xml` comment + `issues.md` (status: fixed) + commit `5df54c9` |

**Related concern from the brief — Java 21 preview signature on `main`:** the `main` method's preview-feature signature issue is `fixed` per `issues.md` (2026-05-15 entry, commit raising `main` to `public static`). No direct dep is tied to that fix. The `--enable-preview` compiler/surefire/spotbugs flags in `pom.xml` are plugin-config, not dep-version, concerns — out of scope for this dep audit (and out of scope per the brief's "Plugin configuration changes" exclusion).

---

## Direct dependencies — inventory

Legend:

- **Current** = version resolved today (either explicit in pom or inherited from the `spring-boot-starter-parent` 4.0.5 BOM).
- **Within major** = latest version available within the current major. Where this audit only captured the absolute latest (because the absolute is a major bump), the cell reads `unknown — only absolute captured`.
- **Absolute** = latest version available overall, including major bumps.
- **Bucket** = the brief's safety classification.

### Group 1 — Explicit-version dependencies in `<dependencies>`

| groupId:artifactId | Current | Within major | Absolute | Bucket | Notes |
| --- | --- | --- | --- | --- | --- |
| `io.jsonwebtoken:jjwt-api` | 0.13.0 | 0.13.0 | 0.13.0 | `safe-patch` | No upgrade available. Used for image-pipeline HS256 token signing. |
| `io.jsonwebtoken:jjwt-impl` | 0.13.0 | 0.13.0 | 0.13.0 | `safe-patch` | No upgrade available. Keep version locked with `jjwt-api`. |
| `io.jsonwebtoken:jjwt-jackson` | 0.13.0 | 0.13.0 | 0.13.0 | `safe-patch` | No upgrade available. Keep version locked with `jjwt-api`. |
| `de.huxhorn.sulky:de.huxhorn.sulky.ulid` | 8.3.0 | 8.3.0 | 8.3.0 | `safe-patch` | No upgrade available. Used for jti claims in image-pipeline JWTs. |
| `com.google.firebase:firebase-admin` | 9.8.0 | 9.9.0 | 9.9.0 | `review-minor` | Minor bump. Auth-critical dep (`FirebaseAuthFilter` is the trust boundary). Release notes not reviewed in this audit; verify the auth + token-revocation paths after upgrade. Also note this dep's transitive `google-cloud-storage` is what made the `opentelemetry-semconv` pin necessary — a Firebase upgrade may shift the transitive semconv-alpha and warrant rechecking the pin. |
| `jakarta.servlet:jakarta.servlet-api` | 6.1.0 | 6.2.0-M1 | 6.2.0-M1 | `review-minor` | Available 6.2.0-M1 is a **milestone**, not a stable release. Defer until 6.2.0 GA. Declared `<scope>provided</scope>` — comes from Tomcat at runtime, so the practical effect of bumping is compile-time only. |
| `org.modelmapper:modelmapper` | 3.2.6 | 3.2.6 | 3.2.6 | `safe-patch` | No upgrade available. |
| `org.apache.commons:commons-lang3` | 3.20.0 | 3.20.0 | 3.20.0 | `safe-patch` | No upgrade available. (Artifact is `commons-lang3`; no 4.x branch exists.) |
| `org.apache.commons:commons-collections4` | 4.5.0 | 4.5.0 | 4.5.0 | `safe-patch` | No upgrade available. |
| `software.amazon.awssdk:s3` | 2.42.28 | 2.44.7 | 2.44.7 | `review-minor` | Two-minor jump (2.42 → 2.44; 28 patch versions across). Used against Cloudflare R2 (S3-compatible). Release notes not reviewed in this audit; verify image-pipeline upload + presign paths against R2 after upgrade. |
| `software.amazon.awssdk:url-connection-client` | 2.42.28 | 2.44.7 | 2.44.7 | `review-minor` | Same as `s3`. Must move in lockstep. |
| `org.owasp.encoder:encoder` | 1.4.0 | 1.4.0 | 1.4.0 | `safe-patch` | No upgrade available. |
| `com.bucket4j:bucket4j-core` | 8.10.1 | 8.10.1 | 8.10.1 | `safe-patch` | No upgrade available. Powers `RateLimitFilter`. |
| `com.bucket4j:bucket4j-redis` | 8.10.1 | 8.10.1 | 8.10.1 | `safe-patch` | No upgrade available. Must move in lockstep with `bucket4j-core`. |
| `com.github.zafarkhaja:java-semver` | 0.10.2 | 0.10.2 | 0.10.2 | `safe-patch` | No upgrade available. |

### Group 2 — Managed-version dependencies (version comes from `spring-boot-starter-parent` 4.0.5)

| groupId:artifactId | Current | Within major | Absolute | Bucket | Notes |
| --- | --- | --- | --- | --- | --- |
| `org.springframework.boot:spring-boot-starter-actuator` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Spring Boot 4.1.0-RC1 is **release candidate**, not GA. Defer until 4.1.0 stable. All Spring Boot starters move together via parent BOM. |
| `org.springframework.boot:spring-boot-starter-data-elasticsearch` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as above. Critically — this is the artifact that surfaced the `opentelemetry-semconv` pin (ES 9.2.6 client → semconv class needed). Verify the semconv pin still resolves correctly after bump. |
| `org.springframework.boot:spring-boot-starter-data-redis` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-starter-data-jpa` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-starter-mail` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-starter-security` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. Tightly coupled to `FirebaseAuthFilter` and the trust boundary in conventions Part 11 — verify auth flows after upgrade. |
| `org.springframework.boot:spring-boot-starter-validation` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-starter-web` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-starter-webflux` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-starter-tomcat` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. |
| `org.springframework.boot:spring-boot-devtools` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Same as Spring Boot row. Runtime + optional scope. |
| `org.springframework.boot:spring-boot-flyway` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Spring Boot's Flyway starter. Same RC concern as the rest of Spring Boot. |
| `org.springframework.boot:spring-boot-starter-test` | 4.0.5 | 4.1.0-RC1 | 4.1.0-RC1 | `review-minor` | Test scope. Same Spring Boot RC concern. |
| `org.springframework.security:spring-security-test` | 7.0.4 | 7.1.0-RC1 | 7.1.0-RC1 | `review-minor` | Spring Security 7.1.0-RC1 is **release candidate**. Moves with the Spring Boot upgrade (Spring Boot 4.1 pulls Spring Security 7.1). Test scope. |
| `com.h2database:h2` | (parent-resolved; `display-dependency-updates` reported no newer version) | n/a — at latest | n/a — at latest | `safe-patch` | No upgrade available within the parent BOM resolution. Runtime scope. |
| `org.postgresql:postgresql` | 42.7.10 | 42.7.11 | 42.7.11 | `safe-patch` | Patch-only bump within the current 42.7.x line — JDBC bugfix release. The only true patch-track upgrade with an available target in this audit. Runtime scope. |
| `org.flywaydb:flyway-core` | 11.14.1 | unknown — only absolute captured | 12.6.1 | `major-skipped` | Major bump 11 → 12 is available; brief excludes major bumps. Latest 11.x within-major not captured by this run — a `-DallowMajorUpdates=false` follow-up would fill that cell. Must move in lockstep with `flyway-database-postgresql`. |
| `org.flywaydb:flyway-database-postgresql` | 11.14.1 | unknown — only absolute captured | 12.6.1 | `major-skipped` | Same as `flyway-core`. |

### Group 3 — Pinned entries from `<dependencyManagement>`

| groupId:artifactId | Pinned at | Bucket | Notes |
| --- | --- | --- | --- |
| `io.opentelemetry.semconv:opentelemetry-semconv` | 1.41.1 | `pinned` | See [Documented pins](#documented-pins). Do not propose changing; the pin **is** the fix. If the Firebase Admin upgrade (9.8.0 → 9.9.0) changes the transitive semconv-alpha version, the pin's necessity needs rechecking — but until then, leave alone. |

Inventory total: 15 explicit-version rows (Group 1) + 18 managed-version rows (Group 2) + 1 pinned-mgmt row (Group 3) = **34** distinct dep rows audited. The summary counts at the top of the file are derived from the per-row enumeration below.

#### Row-by-row bucket assignment

- `safe-patch` (13): `jjwt-api`, `jjwt-impl`, `jjwt-jackson`, `de.huxhorn.sulky.ulid`, `modelmapper`, `commons-lang3`, `commons-collections4`, `encoder`, `bucket4j-core`, `bucket4j-redis`, `java-semver`, `h2`, `postgresql` (the only one with an actual patch target available; the rest are "no upgrade available").
- `safe-minor` (0): none.
- `review-minor` (18): `firebase-admin`, `jakarta.servlet-api`, `s3`, `url-connection-client`, `spring-boot-starter-actuator`, `spring-boot-starter-data-elasticsearch`, `spring-boot-starter-data-redis`, `spring-boot-starter-data-jpa`, `spring-boot-starter-mail`, `spring-boot-starter-security`, `spring-boot-starter-validation`, `spring-boot-starter-web`, `spring-boot-starter-webflux`, `spring-boot-starter-tomcat`, `spring-boot-devtools`, `spring-boot-flyway`, `spring-boot-starter-test`, `spring-security-test`.
- `major-skipped` (2): `flyway-core`, `flyway-database-postgresql`.
- `major-flagged-as-safe` (0): none.
- `pinned` (1): `opentelemetry-semconv`.
- `unknown` (0): none.

---

## Plugin updates — not audited

`./mvnw versions:display-plugin-updates` did not complete within a reasonable time during this audit and was abandoned per Igor's instruction. Plugin upgrade analysis is deferred to a follow-up read-only session.

The plugins below are declared in `pom.xml`'s `<build><plugins>` section, with their **declared versions only**. No upgrade target is asserted; no bucket is assigned.

| Plugin | Declared version | Source | Notes |
| --- | --- | --- | --- |
| `org.springframework.boot:spring-boot-maven-plugin` | (managed by parent — 4.0.5) | `spring-boot-starter-parent` 4.0.5 | Configured with `<mainClass>com.memento.tech.oglasino.OglasinoApplication</mainClass>`. **Side note from the captured `display-dependency-updates` output:** this plugin had an update reported under "pluginManagement of plugins" — 4.0.5 → 4.1.0-RC1, moving in lockstep with the Spring Boot upgrade. Same RC concern as the dep rows. |
| `org.apache.maven.plugins:maven-compiler-plugin` | (managed by parent) | `spring-boot-starter-parent` 4.0.5 | Configured with `<source>21</source>`, `<target>21</target>`, `--enable-preview`. The preview flag is required by the project's Java 21 features (see `issues.md` `Main method uses Java 21 preview signature` entry). |
| `org.apache.maven.plugins:maven-surefire-plugin` | (managed by parent) | `spring-boot-starter-parent` 4.0.5 | Configured with `--enable-preview`. |
| `com.diffplug.spotless:spotless-maven-plugin` | 2.43.0 (explicit) | `pom.xml` line 252 | Embeds `googleJavaFormat` tool version 1.22.0 (inline config). |
| `com.github.spotbugs:spotbugs-maven-plugin` | 4.8.6.6 (explicit) | `pom.xml` line 287 | Configured with `--enable-preview` jvm arg, `<effort>Max</effort>`, `<threshold>High</threshold>`. |

No `<pluginManagement>` section exists in this repo's `pom.xml` directly (plugin versions come from the parent BOM).

### Follow-up suggestion

A targeted read-only session can run `./mvnw versions:display-plugin-updates` in isolation (e.g., redirecting output to a file from the start, with `-q` to reduce noise, and a sane timeout) to produce the plugin-side equivalent of this dep audit. The dep goal (`display-dependency-updates`) completed in ~29s on this machine; whatever caused the plugin goal to hang is plugin-specific to the Maven environment and worth diagnosing separately.

---

## Closing

This audit is the input for the upgrade brief. Recommended single-PR upgrades (lowest-risk, highest-leverage) for Igor's per-case consideration in scoping order:

1. **`postgresql` 42.7.10 → 42.7.11** — pure JDBC patch. The one truly drop-in bump on the table.
2. **Wait for Spring Boot 4.1.0 GA**, then move all 14 Spring-Boot/Security rows in one coordinated PR. While 4.1.0 is RC, this stays in `review-minor` and should not ship.
3. **`firebase-admin` 9.8.0 → 9.9.0** — small minor, but auth-critical. Verify the OpenTelemetry semconv pin is still load-bearing after the bump (the pin exists because of a Firebase transitive chain).
4. **`software.amazon.awssdk` 2.42.28 → 2.44.7** (both s3 and url-connection-client) — two-minor jump. Verify against R2 specifically; AWS SDK release notes worth scanning.
5. **`jakarta.servlet-api`** — wait for 6.2.0 GA. 6.2.0-M1 is not eligible.
6. **Flyway 11 → 12** — major. Defer entirely; not on the table per brief.
