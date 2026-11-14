# Upgrade prompt: `com.open-elements:java-parent` 1.3.0 → 2.0.0

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its
`<parent>`, moving from `1.3.0` to `2.0.0`. This is a **major release with two
breaking changes that always apply**: the parent now manages **Spring Boot 4.1.2**
(was `3.5.14`) and requires **Java 25** (was `21`). It also moves the managed
**OpenAPI stack** to the Spring Boot 4 line (springdoc `3.2.0`), which is breaking for
every consumer that uses springdoc, and raises **jspecify** to `1.0.1`.

The parent itself contains no code and adds no new configuration: there are no new
properties, profiles or plugins, and every Maven plugin version is unchanged from
`1.3.0`. What forces work in the consumer is the framework and JDK it now targets.

The version lines are:

| `java-parent` | Spring Boot | Java |
|---------------|-------------|------|
| 2.x           | 4.1         | 25   |
| 1.x           | 3.5         | 21   |

**If the consumer must stay on Spring Boot 3.5 or Java 21, stop: do not perform this
upgrade.** Stay on `1.3.0`. There is no supported way to use `2.0.0` with Boot 3.5 or
with a Java 21 toolchain.

Your job is to move the consumer's toolchain to Java 25, bump the `<parent>` version,
remove consumer-side overrides that would defeat the parent's version decisions, align
the explicitly versioned `spring-boot-maven-plugin`, migrate the consumer's own code to
Spring Boot 4 where it does not compile or run, and verify the resolved dependency
tree. Apply exactly the changes below and nothing outside this scope.

### What changed in 2.0.0

#### Dependencies

Bump the `<parent>` version of `com.open-elements:java-parent` to `2.0.0`. The managed
versions change as follows — **the consumer does not set any of these itself**; they
arrive through the parent:

| Parent property          | 1.3.0    | 2.0.0     | Effect on the consumer                         |
|--------------------------|----------|-----------|------------------------------------------------|
| `spring-boot.version`    | `3.5.14` | **`4.1.2`** | Boot BOM import: Spring Framework `7.0.10`, Hibernate `7.4.13.Final`, Jackson `3.1.7` |
| `springdoc.version`      | `2.8.17` | **`3.2.0`** | springdoc BOM import                         |
| `swagger.version`        | `2.2.47` | **`2.2.55`** | `swagger-bom` import (all Swagger coordinates) |
| `swagger-ui.version`     | `5.32.2` | **`5.34.0`** | `org.webjars:swagger-ui`                      |
| `jspecify.version`       | `1.0.0`  | **`1.0.1`** | `org.jspecify:jspecify`, scope `compile`     |
| `testcontainers.version` | `2.0.5`  | `2.0.5`   | unchanged                                      |
| `maven.compiler.source` / `.target` | `21` | **`25`** | bytecode level of every compiled class |
| `enforcer.requiredJavaVersion` | `21` | **`25`** | minimum JDK the build runs on            |
| `enforcer.requiredMavenVersion` | `3.9.11` | `3.9.11` | unchanged                              |

**Do not bump anything else**: no Maven / Spotless / JReleaser / CycloneDX /
git-commit-id / pomchecker plugin version changed, and Testcontainers stays at `2.0.5`.
The only consumer-declared version this guide asks you to change is
`spring-boot-maven-plugin` (see below), because the parent never managed it.

#### Breaking: Java 25 is required to build and to run

The parent's `<properties>` now set:

```xml
<!-- 1.3.0 -->
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
<enforcer.requiredJavaVersion>21</enforcer.requiredJavaVersion>

<!-- 2.0.0 -->
<maven.compiler.source>25</maven.compiler.source>
<maven.compiler.target>25</maven.compiler.target>
<enforcer.requiredJavaVersion>25</enforcer.requiredJavaVersion>
```

Two consequences:

- **Building requires JDK 25.** On an older JDK the build fails in the `validate`
  phase, before compilation, with the `maven-enforcer-plugin` `RequireJavaVersion`
  rule; the message ends in, e.g.:

  ```
  ... version 21.0.11 which is not in the allowed range [25,).
  ```

  Every place that selects the build JDK must move to 25: local SDK configuration
  (`.sdkmanrc` → `java=25`, `.tool-versions`, `.java-version`, IDE project SDK), CI
  (`actions/setup-java` `java-version: "25"`, or the equivalent in other CI systems),
  and any Docker build stage that runs Maven (e.g. a `maven:…-eclipse-temurin-21` or
  `eclipse-temurin:21-jdk` builder image → the `25` variant).
