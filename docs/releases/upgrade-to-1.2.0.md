# Upgrade prompt: `com.open-elements:java-parent` 1.1.0 → 1.2.0

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its `<parent>`, moving from `1.1.0` to `1.2.0`. This is a **minor release that is breaking-light for publishing builds only**: a normal `mvn clean verify` (no profile) behaves exactly as before, but the inherited **`full-build` profile now validates the consumer's POM against Maven Central's publishing rules** and can **fail the build** if required metadata is missing. There are **no API/code changes** and **no BOM-managed version changes** (Spring Boot stays `3.5.14`, Testcontainers stays `2.0.5`).

The two additive items — new managed versions for `springdoc-openapi-starter-webmvc-ui` and `jspecify` — require **no consumer action**; they only take effect if the consumer *chooses* to declare those dependencies without a version. Apply exactly the changes below and nothing outside this scope.

### What changed in 1.2.0

#### Dependencies

Bump only the `<parent>` version of `com.open-elements:java-parent` to `1.2.0`. The **BOM-managed versions are unchanged**: `spring-boot-dependencies` stays at `3.5.14` and `testcontainers-bom` stays at `2.0.5`, both still imported via the parent's `<dependencyManagement>`. Do **not** bump those, and do **not** bump any Maven / JReleaser / CycloneDX / Spotless / git-commit-id / pomchecker plugin version in the consumer — the plugin set in `1.2.0` is internal to the parent and has no consumer-facing version effect.

#### Additive: two new managed dependency versions

The parent's `<dependencyManagement>` now pins two additional coordinates, so children may declare them **without a `<version>`**:

```xml
<!-- new in 1.2.0 (parent <dependencyManagement>) -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.17</version>
</dependency>
<dependency>
    <groupId>org.jspecify</groupId>
    <artifactId>jspecify</artifactId>
    <version>1.0.0</version>
    <scope>compile</scope>
</dependency>
```

These are **management entries only** — the parent does **not** add either dependency to any child. A consumer gets them only if it explicitly declares them. Adoption is **strictly optional**; skipping it leaves the build behaving exactly as before. If the consumer *wants* to use them, declare them with the version omitted so the managed version applies:

```xml
<!-- optional: add to the consumer's <dependencies> only if actually needed -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
</dependency>
<dependency>
    <groupId>org.jspecify</groupId>
    <artifactId>jspecify</artifactId>
</dependency>
```

Note the `jspecify` entry is managed at `compile` scope: if the consumer declares it without a `<scope>`, its nullness annotations (`@Nullable`, `@NonNull`, `@NullMarked`) end up on the compile/runtime classpath. A consumer that already declares either coordinate with a pinned version keeps its own version — Maven's nearest-definition rules mean the child's explicit `<version>` wins; align it to the managed value only if you deliberately want to.

#### Breaking-light: `-Pfull-build` now validates the POM for Maven Central (`pomchecker`)

`1.2.0` adds `pomchecker-maven-plugin` (`1.14.0`), version-managed in `<pluginManagement>` and wired into the **`full-build` profile only**. Because profiles are inherited from the parent, a child that builds with `-Pfull-build` now runs an extra check bound to the **`verify`** phase:

```xml
<!-- new in 1.2.0 (parent full-build profile) -->
<plugin>
    <groupId>org.kordamp.maven</groupId>
    <artifactId>pomchecker-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>check-maven-central</id>
            <phase>verify</phase>
            <goals>
                <goal>check-maven-central</goal>
            </goals>
            <configuration>
                <release>false</release>
            </configuration>
        </execution>
    </executions>
</plugin>
```

`check-maven-central` validates that the POM carries every field Maven Central requires: `<name>`, `<description>`, `<url>`, at least one `<license>`, at least one `<developer>`, and a complete `<scm>` (connection, developerConnection, url). `release=false` allows the project version and dependencies to be `SNAPSHOT`, so this runs cleanly on snapshot builds too — but the **metadata requirements still apply**. If a consumer's own `pom.xml` is missing any of those fields, its **`-Pfull-build` build now fails at `verify`** with a pomchecker error, where in `1.1.0` it did not run at all.

This is **build-time only** — a plain `mvn clean verify` (without `-Pfull-build`) is completely unaffected; pomchecker exists solely inside that profile. The fix, when it triggers, is to **complete the consumer's POM metadata**, not to disable the check:

