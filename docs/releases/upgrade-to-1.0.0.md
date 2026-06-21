# Upgrade prompt: `com.open-elements:java-parent` 0.5.1 → 1.0.0

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its `<parent>`, moving from `0.5.1` to `1.0.0`. This is the **first stable major release**, and it is **breaking**. Two things change for consumers:

1. **The parent no longer contributes any dependencies to its children.** In `0.5.1` the parent declared a top-level `<dependencies>` block, so every child inherited four artifacts directly on its classpath. In `1.0.0` that block is gone — children that used any of those four types must now declare them themselves.
2. **A `maven-enforcer-plugin` rule now fails the build** if it runs on Maven `< 3.9.11` or Java `< 21`.

There are **no managed-dependency version changes** (Spring Boot stays `3.5.14`, Testcontainers stays `2.0.5`) and **no API/code changes** — this is a build-model change only. Apply exactly the changes below and nothing outside this scope.

### What changed in 1.0.0

#### Dependencies

Bump only the `<parent>` version of `com.open-elements:java-parent` to `1.0.0`. The **BOM-managed versions are unchanged**: `spring-boot-dependencies` stays at `3.5.14` and `testcontainers-bom` stays at `2.0.5`, both still imported via the parent's `<dependencyManagement>`. Do **not** bump those, and do **not** bump any Maven / JReleaser / CycloneDX / Spotless plugin version in the consumer — the plugin bumps in `1.0.0` are internal to the parent and have no consumer-facing effect.

#### Breaking: inherited `<dependencies>` removed from the parent

In `0.5.1` the parent declared these as a **top-level `<dependencies>` block** (not `<dependencyManagement>`), so every child inherited all four as real, resolved dependencies on its compile classpath:

```xml
<!-- present in 0.5.1, REMOVED in 1.0.0 -->
<dependencies>
    <dependency>
        <groupId>io.swagger.core.v3</groupId>
        <artifactId>swagger-annotations-jakarta</artifactId>
        <version>2.2.29</version>
    </dependency>
    <dependency>
        <groupId>org.jspecify</groupId>
        <artifactId>jspecify</artifactId>
        <version>1.0.0</version>
        <scope>compile</scope>
    </dependency>
    <dependency>
        <groupId>com.slack.api</groupId>
        <artifactId>slack-api-client</artifactId>
        <version>1.45.3</version>
    </dependency>
    <dependency>
        <groupId>org.wiremock</groupId>
        <artifactId>wiremock-standalone</artifactId>
        <version>3.10.0</version>
    </dependency>
</dependencies>
```

In `1.0.0` the parent declares **no** top-level `<dependencies>` at all. The accompanying `<properties>` versions were also removed (`swagger-annotations-jakarta.version`, `jspecify.version`, `slack-api-client.version`, `wiremock.version`, and the unused `junit-jupiter.version`).

A consumer that **used any of these types in its own source or tests** will now fail to compile (e.g. `package org.jspecify.annotations does not exist`, `package io.swagger.v3.oas.annotations does not exist`, `cannot find symbol: class WireMockServer` / `com.slack.api.*`). A consumer that never referenced them is unaffected and needs no action.

For each of the four that the consumer actually uses, add it back as an **explicit dependency in the consumer's own `pom.xml`**, pinning the same version that `0.5.1` provided (so behavior is byte-for-byte unchanged) and choosing the correct scope:

```xml
<!-- Add ONLY the ones the consumer actually references -->

<!-- JSpecify nullness annotations (@Nullable, @NonNull) — used in main source -->
<dependency>
    <groupId>org.jspecify</groupId>
    <artifactId>jspecify</artifactId>
    <version>1.0.0</version>
</dependency>

<!-- Swagger / OpenAPI annotations (io.swagger.v3.oas.annotations.*) -->
<dependency>
    <groupId>io.swagger.core.v3</groupId>
    <artifactId>swagger-annotations-jakarta</artifactId>
    <version>2.2.29</version>
</dependency>

<!-- Slack API client (com.slack.api.*) -->
<dependency>
    <groupId>com.slack.api</groupId>
    <artifactId>slack-api-client</artifactId>
    <version>1.45.3</version>
</dependency>

<!-- WireMock standalone — typically test-only; use <scope>test</scope> unless
     the consumer genuinely references it in main source -->
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.10.0</version>
    <scope>test</scope>
</dependency>
```

