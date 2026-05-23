# OpenTelemetry semconv — dependency diagnosis

**Date:** 2026-05-15
**Repo:** oglasino-backend
**Issue:** `NoClassDefFoundError: io/opentelemetry/semconv/DbAttributes` at startup

## Command

```
./mvnw dependency:tree -Dverbose '-Dincludes=io.opentelemetry*:*'
```

## Relevant tree slice (verbatim)

```
[INFO] com.memento.tech:oglasino:jar:0.0.1-SNAPSHOT
[INFO] +- org.springframework.boot:spring-boot-starter-data-elasticsearch:jar:4.0.5:compile
[INFO] |  \- org.springframework.boot:spring-boot-data-elasticsearch:jar:4.0.5:compile (version managed from 4.0.5)
[INFO] |     \- org.springframework.data:spring-data-elasticsearch:jar:6.0.4:compile (version managed from 6.0.4)
[INFO] |        \- co.elastic.clients:elasticsearch-java:jar:9.2.6:compile (version managed from 9.2.6)
[INFO] |           +- (io.opentelemetry:opentelemetry-api:jar:1.55.0:runtime - version managed from 1.37.0; omitted for duplicate)
[INFO] |           \- (io.opentelemetry.semconv:opentelemetry-semconv:jar:1.37.0:runtime - omitted for conflict with 1.29.0-alpha)
[INFO] \- com.google.firebase:firebase-admin:jar:9.8.0:compile
[INFO]    +- com.google.cloud:google-cloud-storage:jar:2.63.0:compile
[INFO]    |  +- io.opentelemetry:opentelemetry-context:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  |  \- io.opentelemetry:opentelemetry-common:jar:1.55.0:compile (version managed from 1.55.0)
[INFO]    |  +- io.opentelemetry:opentelemetry-sdk:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  +- io.opentelemetry:opentelemetry-sdk-trace:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  +- io.opentelemetry:opentelemetry-sdk-logs:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  +- io.opentelemetry:opentelemetry-api:jar:1.55.0:compile (version managed from 1.51.0; scope not updated to compile)
[INFO]    |  +- io.opentelemetry:opentelemetry-sdk-metrics:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  +- io.opentelemetry:opentelemetry-sdk-common:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  +- io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi:jar:1.55.0:compile (version managed from 1.51.0)
[INFO]    |  +- io.opentelemetry.semconv:opentelemetry-semconv:jar:1.29.0-alpha:compile (scope not updated to compile)
[INFO]    |  \- io.opentelemetry.contrib:opentelemetry-gcp-resources:jar:1.37.0-alpha:compile
[INFO]    \- com.google.cloud:google-cloud-firestore:jar:3.37.0:compile
[INFO]       +- (io.opentelemetry:opentelemetry-api:jar:1.55.0:compile - version managed from 1.51.0; omitted for duplicate)
[INFO]       +- (io.opentelemetry:opentelemetry-context:jar:1.55.0:compile - version managed from 1.51.0; omitted for duplicate)
[INFO]       +- io.opentelemetry.instrumentation:opentelemetry-grpc-1.6:jar:2.1.0-alpha:compile
[INFO]       +- io.opentelemetry.instrumentation:opentelemetry-instrumentation-api:jar:2.1.0:compile
[INFO]       +- io.opentelemetry:opentelemetry-extension-incubator:jar:1.35.0-alpha:runtime
[INFO]       +- (io.opentelemetry.semconv:opentelemetry-semconv:jar:1.29.0-alpha:compile - omitted for duplicate)
[INFO]       +- io.opentelemetry.instrumentation:opentelemetry-instrumentation-api-incubator:jar:2.1.0-alpha:compile
```

## What this shows

- `io.opentelemetry.semconv:opentelemetry-semconv` **is** present, but resolved to **`1.29.0-alpha`** — pulled in transitively by `com.google.cloud:google-cloud-storage:2.63.0` (which sits under `firebase-admin:9.8.0`).
- The Elasticsearch Java client (`co.elastic.clients:elasticsearch-java:9.2.6`) requests **`1.37.0`** (stable track). Maven dropped that request with: `omitted for conflict with 1.29.0-alpha`. The nearer dependency (firebase-admin's chain) won the conflict resolution.
- All other `io.opentelemetry:*` artifacts are aligned at **`1.55.0`** by Spring Boot 4.0.5's parent BOM. The semconv artifact is **not** managed by that BOM (it lives on its own version line: `1.27.0` / `1.28.0` / ... / `1.41.1`).

## Why this breaks startup

The `opentelemetry-semconv` artifact has two version tracks:

- **`*-alpha`** (e.g. `1.29.0-alpha`) — the historical incubating-only track. It does **not** contain the stable `io.opentelemetry.semconv.DbAttributes` class.
- **Stable** (e.g. `1.30.0`, `1.31.0`, ..., `1.41.1`) — introduced once stable semantic conventions for database attributes were finalized. Contains `DbAttributes`.

The Elasticsearch Java 9.x client's built-in OpenTelemetry instrumentation calls `io.opentelemetry.semconv.DbAttributes` at runtime. With `1.29.0-alpha` winning the classpath, that class is absent and the JVM throws `NoClassDefFoundError`, which (being an `Error`, not an `Exception`) bypasses the `catch (Exception)` in `GlobalIndexerService.onAppReady` and propagates up, failing `ApplicationReadyEvent` and shutting Boot down.

## Latest stable version of opentelemetry-semconv

Per `https://repo1.maven.org/maven2/io/opentelemetry/semconv/opentelemetry-semconv/maven-metadata.xml` (fetched 2026-05-15):

- `<latest>` and `<release>` both report **`1.41.1`**.
- Stable versions (no `-alpha`, no `-rc`): `1.30.0`, `1.31.0`, `1.32.0`, `1.34.0`, `1.36.0`, `1.37.0`, `1.38.0`, `1.39.0`, `1.40.0`, `1.41.0`, `1.41.1`.

## Fix applied

This matches the brief's option 2 ("semconv is present at a version older than 1.27" — adapted: numerically newer but on the alpha-only track that predates `DbAttributes`, same functional result). Picked option 2 over option 3 (full OpenTelemetry BOM) because the conflict is isolated to a single artifact whose versioning is independent of the `io.opentelemetry:*` API/SDK line that Spring Boot already manages at `1.55.0`; a BOM would force a coordinated re-alignment that this defect does not require.

Added a `<dependencyManagement>` entry in `pom.xml` pinning `io.opentelemetry.semconv:opentelemetry-semconv` to **`1.41.1`** (latest stable per Maven Central; not a snapshot or RC).
