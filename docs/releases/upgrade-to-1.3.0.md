# Upgrade prompt: `com.open-elements:java-parent` 1.2.1 → 1.3.0

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its
`<parent>`, moving from `1.2.1` to `1.3.0`. This is a **minor release with three
breaking-light changes and several additive ones**. There are **no API/code changes**
and **no source edits are required**: consumer source compiles unchanged.

What can change under you: **artifact bytes** (every archive entry now carries a fixed
timestamp instead of the build time), **`spotless:check` results** (the formatter now
writes LF unconditionally), and **the resolved Swagger version** (the OpenAPI stack is
now version-managed, which upgrades a transitively resolved `2.2.29` to `2.2.47`).
Spring Boot stays `3.5.14` and Testcontainers stays `2.0.5`.

Your job is to bump the `<parent>` version, deal with the three breaking-light items
if they apply, and rebuild. Apply exactly the changes below and nothing outside this
scope.

### What changed in 1.3.0

#### Dependencies

Bump only the `<parent>` version of `com.open-elements:java-parent` to `1.3.0`.
**Do not bump anything else**: `spring-boot-dependencies` stays at `3.5.14` and
`testcontainers-bom` stays at `2.0.5`. Do not bump any Maven / JReleaser / CycloneDX /
Spotless / git-commit-id / pomchecker plugin version in the consumer — the plugin set
is unchanged from `1.2.1`.

The Swagger and springdoc versions the consumer resolves **may change without any
consumer edit** — see the OpenAPI section below. That is the intended effect of this
release, not something to counteract.

#### Breaking-light: artifacts now carry a fixed timestamp instead of the build time

The parent's `<properties>` now sets:

```xml
<!-- new in 1.3.0 (parent <properties>) -->
<project.build.outputTimestamp>2026-09-10T00:00:00Z</project.build.outputTimestamp>
```

The exact literal is the date the `1.3.0` release was cut; read it from the published
`java-parent-1.3.0.pom` if you need the precise value. Every child inherits it, and
`maven-jar-plugin`, `maven-source-plugin`, `maven-javadoc-plugin` and
`cyclonedx-maven-plugin` all use it for archive entry timestamps and manifests.

Under `1.2.1` no such property existed, so those timestamps were the wall-clock time
of the build and two builds of the same source produced different bytes. Under `1.3.0`
they are deterministic: **the same source built with the same toolchain produces
byte-identical artifacts**, and a third party can rebuild a release from a tag with a
plain `mvn -Pfull-build clean verify` and no extra flags.

What this means for the consumer:

- **No `pom.xml` edit is required.** The property is inherited; children set nothing.
- **Artifacts differ from anything built under `1.2.1`.** If the consumer compares
  build output byte-for-byte against a stored reference jar, re-baseline that reference
  against a `1.3.0` build. Differing bytes here are the expected effect, not a
  regression.
- **The value is not a build time and must never be presented as one.** It describes
  the `java-parent` release the artifact was built against. If the consumer surfaces a
  "built at" value anywhere — a `build-info.properties`, an actuator info contributor,
  a version endpoint — do **not** source it from this property. The parent's
  `full-build` profile writes a `Git-Commit-Time` manifest entry (UTC,
  commit-derived) for exactly that purpose, alongside `Git-Commit`, `Git-Branch`,
  `Git-Tag` and `Git-Dirty`.
- **Existing overrides keep working, and defeat the benefit.** Precedence is
  `-Dproject.build.outputTimestamp` on the command line > the child's `<properties>` >
  the inherited parent value. If the consumer's CI passes that flag (a common pattern
  before this release) or the child POM sets the property, the inherited literal is
  ignored and an external verifier cannot reproduce the published bytes. Remove such
  overrides unless there is a specific reason to keep them.
- **A `-SNAPSHOT` parent is not reproducible.** Snapshots between releases carry the
  previous release's date, which moves whenever the snapshot is republished. Only pin
  a released parent version if reproducibility matters.

