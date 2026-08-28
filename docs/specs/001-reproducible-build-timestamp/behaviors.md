# Behaviors: Reproducible build timestamp

## Timestamp inheritance and determinism

### A child project builds byte-identically twice

- **Given** a project inheriting from a `java-parent` version that carries the
  `project.build.outputTimestamp` literal, and setting no timestamp property itself
- **When** `./mvnw -Pfull-build clean verify` is run twice, with time passing between
  the runs and no additional flags
- **Then** the jar, sources jar, javadoc jar, `bom.json` and `bom.xml` from both runs
  are byte-identical

### The parent's own build is deterministic

- **Given** the modified `java-parent` (`packaging=pom`)
- **When** `./mvnw -Pfull-build clean verify` is run twice
- **Then** `bom.json` and `bom.xml` are byte-identical across both runs

### A child inherits the parent's value without declaring anything

- **Given** a child POM with no `project.build.outputTimestamp` in its `<properties>`
- **When** the child is built
- **Then** every archive entry and manifest carries the timestamp declared in the
  parent POM

### A child pinning an older parent is unaffected

- **Given** a project pinning `java-parent` 1.2.1, which predates this change
- **When** the project is built twice
- **Then** the artifacts still differ between runs, because nothing is inherited
- **And** this is expected: the project becomes reproducible only after its parent
  version is upgraded

## Override precedence

### A command-line override wins over the inherited value

- **Given** a child inheriting a timestamp from the parent
- **When** the build is run with `-Dproject.build.outputTimestamp=<other value>`
- **Then** the artifacts are stamped with the command-line value, not the inherited one
- **And** an external verifier running without that flag gets different bytes

### A child POM property wins over the inherited value

- **Given** a child declaring `project.build.outputTimestamp` in its own `<properties>`
- **When** the child is built without command-line flags
- **Then** the artifacts are stamped with the child's value, not the parent's

## Third-party verification

### A verifier reproduces a release with no insider knowledge

- **Given** a released `java-parent` version and its artifacts published to Maven Central
- **When** a third party checks out the corresponding `vA.B.C` tag and runs
  `./mvnw -Pfull-build clean verify` with no additional arguments
- **Then** the produced artifacts are byte-identical to the published ones

### A build without a Git directory still succeeds and is stamped

- **Given** a source archive extracted without a `.git` directory
- **When** the project is built with `-Pfull-build`
- **Then** the build succeeds, because `failOnNoGitDirectory` is `false`
- **And** the artifacts carry the same `project.build.outputTimestamp` as a build from
  a Git checkout

### Locale and timezone of the build machine do not affect the stamp

- **Given** two machines with different system timezones and locales, on the same JDK
  and Maven version
- **When** the same source is built on both
- **Then** the archive entry timestamps are identical, because the property is an
  absolute UTC instant

## Release process

### Cutting a release rewrites the timestamp to the release date

- **Given** a `java-parent` working tree at `1.3.0-SNAPSHOT` with a timestamp from a
  previous release
- **When** `./release.sh 1.4.0 1.5.0-SNAPSHOT` is run
- **Then** `pom.xml` contains `project.build.outputTimestamp` set to the current UTC
  date at midnight, in the form `YYYY-MM-DDT00:00:00Z`

### The release commit carries both the version and the timestamp

- **Given** a release cut with `release.sh`
- **When** the `Version 1.4.0` commit is inspected
- **Then** it contains both the version change and the timestamp change
- **And** the `v1.4.0` tag points at that commit, so the tag is self-sufficient for a
  verifier

### The timestamp is written before the verification build

- **Given** `release.sh` is cutting a release
- **When** the local `./mvnw -Pfull-build clean verify` step runs
- **Then** it already builds with the new timestamp, so the locally verified bytes
  match what CI will publish

### Bumping to the next snapshot leaves the timestamp untouched

- **Given** a completed release at `1.4.0` with timestamp `2026-09-14T00:00:00Z`
- **When** `release.sh` bumps the version to `1.5.0-SNAPSHOT`
- **Then** `project.build.outputTimestamp` still reads `2026-09-14T00:00:00Z`
- **And** snapshot builds until the next release carry that date

### Two releases on the same day share a timestamp

- **Given** `1.4.0` was released earlier today
- **When** `1.4.1` is released on the same UTC day
- **Then** both carry the identical timestamp value
- **And** neither build fails, since the values need only be deterministic, not unique

### A failed property rewrite aborts the release

- **Given** `versions:set-property` fails, for example because the property is missing
  from `pom.xml`
- **When** `release.sh` reaches that step
- **Then** the script exits non-zero under `set -e`
- **And** no tag is pushed and no release is published with a stale timestamp

## Continuous integration

### The release workflow builds without a timestamp flag

- **Given** the updated `.github/workflows/release.yml`
- **When** a `vA.B.C` tag is pushed and the workflow builds
- **Then** no `-Dproject.build.outputTimestamp` argument is passed
- **And** the published artifacts carry the value from the POM

### The snapshot workflow builds without a timestamp flag

- **Given** the updated `.github/workflows/snapshot.yml`
- **When** a commit is pushed to `main`
- **Then** no `-Dproject.build.outputTimestamp` argument is passed
- **And** the snapshot artifacts carry the last release's timestamp from the POM

### CI and a third party run the same command

- **Given** the updated workflows
- **When** the CI build command is compared to the command documented in the README
- **Then** they differ only in deployment arguments, never in anything affecting the
  produced bytes

## Snapshot parents — accepted non-reproducibility

### A child on a SNAPSHOT parent inherits a moving value

- **Given** a child pinning `java-parent:1.4.0-SNAPSHOT`
- **When** the parent snapshot is republished with a different timestamp and the child
  is rebuilt from unchanged source
- **Then** the child's artifacts differ from the earlier build
- **And** this is accepted: snapshots are not a reproducibility target

## Build metadata

### Git-Commit-Time remains available and distinct

- **Given** a jar built with `-Pfull-build` from a Git checkout
- **When** its `META-INF/MANIFEST.MF` is read
- **Then** `Git-Commit-Time` is present in UTC and describes when the source state came
  into being
- **And** it is independent of `project.build.outputTimestamp`, which describes the
  parent release the artifact was built against

### No build-time field is introduced

- **Given** the completed change
- **When** the parent's configuration is inspected
- **Then** no property, manifest entry or generated file presents
  `project.build.outputTimestamp` as the time the build ran

## Documentation

### The README states only the measured claim

- **Given** the updated `README.md`
- **When** the reproducibility section is read
- **Then** it promises byte-identical output for the same source built with the **same
  toolchain**
- **And** it does not claim reproducibility across differing JDK patch versions, which
  has not been measured

### Deferred work is discoverable

- **Given** the completed change
- **When** `docs/TODO.md` is read
- **Then** it records the absence of an automated check, the unmeasured cross-toolchain
  behaviour, and the pending parent upgrade in consumer projects
