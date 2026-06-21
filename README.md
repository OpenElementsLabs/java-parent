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
- **Code formatting** via Spotless using
  [Google Java Format](https://github.com/google/google-java-format).
- **Surefire** pre-configured with the `--add-opens` flags commonly needed by
  reflection-based test/mocking libraries.
- **Toolchain enforcement** (see [Requirements](#requirements)).

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
2. Runs `./mvnw -Pfull-build clean verify` locally — so a broken build, missing
   Javadoc link, or SBOM error fails **here**, not after the tag is pushed.
3. Best-effort generates upgrade documentation under `docs/releases/`
   (requires the Claude Code CLI; skipped with a warning if absent).
4. Commits, tags `vA.B.C`, pushes, then bumps to the next `-SNAPSHOT`.

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