Scope of the guarantee: *same source + same toolchain*. Reproducibility across
differing JDK patch versions, Maven versions or operating systems is **not** claimed
and has not been measured.

#### Breaking-light: Spotless now writes LF unconditionally

The parent's Spotless configuration now sets:

```xml
<!-- new in 1.3.0 (parent <build><plugins> spotless configuration) -->
<lineEndings>UNIX</lineEndings>
```

Spotless defaults to `GIT_ATTRIBUTES`, which resolves to whatever the developer's
`core.autocrlf` implies — CRLF on a default Windows install. With `UNIX` the
formatter's output no longer depends on the platform or the local Git configuration.

This is the only change in this release that can **fail a previously green build**:

- If the consumer's Java sources currently contain CRLF line endings, **`spotless:check`
  will now fail**, and `spotless:apply` will rewrite every affected file to LF —
  producing a large, purely-whitespace diff.
- The fix is to accept the normalization once, not to suppress it. Run
  `mvn spotless:apply`, commit the result as its own commit, and keep it out of any
  functional PR so review stays readable.
- To stop CRLF coming back on the next Windows checkout, add a `.gitattributes` to the
  consumer repository. The parent cannot deliver one — Git attributes are
  repository-local and Maven inheritance does not transport them. A ready-to-copy block
  is in the parent's README under **Line endings**; the minimum is:

  ```gitattributes
  *      text=auto eol=lf
  *.bat  text eol=crlf
  *.cmd  text eol=crlf
  ```

  After adding it, run `git add --renormalize .` once and commit whatever it stages.

Note the limit of the inherited setting: it applies only to files Spotless formats, and
only when `spotless:apply` or `spotless:check` is actually invoked — neither is bound to
a lifecycle phase. It guarantees the formatter never *introduces* CRLF. It does not make
the consumer's sources jar LF-clean; only the consumer's own `.gitattributes` does that.

#### Breaking-light: the OpenAPI stack is now version-managed

Under `1.2.1` the parent managed a single OpenAPI coordinate:

```xml
<!-- 1.2.1 (parent <dependencyManagement>) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.17</version>
</dependency>
```

Under `1.3.0` it manages the whole coupled stack:

```xml
<!-- 1.3.0 (parent <dependencyManagement>) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-bom</artifactId>
    <version>2.8.17</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>io.swagger.core.v3</groupId>
    <artifactId>swagger-bom</artifactId>
    <version>2.2.47</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>swagger-ui</artifactId>
    <version>5.32.2</version>
</dependency>
```

with two new overridable properties, `swagger.version` (`2.2.47`) and
`swagger-ui.version` (`5.32.2`), alongside the pre-existing `springdoc.version`
(still `2.8.17`).

**The bug this fixes.** springdoc declares its Swagger dependency without a version and
inherits it from its own aggregator POM, so under `1.2.1` Swagger floated and was
decided by Maven's nearest-wins mediation in every consuming project. A library that
uses only the Swagger annotations declares only `swagger-annotations-jakarta` — the
normal thing to do — and that declaration sits closer to the application than the
`2.2.47` springdoc contributes two levels down via `swagger-core-jakarta`. The stack
then **splits**: annotations at the older version, core at the newer one. Swagger's own
modules then call each other across incompatible releases:

```
io.swagger.v3.core.jackson.ModelResolver                      [swagger-core-jakarta 2.2.47]
  invokeinterface io/swagger/v3/oas/annotations/media/Schema.$dynamicRef:()Ljava/lang/String;
                                                added in swagger-annotations-jakarta 2.2.47
                                                absent in swagger-annotations-jakarta 2.2.29
```

The symptom is a `NoSuchMethodError` at schema resolution — typically on startup or on
the first `/v3/api-docs` request — not a missing feature. `dependencyManagement` is
consulted before mediation and applies at any depth, so managing the versions in the
parent removes the mediation entirely.