```xml
<!-- add whichever of these the consumer's own pom.xml is missing -->
<name>Your Artifact Name</name>
<description>A one-line description of the artifact.</description>
<url>https://github.com/your-org/your-repo</url>
<licenses>
    <license>
        <name>The Apache License, Version 2.0</name>
        <url>https://www.apache.org/licenses/LICENSE-2.0.txt</url>
    </license>
</licenses>
<developers>
    <developer>
        <name>Your Name</name>
        <email>you@example.com</email>
    </developer>
</developers>
<scm>
    <connection>scm:git:https://github.com/your-org/your-repo.git</connection>
    <developerConnection>scm:git:https://github.com/your-org/your-repo.git</developerConnection>
    <url>https://github.com/your-org/your-repo</url>
</scm>
```

A consumer whose POM is already Maven-Central-complete (which any project already publishing through this parent should be) passes the check with no changes.

#### Breaking-light: `full-build` default goal changed `package` → `verify`

The `full-build` profile's `<defaultGoal>` changed from `package` to `verify`. This only matters for an invocation of `mvn -Pfull-build` with **no explicit goal/phase**: in `1.1.0` that stopped at `package`; in `1.2.0` it now runs through `verify`, which is exactly where the new pomchecker execution binds. A build that already passes an explicit phase (e.g. `mvn -Pfull-build verify`, `... deploy`) is unaffected by the default-goal change itself — but note that any explicit goal at or past `verify` will also trigger pomchecker (see above). The javadoc/sources jar attachment and the CycloneDX SBOM already bound within `package`/`verify` are unchanged.

#### Internal (no consumer action)

These changed in the `java-parent` repository but have no consumer-facing effect: the addition of `pomchecker-maven-plugin` to `<pluginManagement>` (version management only), and explanatory `<!-- ... -->` description comments added to the `full-build` and `deploy-release` profiles.

### Steps

1. In the consumer's `pom.xml`, set the `<parent>` `<version>` of `com.open-elements:java-parent` to `1.2.0`. Leave all other coordinates untouched.
2. Run the consumer's **publishing/CI build** the way it is normally invoked — i.e. with `-Pfull-build` (e.g. `mvn -U -Pfull-build clean verify`). If pomchecker's `check-maven-central` fails, add whichever of `<name>`, `<description>`, `<url>`, `<licenses>`, `<developers>`, `<scm>` fields are missing from the **consumer's own `pom.xml`** (block above). Do not disable or skip the check.
3. (Optional) If the consumer needs SpringDoc OpenAPI or JSpecify, declare the coordinate(s) **without a `<version>`** so the parent's managed version applies. Otherwise do nothing.
4. Run a plain `mvn -U clean verify` (no profile) and confirm the project still compiles, tests pass, and dependencies resolve — this path is unchanged and must stay green.
5. Confirm both invocations succeed before committing.

### Guard rails

- Do **not** bump Spring Boot, Testcontainers, or any plugin version in the consumer to "match 1.2.0" — the BOM-managed versions are unchanged and the plugin set (including `pomchecker`) is internal to the parent with no consumer-facing version effect.
- Do **not** add `springdoc-openapi-starter-webmvc-ui` or `jspecify` to the consumer's `<dependencies>` unless the project actually uses them; these are optional management entries, not new transitive dependencies.
- When declaring the new managed dependencies, **omit the `<version>`** so the parent's version applies — do not re-pin them in the consumer.
- If `-Pfull-build` fails on pomchecker, fix it by **completing the consumer's POM metadata**, not by setting `<skip>`, excluding the plugin, or removing the profile activation.
- Make POM-metadata additions in the **consumer's own `pom.xml`**, not by editing `java-parent`.

### Don't do this

- Do not disable, skip, or `<version>`-override the `pomchecker-maven-plugin` in the consumer to make a failing `-Pfull-build` build pass — the check exists so a missing field fails locally instead of during the Maven Central deploy.
- Do not copy the new `springdoc`/`jspecify` version numbers into the consumer's `<properties>` or dependency declarations — rely on the inherited management.
- Do not add either new dependency "just in case"; a management entry the consumer never declares has zero effect and should stay that way.
- Do not bundle this upgrade with unrelated dependency bumps, plugin changes, or feature work in the same PR.
