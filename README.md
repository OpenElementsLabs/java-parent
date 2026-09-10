# Java Parent

Maven parent POM for Java projects by [Open Elements](https://open-elements.com).

Inherit from this POM to get a consistent, modern Java build out of the box:
pinned plugin versions, managed dependency BOMs, code formatting, an SBOM,
and a complete publish-to-Maven-Central release pipeline — without copying
boilerplate into every project.

## Coordinates

```xml
<parent>
    <groupId>com.open-elements</groupId>
    <artifactId>java-parent</artifactId>
    <version>1.0.0</version>
</parent>
```

Released artifacts are published to [Maven Central](https://central.sonatype.com/artifact/com.open-elements/java-parent);
`-SNAPSHOT` builds are published to the Central Portal snapshot repository.

## Requirements

| Tool  | Version    | Enforced by                                                   |
|-------|------------|--------------------------------------------------------------|
| Java  | 21         | `maven-enforcer-plugin` (`requireJavaVersion`)               |
| Maven | 3.9.11+    | `maven-enforcer-plugin` (`requireMavenVersion`) + wrapper    |

The build fails fast in the `validate` phase if the local toolchain does not
meet these requirements. Both minimums can be raised by a child project by
overriding the `enforcer.requiredJavaVersion` / `enforcer.requiredMavenVersion`
properties.

A Maven Wrapper (`./mvnw`) pinned to 3.9.11 is included, so no local Maven
installation is required.

## What you get

### Managed dependency versions (BOM imports)

Import-scoped BOMs so child projects can declare these dependencies **without a
`<version>`**:

- **Spring Boot** — `spring-boot-dependencies` (`3.5.14`)
- **Testcontainers** — `testcontainers-bom` (`2.0.5`)

### Pinned plugin versions

All common build plugins are version-managed in `<pluginManagement>`, so child
builds are reproducible and free of "you should pin this plugin" warnings:

`maven-resources`, `maven-compiler`, `maven-surefire`, `maven-javadoc`,
`maven-source`, `maven-gpg`, `maven-jar`, `maven-deploy`, `maven-clean`,
`maven-enforcer`, `cyclonedx`, `jreleaser`, `versions`, `spotless`.

### Build conventions

- **Java 21**, source/target via `maven.compiler.*`.
- **UTF-8** for sources and reporting.
- **`-parameters`** compiler flag (parameter names retained — useful for
  frameworks like Spring and Jackson).
- **Reproducible builds** via an inherited `project.build.outputTimestamp`
  (see [Reproducible builds](#reproducible-builds)).
- **Code formatting** via Spotless using
  [Google Java Format](https://github.com/google/google-java-format).
- **Surefire** pre-configured with the `--add-opens` flags commonly needed by
  reflection-based test/mocking libraries.
- **Toolchain enforcement** (see [Requirements](#requirements)).

## Reproducible builds

Building the same source twice yields **byte-identical artifacts** — jar, sources jar,
javadoc jar and the CycloneDX SBOM.

This works because `java-parent` declares a fixed timestamp that every child project
inherits:

```xml
<properties>
    <project.build.outputTimestamp>2026-08-28T00:00:00Z</project.build.outputTimestamp>
</properties>
```

Child projects **set nothing**. They inherit the value by pinning a `java-parent`
version, and `release.sh` updates it whenever a new `java-parent` release is cut.

### What is promised

> The same source, built with the **same toolchain**, produces byte-identical artifacts —
> regardless of when or where the build runs.

"Same toolchain" is part of the promise, not a footnote. Reproducibility across
*differing* JDK patch versions, Maven versions, operating systems or locales has **not
been measured** and is not claimed. Javadoc output in particular is known to vary
between JDK builds. Use the Java and Maven versions this project enforces.

### Verifying a release yourself

No build flags and no insider knowledge are needed. Apart from its deployment
arguments, this is the same command CI runs:

```bash
git checkout vA.B.C          # any release that carries the timestamp property
./mvnw -Pfull-build clean verify
```

Then compare the result against the artifacts published on Maven Central, for example
with `sha256sum`. They must match.

> **Do not pass `-Dproject.build.outputTimestamp`, and do not override the property in
> a child POM.** Both take precedence over the inherited value (`-D` beats a child POM,
> which beats the parent), so either one silently produces artifacts that nobody else
> can reproduce.

### The timestamp is not a build time

`project.build.outputTimestamp` records **which `java-parent` release an artifact was
built against**. It is deliberately not the time the build ran — a real build time
cannot be reproduced, which is the whole point.

If an application needs to answer *"which state is this?"*, use the Git metadata that
the `full-build` profile writes into every jar's `META-INF/MANIFEST.MF`:

| Manifest entry | Meaning |
|----------------|---------|
| `Git-Commit-Time` | When the source state came into being (UTC, commit-derived) |
| `Git-Commit` | Abbreviated commit id |
| `Git-Branch` | Branch the build came from |
| `Git-Tag` | Tags pointing at the commit |
| `Git-Dirty` | Whether the working tree had uncommitted changes |

`Git-Commit-Time` is the correct replacement for any `buildTime` / `build.time` field.
Such a field sourced from `project.build.outputTimestamp` would be misleading and must
not be introduced.

When the build runs outside a Git checkout — for example from a published source
archive — these entries are present but **empty**. The build deliberately does not fail
(`failOnNoGitDirectory` is `false`) so that reproducing from sources stays possible, and
the empty values are themselves deterministic.

### Known limitations

Tracked in [`docs/TODO.md`](docs/TODO.md):

- No automated check guards reproducibility — a plugin upgrade could reintroduce
  non-determinism unnoticed.
- Cross-toolchain reproducibility is unmeasured — see
  [What is promised](#what-is-promised).
- Snapshot builds are **not** reproducible, by decision. A project pinning a
  `-SNAPSHOT` parent inherits a value that moves when the snapshot is republished.
- A project pinning a `java-parent` version older than the first release carrying the
  property **inherits nothing** and stays non-reproducible. Raising the parent version
  is what actually switches this on for a downstream project.

## Common commands

```bash
# Build and test
./mvnw clean verify

# Apply code formatting / check formatting
./mvnw spotless:apply
./mvnw spotless:check

# Full build: also attaches Javadoc jar, sources jar and a CycloneDX SBOM
./mvnw -Pfull-build clean verify

# Check for newer dependency, plugin and property versions
./check-dependencies.sh
```

### Checking for updates

`check-dependencies.sh` runs the
[`versions-maven-plugin`](https://www.mojohaus.org/versions/) and writes three
reports to `target/`:

- `dependency-updates.txt`
- `plugin-updates.txt`
- `property-updates.txt`

## Build profiles

| Profile          | Purpose                                                                                          |
|------------------|--------------------------------------------------------------------------------------------------|
| `full-build`     | Attaches the Javadoc jar, sources jar, and generates a CycloneDX SBOM. Used for releases & CI.   |
| `deploy-release` | Signs artifacts (GPG) and publishes to Maven Central + creates the GitHub release via JReleaser. |

## Releasing

Releases are cut with `release.sh` and finished by CI — the script only prepares
git state, it never deploys:

```bash
./release.sh <release-version> <next-snapshot-version>
# e.g.
./release.sh 1.1.0 1.2.0-SNAPSHOT
```

The script:

1. Sets the release version in the POM.
2. Pins `project.build.outputTimestamp` to the release date and verifies the
   rewrite took effect — a stale timestamp would publish artifacts that no
   rebuild could match, so the release aborts rather than continuing.
3. Runs `./mvnw -Pfull-build clean verify` locally — so a broken build, missing
   Javadoc link, or SBOM error fails **here**, not after the tag is pushed.
   Because the timestamp is already pinned, these are the bytes CI will publish.
4. Best-effort generates upgrade documentation under `docs/releases/`
   (requires the Claude Code CLI; skipped with a warning if absent).
5. Commits version and timestamp together, tags `vA.B.C`, pushes, then bumps to
   the next `-SNAPSHOT` (leaving the timestamp at the release date).

Pushing the `vA.B.C` tag triggers the release workflow, which verifies the POM
version matches the tag, stages artifacts, and publishes to Maven Central while
creating the GitHub release.

## Continuous integration

| Workflow       | Trigger                 | What it does                                                          |
|----------------|-------------------------|----------------------------------------------------------------------|
| `build.yml`    | Pull requests to `main` | `./mvnw clean verify`                                                 |
| `snapshot.yml` | Push to `main`          | Publishes `-SNAPSHOT` artifacts to the Central Portal snapshot repo.  |
| `release.yml`  | Push of a `v*.*.*` tag  | Builds, signs, and deploys to Maven Central; creates a GitHub release.|

## License

Licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