What this means for the consumer:

- **A transitively resolved older Swagger is upgraded to `2.2.47` with no consumer
  edit.** Verify with
  `mvn dependency:tree -Dincludes=io.swagger.core.v3,org.webjars:swagger-ui` that
  `swagger-annotations-jakarta`, `swagger-models-jakarta` and `swagger-core-jakarta`
  all show `2.2.47`.
- **If the consumer pinned springdoc backwards to work around this**, for example to
  `2.8.6` (whose matching Swagger is `2.2.29`, so the stack was uniform by accident),
  remove that pin and let the managed `2.8.17` apply.
- **A direct declaration with an explicit `<version>` still wins and is not fixed.**
  This is the one case the upgrade cannot repair: management overrides *transitive*
  resolution only. If the consumer declares a Swagger artifact directly with a
  `<version>`, it keeps that version, the split can persist, and nothing warns. Remove
  the `<version>` so the managed one applies.
- **A library in the consumer's own reactor compiled against `2.2.29` now runs against
  `2.2.47`.** The change is additive — between those releases the jakarta artifacts
  gained 8 classes and 89 public or protected members, and lost 1 — so this is low
  risk, but rebuild and test such a module once.

#### Additive: every springdoc starter is now usable without a version

Because the springdoc BOM is imported rather than a single coordinate managed by hand,
children may now declare any springdoc starter with the `<version>` omitted —
`springdoc-openapi-starter-webflux-ui`, `-webmvc-api`, `-webflux-api`,
`-webmvc-scalar`, `-webflux-scalar` and `-common`, in addition to the
`-webmvc-ui` that was already managed. Under `1.2.1` anything other than
`-webmvc-ui` required an explicit version. No action required; this only helps if the
consumer wants it.

#### Additive: all 14 Swagger coordinates are managed

`swagger-bom` covers the full Swagger artifact set, not only the three jakarta
artifacts springdoc pulls. A consumer declaring the non-jakarta line
(`swagger-annotations`, `swagger-core`, `swagger-models`) or the JAX-RS artifacts
(`swagger-jaxrs2-jakarta`, the servlet initializers) without a version now silently
gets `2.2.47` — a version selected for springdoc compatibility rather than for that
consumer. The build succeeds either way. If a consumer needs a different version for
the non-jakarta line, declare it directly with an explicit `<version>`.

#### Additive: retargeting the Swagger stack, and its floor

Setting `<swagger.version>` in the consumer's `<properties>` moves all 14 Swagger
coordinates at once, because property interpolation happens in the child's effective
POM and therefore reaches the parent's BOM import.

**The floor is `2.2.47`.** `io.swagger.core.v3:swagger-bom` was first published at that
version; every earlier version 404s on Maven Central. Setting `swagger.version` lower
does not downgrade the stack — it fails the build at model-building time with
`Non-resolvable import POM`. To use an older Swagger, declare the artifact directly
with an explicit `<version>` instead of moving the property.

#### Additive: recommended repository files

The parent repository now carries a `.gitattributes` and an `.editorconfig`, and the
README documents both for child projects. Neither is inherited — they are repository
files, not build configuration. Copying them is **optional but recommended**; the
`.gitattributes` is what keeps the Spotless change above from being re-broken by the
next Windows checkout.

One detail if the consumer already has an `.editorconfig`: the parent's `[*.java]`
block specifies **2 spaces and a 100 column limit**, matching the
`googleJavaFormat` that Spotless enforces. An `.editorconfig` specifying anything else
for Java makes the editor fight the formatter on every save.

#### Internal (no consumer action)

The parent's own release tooling changed: `release.sh` now rewrites
`project.build.outputTimestamp` to the release date and aborts if the rewrite does not
land, and both GitHub workflows dropped their `-Dproject.build.outputTimestamp`
command-line override so CI runs the same command a third party runs. The README gained
sections on reproducible builds, line endings and the OpenAPI stack, and
`docs/specs/` plus `docs/TODO.md` record the design work. None of this has any
consumer-facing effect.