- **Running requires Java 25.** Compiled classes now have class-file major version
  `69`. A Java 21 runtime rejects them with `UnsupportedClassVersionError`. Every
  runtime image or host JRE that executes the consumer's artifact — Dockerfile runtime
  stage (`eclipse-temurin:21-jre` → `eclipse-temurin:25-jre`), `jib`/buildpack
  `BP_JVM_VERSION` / base-image settings, deployment descriptors — must move to 25 as
  well. A library consumer's own downstream users inherit the same requirement.

The parent makes the Java baseline decision; children do not. If the consumer's
`pom.xml` sets `maven.compiler.source`, `maven.compiler.target`, `maven.compiler.release`
or configures `<source>` / `<target>` / `<release>` on `maven-compiler-plugin` to `21`
(or anything below 25), remove that setting so the inherited `25` applies. Leaving
`21` there produces Java 21 bytecode while still demanding a JDK 25 build — the worst
of both.

A child may still **raise** `enforcer.requiredJavaVersion` above 25; that mechanism is
unchanged from 1.x.

#### Breaking: Spring Boot 4.1.2 is managed

The parent imports `org.springframework.boot:spring-boot-dependencies` at
`${spring-boot.version}`, now `4.1.2`. Every Spring Boot starter and every library
version-managed by Boot that the consumer declares without a `<version>` now resolves
to the Boot 4.1.2 line — Spring Framework `7.0.10`, Hibernate ORM `7.4.13.Final`,
Jackson `3.1.7` (`tools.jackson.*`), and so on.