#### Breaking: build environment now enforced (Maven ≥ 3.9.11, Java ≥ 21)

`1.0.0` binds `maven-enforcer-plugin` to the build with a rule that fails fast on an unsupported toolchain:

```xml
<!-- new in 1.0.0 (parent) -->
<requireMavenVersion><version>3.9.11</version></requireMavenVersion>
<requireJavaVersion><version>21</version></requireJavaVersion>
```

If the consumer builds on Maven `< 3.9.11` or a JDK `< 21`, the build now stops with an enforcer error such as `Detected Maven Version: ... is not in the allowed range 3.9.11` or `Detected JDK version ... is not in the allowed range 21`. This is the intended behavior — these are the minimums the parent already targeted (`maven.compiler.source/target` were already `21`); `1.0.0` just makes the requirement explicit and hard.

These minimums are exposed as overridable properties, so a child that needs a **higher** floor can raise them (it cannot lower them below what the parent enforces in practice without redefining the rule):

```xml
<properties>
    <enforcer.requiredMavenVersion>3.9.11</enforcer.requiredMavenVersion>
    <enforcer.requiredJavaVersion>21</enforcer.requiredJavaVersion>
</properties>
```

#### Additive / internal (no consumer action)

These changed in the parent but do not require any consumer edit: `cyclonedx-maven-plugin` `2.9.1`→`2.9.2`, `jreleaser-maven-plugin` `1.23.0`→`1.24.0`, `spotless-maven-plugin` `3.4.0`→`3.7.0`, `versions-maven-plugin` gained `<overwriteOutput>true</overwriteOutput>`, plus CI, helper-script, and `target/` artifact cleanup internal to the `java-parent` repository.

### Steps

1. In the consumer's `pom.xml`, set the `<parent>` `<version>` of `com.open-elements:java-parent` to `1.0.0`. Leave all other coordinates untouched.
2. Confirm the build toolchain meets the new floor: Maven `≥ 3.9.11` and JDK `≥ 21` (check `mvn -version`). Upgrade the local/CI toolchain if either is below the minimum — do **not** weaken the enforcer rule.
3. Determine which of the four removed dependencies the consumer actually references. Search the consumer's source and tests for: `org.jspecify`, `io.swagger.v3.oas` (or `io.swagger.core.v3`), `com.slack.api`, and `org.wiremock` / `WireMock`.
4. For each one found, add the matching explicit `<dependency>` (from the block above) to the consumer's `pom.xml`, using the listed version and the correct scope (WireMock is normally `test`).
5. Run `mvn -U clean verify` (or the project's equivalent) and confirm the project compiles, all dependencies resolve, and the test suite is green before committing.

### Guard rails

- Do **not** bump Spring Boot, Testcontainers, or any plugin version in the consumer to "match 1.0.0" — the BOM-managed versions are unchanged and the plugin bumps are internal to the parent.
- Do **not** add any of the four dependencies that the consumer does not actually reference. Re-adding them speculatively re-introduces classpath bloat that this release intentionally removed.
- Pin the re-added dependencies to the **exact versions listed** (the ones `0.5.1` provided) so behavior is unchanged; do not silently upgrade them as part of this bump.
- Do **not** lower, disable, or `<skip>` the enforcer rule to get an older Maven/JDK to build. Fix the toolchain instead.
- Add WireMock with `<scope>test</scope>` unless the consumer genuinely uses it in main source; the parent previously leaked it as compile scope, which you should not reproduce.

### Don't do this

- Do not "shim" the removal by adding the four dependencies back into `java-parent` itself, or by re-declaring a top-level `<dependencies>` block in an intermediate parent so children keep inheriting them — the removal is intentional.
- Do not move the re-added dependencies into the consumer's `<dependencyManagement>` only; they must be real `<dependencies>` for the code that uses them to compile.
- Do not edit, override, or delete the enforcer rule, `enforcer.requiredMavenVersion`, or `enforcer.requiredJavaVersion` to dodge the toolchain requirement.
- Do not bundle this upgrade with unrelated dependency bumps, plugin changes, or feature work in the same PR.
