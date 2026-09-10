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

## ~~Line endings are not pinned~~

Promoted to spec [`002-pinned-line-endings`](specs/002-pinned-line-endings/design.md).

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

## Windows line-ending behaviour is unverified on real Windows

Spec 002 pins `* text=auto eol=lf`, which matters on exactly one platform: Windows,
where it has to beat the `core.autocrlf=true` default. During implementation the
mechanism was measured — checking files out with `core.autocrlf=true` and with
`core.eol=crlf` forced still yields LF for text files and CRLF for `*.cmd`, so the
attribute wins as specified. Git's conversion logic is the same implementation
everywhere, so this covers the decisive behaviour.

What is still unobserved is the platform itself: no build has run on a real Windows
machine. A `windows-latest` CI job asserting that `git ls-files --eol` reports no
`w/crlf` outside `*.cmd`/`*.bat` would close that remainder for roughly fifteen lines.

**Context:** A CI guard was offered during the grill session for spec 002 and
declined, consistent with spec 001 shipping without automated verification. The
maintainer develops on macOS.
## `claude-base` conventions need two fixes

The org-wide convention documents live in `claude-base` and are vendored into each
repository under `.claude/skills/`. Two problems surfaced while writing spec 002, and
both must be fixed at the source rather than in the vendored copy, which would drift:

1. **`.gitattributes` is not a documented requirement.** `references/repo-setup.md`
   makes `.editorconfig` mandatory and never mentions `.gitattributes`. Empirically
   1 of 26 Open Elements repositories has one. The block from spec 002 is a ready
   template.
2. **The standard `.editorconfig` contradicts Google Java Format.**
   `references/editorconfig.md` specifies `indent_size = 4` globally and
   `max_line_length = 120` for `[*.java]`, while `googleJavaFormat` — which
   `java-parent` enforces via Spotless for every child — formats with 2 spaces and
   100 columns. Any Java project following both gets an IDE that fights the
   formatter. Spec 002 corrects the block locally; the standard itself is still
   wrong.

**Context:** Both identified during the grill session for spec 002 and deliberately
kept out of that spec, because editing the vendored skill copies creates drift
against `claude-base`.

## `swagger-bom` pins the javax and JAX-RS Swagger line too

Spec 003 imports `io.swagger.core.v3:swagger-bom` to keep the jakarta trio
(`swagger-annotations-jakarta`, `swagger-models-jakarta`, `swagger-core-jakarta`)
uniform. The BOM manages all 14 Swagger coordinates, so the non-jakarta line and the
JAX-RS artifacts are pinned for every child as a side effect — at a version selected
for springdoc compatibility, not for that child. It is silent: the build succeeds
either way.

Worth deciding whether that is wanted. The alternatives are pinning only the three
jakarta artifacts by hand (an artifact list that goes stale when springdoc adds a
Swagger module) or leaving it as is and documenting the reach.

**Context:** Identified while choosing between `swagger-bom` and hand-written pins
during spec 003; the wider reach was accepted to avoid maintaining an artifact list.

## The springdoc/Swagger lockstep is documented, not enforced

Spec 003 couples three version properties — `springdoc.version`, `swagger.version`,
`swagger-ui.version` — that must always be bumped together, and enforces this with a
comment and a README section. Nothing checks it. Bumping springdoc alone would
publish a split Swagger stack to every child at once, and the parent's own build
would not notice, because it has no Java sources and never exercises the stack.

Two ways out, both rejected in spec 003 for stated reasons: import
`org.springdoc:springdoc-openapi` (the aggregator) as a BOM so one property pulls the
whole consistent set — rejected because it is springdoc's internal build structure
rather than a published contract, and it also manages jjwt, scalar and
spring-cloud-function — or add a check that reads `<swagger-api.version>` from the
springdoc POM being used and fails when the parent disagrees.

**Context:** Raised during the discussion for spec 003 and accepted as a documented
maintenance rule; this entry records the residual risk.

## Downstream: libraries pinning Swagger directly are not fixed

Spec 003 fixes the case where a library *transitively* contributes an old Swagger
version. It cannot fix a project that declares a Swagger artifact **directly with an
explicit version** — a direct declaration beats `dependencyManagement`. Those
projects keep whatever they had, silently, and see neither the error nor the fix.

`spring-services-core` is the known case: it pins `swagger-annotations-jakarta`
2.2.29 and is the source of the mediation conflict that motivated spec 003. Dropping
that pin in favour of the managed version would remove the conflict at its origin,
and the library should be rebuilt and tested once against 2.2.47.

**Context:** Surfaced while establishing the precise reach of parent-level
`dependencyManagement` during spec 003.

**Prerequisite:** Spec 003 released as a `java-parent` version.