The parent does **not** migrate consumer code. Boot 4's own breaking changes are the
consumer's concern and are handled in this upgrade, following the official
[Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
(and the 4.1 release notes linked from the Spring Boot wiki). The areas that most
commonly require consumer edits:

- **Starters were renamed/split.** For example the servlet web starter is
  `spring-boot-starter-webmvc` in Boot 4 (the form `java-parent` 2.0.0 was verified
  with); test support is split into per-technology test starters. Follow the
  migration guide's starter table rather than guessing names.
- **Jackson 3.** Boot 4 uses Jackson 3, whose packages and Maven group move from
  `com.fasterxml.jackson.core` / `com.fasterxml.jackson.databind` to
  `tools.jackson.core` / `tools.jackson.databind`. Code importing Jackson databind
  types (`ObjectMapper`, serializers, modules) and Boot Jackson customizers must be
  migrated per the migration guide.
- **Spring Framework 7 and Hibernate 7** API removals and deprecations.

How to recognise that a Boot 4 change is at fault: compile errors on Spring, Jackson
or Hibernate types, `ClassNotFoundException` / missing auto-configuration at startup,
or a starter that no longer resolves. Fix each one in the consumer's own sources per the
migration guide; do not work around them by pinning Boot 3 artifacts.

**Do not set `spring-boot.version` in the consumer.** A child property of that name
shadows the parent's and silently pins the Boot BOM import to the child's value. If
the consumer currently sets it (for example to `3.5.14`), remove it.

#### Breaking: `spring-boot-maven-plugin` must be bumped by the consumer

A `scope=import` BOM contributes `dependencyManagement` only, never
`pluginManagement`. `spring-boot-maven-plugin` was therefore **never** managed by
`java-parent` — not in 1.x and not in 2.0.0 — and consumers declare it with an explicit
`<version>`. That version must now match Boot 4.1.2:

```xml
<!-- 1.3.0 era -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>3.5.14</version>
</plugin>

<!-- 2.0.0 -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>${spring-boot.version}</version>  <!-- inherited from the parent: 4.1.2 -->
</plugin>
```

Referencing the inherited `${spring-boot.version}` property is preferred over the
literal `4.1.2`, because it keeps the plugin in lockstep with future parent releases.
Reading the property is fine; *setting* it in the child is not (see above). If the
consumer does not declare `spring-boot-maven-plugin` (typical for libraries), skip this.

#### Breaking (springdoc users): the OpenAPI stack moves to springdoc 3.2.0

springdoc 2.x supports Spring Boot 3 only; springdoc `3.2.0` is built against
`spring-boot-starter-parent` `4.1.2`. The parent keeps the coupled-stack management
introduced in 1.3.0 and moves all three versions together:

```xml
<!-- 1.3.0 -->
<springdoc.version>2.8.17</springdoc.version>
<swagger.version>2.2.47</swagger.version>
<swagger-ui.version>5.32.2</swagger-ui.version>

<!-- 2.0.0 -->
<springdoc.version>3.2.0</springdoc.version>
<swagger.version>2.2.55</swagger.version>
<swagger-ui.version>5.34.0</swagger-ui.version>
```

The springdoc artifact IDs are unchanged (`springdoc-openapi-starter-webmvc-ui`,
`-webflux-ui`, `-webmvc-api`, … all still managed without a `<version>`). The Swagger
values are the ones springdoc 3.2.0's own aggregator POM declares
(`<swagger-api.version>` and `<swagger-ui.version>` in `springdoc-openapi-3.2.0.pom`).

What this means for the consumer:

- **No edit is needed** if springdoc and Swagger are declared without `<version>`.
- **Remove any consumer-side override** of `springdoc.version`, `swagger.version` or
  `swagger-ui.version`, and any explicit `<version>` on a springdoc, `io.swagger.core.v3`
  or `org.webjars:swagger-ui` dependency. A leftover `2.8.x` springdoc pin is
  incompatible with Boot 4; a leftover Swagger pin reintroduces the annotations/core
  version split that 1.3.0 fixed (symptom: `NoSuchMethodError` during schema
  resolution).
- The `swagger.version` floor from 1.3.0 still holds: `swagger-bom` does not exist below
  `2.2.47`, so a lower value fails with `Non-resolvable import POM`. In practice, keep
  the managed `2.2.55` — it is the version springdoc 3.2.0 was built against.
- Verify with
  `mvn dependency:tree -Dincludes=org.springdoc,io.swagger.core.v3,org.webjars:swagger-ui`:
  springdoc artifacts at `3.2.0`; `swagger-annotations-jakarta`,
  `swagger-models-jakarta` and `swagger-core-jakarta` all at `2.2.55`; `swagger-ui` at
  `5.34.0`.

#### Breaking-light (springdoc users): Jackson 2 and Jackson 3 side by side

`swagger-core-jakarta` `2.2.55`, which springdoc 3.2.0 uses, still depends on
**Jackson 2** (`com.fasterxml.jackson.core:jackson-databind`, observed at `2.21.7`),
while Boot 4 uses **Jackson 3** (`tools.jackson.core:jackson-databind` `3.1.7`).
Applications using springdoc therefore have both on the runtime classpath. This is
expected and is not a dependency conflict to resolve — the two use different packages
and Maven groups.

What this means for the consumer:

- **Do not exclude** `com.fasterxml.jackson.*` from springdoc / Swagger. Swagger needs
  it at runtime.
- **Do not let the presence of Jackson 2 hide incomplete migration.** Because
  `com.fasterxml.jackson.databind.ObjectMapper` is still on the classpath, consumer code
  that still imports Jackson 2 databind types **compiles**, but it is not the
  `ObjectMapper` Spring Boot 4 configures and uses for HTTP message conversion. Migrate
  the consumer's own Jackson usage to `tools.jackson.*` per the Boot 4 migration guide;
  do not rely on "it compiles".
- Projects that use only `io.swagger.core.v3:swagger-annotations-jakarta` (typical for
  libraries that annotate DTOs) are **not affected** — that artifact has no
  dependencies.

#### Breaking-light: jspecify 1.0.0 → 1.0.1

The parent's explicit `org.jspecify:jspecify` entry (scope `compile`) moves to `1.0.1`,
the version Spring Boot 4.1.2 itself manages. Under the old value the parent's explicit
entry would have won over Boot's BOM and downgraded jspecify below what Spring
Framework 7 was built with. `1.0.1` is a patch release; annotation usage is unchanged.
No action unless the consumer pins jspecify itself — then remove the pin.

#### Breaking-light: artifacts carry the 2.0.0 release timestamp

The reproducible-build mechanism from 1.3.0 is unchanged; only the literal moves to
the 2.0.0 release date:

```xml
<!-- 2.0.0 (parent <properties>) -->
<project.build.outputTimestamp>2026-11-14T00:00:00Z</project.build.outputTimestamp>
```

Consumer artifacts built against 2.0.0 differ byte-for-byte from those built against
1.3.0. That is expected. Do not add or override the property in the consumer.

#### Unchanged (no consumer action)

- Testcontainers BOM `2.0.5`.
- Every pinned plugin version, the Spotless configuration (`googleJavaFormat`,
  `lineEndings` `UNIX`), the `full-build` and release profiles, and the Maven minimum
  `3.9.11`.
- The `-parameters` compiler flag and UTF-8 encoding.

#### Internal (no consumer action)

The parent's own CI workflows (`build.yml`, `snapshot.yml`, `release.yml`) and its
`.sdkmanrc` moved to Java 25 so that the parent can build itself under its new
enforcer rule. The README gained a "Version lines" section and the Jackson 2 + 3 note,
and `docs/specs/004-spring-boot-4/` records the design.

### Steps

1. **Check eligibility.** If the consumer cannot move to both Spring Boot 4 and Java 25
   now, stop and stay on `java-parent` `1.3.0`.
2. **Move the build toolchain to JDK 25**: local SDK files (`.sdkmanrc`,
   `.tool-versions`, `.java-version`), CI `setup-java` (or equivalent) `java-version`,
   and any Docker build stage that runs Maven. Confirm `java -version` reports 25 in the
   environment that will run the build.
3. **Move the runtime to Java 25**: Dockerfile runtime base image, Jib / buildpack JVM
   settings, and any deployment configuration that pins a JRE 21. Skip if the consumer
   is a library that ships no runtime.
4. In the consumer's `pom.xml`, set the `<parent>` `<version>` of
   `com.open-elements:java-parent` to `2.0.0`.
5. **Remove consumer overrides of parent decisions** from the consumer's
   `<properties>` / plugin configuration, if present:
   `spring-boot.version`, `springdoc.version`, `swagger.version`, `swagger-ui.version`,
   `jspecify.version`, `maven.compiler.source`, `maven.compiler.target`,
   `maven.compiler.release`, and `<source>`/`<target>`/`<release>` below 25 on
   `maven-compiler-plugin`. Also remove explicit `<version>` elements on springdoc,
   `io.swagger.core.v3`, `org.webjars:swagger-ui` and `org.jspecify:jspecify`
   dependencies.
6. If the consumer declares `spring-boot-maven-plugin`, set its `<version>` to
   `${spring-boot.version}` (resolves to `4.1.2`).
7. Run `mvn -U clean verify`. Expect failures from Boot 4. Migrate the consumer's own
   sources and dependency declarations per the
   [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide):
   renamed/split starters (including test starters), Jackson 3 (`tools.jackson.*`),
   Spring Framework 7 and Hibernate 7 API changes. Repeat until compile and tests pass.
8. Grep the consumer's sources for `com.fasterxml.jackson.databind` and
   `com.fasterxml.jackson.core` imports and migrate the remaining ones to Jackson 3,
   even if they compile (see the Jackson 2 + 3 section). Jackson **annotations**
   (`com.fasterxml.jackson.annotation.*`) follow the migration guide's guidance rather
   than this grep.
9. **Verify the resolved versions**:
   - `mvn dependency:tree -Dincludes=org.springframework.boot` → `4.1.2`;
     `-Dincludes=org.springframework:spring-core` → `7.0.10`.
   - If springdoc is used:
     `mvn dependency:tree -Dincludes=org.springdoc,io.swagger.core.v3,org.webjars:swagger-ui`
     → springdoc `3.2.0`, the three Swagger jakarta artifacts `2.2.55`, `swagger-ui`
     `5.34.0`.
   - If jspecify is used: `-Dincludes=org.jspecify` → `1.0.1`.
   - If Testcontainers is used: still `2.0.5`.
10. If the consumer is an application, start it once (or run its integration tests) on
    Java 25 and, if it uses springdoc, request `/v3/api-docs` to confirm schema
    resolution works.
11. Confirm the build is green on JDK 25 locally and in CI before committing.

### Guard rails

- Do **not** perform this upgrade on a project that must stay on Spring Boot 3.5 or
  Java 21. Use `java-parent` `1.3.0`.
- Do **not** set `spring-boot.version` in the consumer — not even to `4.1.2`. It
  shadows the parent's property and pins the Boot BOM. Reading it via
  `${spring-boot.version}` is fine.
- Do **not** lower `enforcer.requiredJavaVersion` or set compiler source/target/release
  below 25 to keep an older JDK working. The parent's baseline is the decision.
- Do **not** move only the build JDK and forget the runtime. Java 25 bytecode fails on a
  Java 21 JRE with `UnsupportedClassVersionError`.
- Do **not** pin springdoc back to `2.8.x` or any Swagger artifact to an explicit
  version. springdoc 2.x does not support Boot 4, and a direct Swagger pin reopens the
  annotations/core split.
- Do **not** bump Testcontainers or any Maven plugin version "to match 2.0.0" — none
  changed. The only plugin version to change is the consumer's own
  `spring-boot-maven-plugin`.
- Make every fix in the **consumer's own sources and POM**, not by editing
  `java-parent`.

### Don't do this

- Do not exclude Jackson 2 (`com.fasterxml.jackson.*`) from springdoc/Swagger to "clean
  up" the classpath — Swagger needs it at runtime.
- Do not treat "it compiles" as proof that Jackson usage is migrated; Jackson 2 is still
  on the classpath via Swagger.
- Do not shim Boot 3 behaviour back in by declaring Boot 3 artifacts with explicit
  versions next to the Boot 4 BOM.
- Do not add `project.build.outputTimestamp` to the consumer or treat differing artifact
  bytes versus a 1.3.0 build as a regression.
- Do not skip intermediate guides: a consumer still on a parent older than `1.3.0` must
  first apply `upgrade-to-1.3.0.md` (and any earlier guides) in sequence.
- Do not bundle this upgrade with unrelated dependency bumps, refactors or feature work
  in the same PR. If the Boot 4 code migration is large, keep it in the same PR as the
  parent bump (they cannot be separated without a broken build), but nothing else.
