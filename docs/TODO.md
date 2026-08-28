# TODO

## No automated check guards build reproducibility

Nothing verifies that artifacts stay byte-identical. Spec 001 pins
`project.build.outputTimestamp` in the parent POM, and a one-time manual double-build
confirmed the effect — but that run is never repeated. A plugin upgrade, a new plugin
in the `full-build` profile, or a generated resource carrying a timestamp would
silently reintroduce non-determinism and nothing would fail.

What a check needs to look like: `java-parent` has `packaging=pom` and produces only
`bom.xml`/`bom.json` on its own, so a double-build of the parent alone exercises
almost nothing. A meaningful check requires a small **fixture child project** in the
repository that inherits the parent and contains at least one Java source file, built
twice with `-Pfull-build clean verify` and compared across jar, sources jar, javadoc
jar and both SBOM files. Wire it as a separate CI job so a failure is easy to
diagnose. See the `reproducible-builds-check` skill for the script shape.

**Context:** Deliberately deferred during the grill session for spec 001. A check
scoped to the parent alone was considered and rejected as near-worthless; building a
fixture child was judged larger than the spec's intended scope.

## Cross-toolchain reproducibility is unmeasured

The measurement behind spec 001 compared two builds on one machine — same JDK patch
version, same path, same locale, same timezone. It proves reproducibility *over time*,
not *across environments*. Javadoc HTML in particular is known to vary between JDK
builds, so an artifact built with Temurin 21.0.9 may not match one built with 21.0.5.

Until this is measured, the README may promise only "same source + same toolchain".
Establishing the broader claim means: building the same source on differing JDK patch
versions, Maven versions, operating systems and locales, diffing the results, and
either pinning the toolchain precisely or documenting the tolerance. Publishing a
`.buildinfo` file via `maven-artifact-plugin` (`artifact:buildinfo`) is the standard
way to record the toolchain a release was built with, and is what the
`reproducible-central` project consumes.

**Context:** Surfaced in the grill session for spec 001 and explicitly deferred.

**Prerequisite:** Spec 001 (the timestamp must be fixed before toolchain variance is
isolable).

## Consumer projects still pin the pre-reproducibility parent

`spring-services` and `open-crm/backend` pin `java-parent` 1.2.1, which carries no
timestamp property. Spec 001 prepares the parent but changes nothing downstream —
both remain non-reproducible until their parent version is raised to the first release
containing the property. Nothing is actually reproducible in practice until this
happens.

Also worth checking during that upgrade: whether any downstream project sets
`project.build.outputTimestamp` itself or passes `-Dproject.build.outputTimestamp` in
its own CI, since either defeats external verification.

**Context:** Explicitly placed outside spec 001 during the grill session.

**Prerequisite:** Spec 001 released as a `java-parent` version.

## Line endings are not pinned

The repository has neither `.gitattributes` nor `.editorconfig`. Source files checked
out on Windows can carry CRLF, which changes the bytes inside a published sources jar
and breaks reproducibility for anyone building on a differently configured machine.
Add `.gitattributes` enforcing LF for source and resource files, plus a matching
`.editorconfig`, and consider recommending both to child projects.

**Context:** Identified while auditing the parent against the
`reproducible-builds-check` skill during spec 001; out of the chosen scope.

## Release builds do not reject SNAPSHOT or ranged dependencies

The enforcer currently checks only the Maven and Java versions. A release could
resolve a SNAPSHOT dependency or a version range and still publish, which makes the
result unreproducible regardless of the timestamp. Add `requireReleaseDeps` (active
for release builds only, so development branches stay workable) and consider
`requireUpperBoundDeps` to catch transitive version skew.

**Context:** Identified while auditing the parent against the
`reproducible-builds-check` skill during spec 001; out of the chosen scope.

## No buildinfo is published for released artifacts

Maven Central reproducibility verification, and the `reproducible-central` rebuild
project, consume a `.buildinfo` file that records the artifacts' checksums together
with the JDK, Maven and OS used. `maven-artifact-plugin` produces it via
`artifact:buildinfo`, and `artifact:check-buildplan` additionally flags plugins in the
build plan that are known not to support reproducible output. Adding both would turn
the reproducibility claim from something asserted into something an outside party can
check mechanically.

**Context:** Surfaced in the grill session for spec 001 while establishing what an
external verifier actually needs; deferred with the rest of the verification tooling.

**Prerequisite:** Spec 001.