### Steps

1. In the consumer's `pom.xml`, set the `<parent>` `<version>` of
   `com.open-elements:java-parent` to `1.3.0`. Leave all other coordinates untouched.
2. Run `mvn -U clean verify` and confirm the project compiles, tests pass, and
   dependencies resolve.
3. If the consumer uses springdoc, run
   `mvn dependency:tree -Dincludes=io.swagger.core.v3,org.webjars:swagger-ui` and
   confirm `swagger-annotations-jakarta`, `swagger-models-jakarta` and
   `swagger-core-jakarta` all resolve to `2.2.47`. If any of them does not, find the
   direct declaration with an explicit `<version>` that is overriding it and remove
   that `<version>`.
4. If the consumer pinned `springdoc.version` (or the springdoc starter itself)
   backwards to avoid a `NoSuchMethodError`, remove that pin now — the workaround is
   obsolete.
5. Run `mvn spotless:check`. If it fails, run `mvn spotless:apply` and commit the
   result **as its own commit**, separate from the version bump.
6. If step 5 produced changes, add a `.gitattributes` to the consumer repository (see
   the block above), then run `git add --renormalize .` and commit whatever it stages.
7. If the consumer's CI passes `-Dproject.build.outputTimestamp`, or the consumer's
   `pom.xml` sets that property, remove it so the inherited literal applies. Skip this
   only if there is a specific reason to keep the override.
8. If the consumer exposes a "built at" timestamp sourced from
   `project.build.outputTimestamp`, switch it to the `Git-Commit-Time` manifest entry,
   or remove it. The property is not a build time.
9. If (and only if) the consumer performs byte-for-byte artifact comparisons against a
   stored reference, re-baseline that reference against a `1.3.0` build.
10. Confirm the build is green before committing.

### Guard rails

- Do **not** bump Spring Boot, Testcontainers, or any plugin version in the consumer to
  "match 1.3.0" — none of them changed.
- Do **not** set `<swagger.version>` below `2.2.47`. It will not downgrade the stack; it
  will fail the build with a non-resolvable import POM. Use a direct declaration with an
  explicit `<version>` if an older Swagger is genuinely required.
- Do **not** add `<project.build.outputTimestamp>` to the consumer's `pom.xml` or pass
  it on the command line. Both override the inherited literal and make the consumer's
  releases unverifiable by an outside party — which is the entire point of the change.
- Do **not** revert the Spotless line-ending normalization by setting
  `<lineEndings>` back to `GIT_ATTRIBUTES` or `PLATFORM_NATIVE` in the consumer.
  Normalize once and add a `.gitattributes` instead.
- Keep the whitespace-only `spotless:apply` commit **separate** from the version bump
  and from any functional change, so the real diff stays reviewable.
- Make any fix in the **consumer's own sources**, not by editing `java-parent`.

### Don't do this

- Do not treat differing artifact bytes versus a `1.2.1` build as a regression — a fixed
  archive timestamp is the intended effect of this release.
- Do not present `project.build.outputTimestamp` as the time the build ran. It is the
  date of the `java-parent` release the artifact was built against. Use
  `Git-Commit-Time`.
- Do not suppress, skip, or configure away `spotless:check` to avoid the LF
  normalization — apply it and commit it.
- Do not pin Swagger back to `2.2.29` to "keep things as they were". That is the split
  this release exists to prevent, and a direct pin will silently reintroduce it.
- Do not assume this release makes builds reproducible across different JDKs, Maven
  versions or operating systems. The claim is *same source + same toolchain*; anything
  wider is unmeasured.
- Do not bundle this upgrade with unrelated dependency bumps, plugin changes, or feature
  work in the same PR.
