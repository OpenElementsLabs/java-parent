# Upgrade prompt: `com.open-elements:java-parent` 1.0.0 → 1.1.0

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its `<parent>`, moving from `1.0.0` to `1.1.0`. This is a **minor release that is breaking-light**: nothing fails to compile from the bump alone, but the parent **stops contributing two build settings that children inherited in `1.0.0`** — the `-parameters` compiler flag and the Surefire `--add-opens` test JVM flags. Code that relied on either can change behavior or fail **at runtime / during tests**, silently, with no compile error.

There are **no managed-dependency version changes** (Spring Boot stays `3.5.14`, Testcontainers stays `2.0.5`) and **no API/code changes** — this is a build-model change only. The two additive items (an active Spotless config and `git-commit-id` manifest stamping) require **no consumer action**. Apply exactly the changes below and nothing outside this scope.

### What changed in 1.1.0

#### Dependencies

Bump only the `<parent>` version of `com.open-elements:java-parent` to `1.1.0`. The **BOM-managed versions are unchanged**: `spring-boot-dependencies` stays at `3.5.14` and `testcontainers-bom` stays at `2.0.5`, both still imported via the parent's `<dependencyManagement>`. Do **not** bump those, and do **not** bump any Maven / JReleaser / CycloneDX / Spotless / git-commit-id plugin version in the consumer — the plugin set in `1.1.0` is internal to the parent and has no consumer-facing version effect.

#### Breaking-light: parent no longer adds `-parameters` to the compiler

In `1.0.0` the parent's top-level `<build><plugins>` actively configured the compiler so that **method/constructor parameter names were retained in the bytecode**:

```xml
<!-- present in 1.0.0 (parent <build><plugins>), REMOVED in 1.1.0 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <parameters>true</parameters>
    </configuration>
</plugin>
```

In `1.1.0` this block is gone. The `maven-compiler-plugin` is still **version-managed** in the parent's `<pluginManagement>` (`3.15.0`), but it no longer sets `<parameters>true</parameters>`, so children compile **without** `-parameters` unless they set it themselves. This compiles cleanly — the breakage is at **runtime**, in frameworks that look up parameters by name:

- Spring MVC handler args without an explicit name: `@PathVariable Long id`, `@RequestParam String q` → `IllegalArgumentException: Name for argument of type [...] not specified, and parameter name information not available via reflection`.
- Spring Data derived/`@Query` method parameters bound by name.
- Jackson `ParameterNamesModule` constructor binding.
- Any code calling `Parameter#getName()` and expecting the real name instead of `arg0`, `arg1`, …

A consumer that **relies on parameter-name retention** must restore the flag in its **own** `pom.xml`:

```xml
<!-- Add to the consumer's <build><plugins> only if it relies on -parameters -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <parameters>true</parameters>
    </configuration>
</plugin>
```

A consumer that does not depend on parameter names (or already sets `-parameters` itself, e.g. via Spring Boot's own parent in a non-inheriting setup) is unaffected and needs no action.

#### Breaking-light: parent no longer adds `--add-opens` to the test JVM

In `1.0.0` the parent's top-level `<build><plugins>` set a Surefire `argLine` that opened three JDK packages to reflective access during tests:

```xml
<!-- present in 1.0.0 (parent <build><plugins>), REMOVED in 1.1.0 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.lang.reflect=ALL-UNNAMED
            --add-opens java.base/java.util=ALL-UNNAMED
        </argLine>
    </configuration>
</plugin>
```

In `1.1.0` this block is gone. `maven-surefire-plugin` is still version-managed (`3.5.5`) but injects no `argLine`. Tests compile and start, but reflection-heavy test/mocking libraries that deep-reflect into `java.base` can now fail at runtime with:

```
java.lang.reflect.InaccessibleObjectException: Unable to make ... accessible:
module java.base does not "opens java.lang" to unnamed module ...
```

A consumer whose test suite needs that access must restore it in its **own** `pom.xml`:

```xml
<!-- Add to the consumer's <build><plugins> only if its tests need deep reflection -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.lang.reflect=ALL-UNNAMED
            --add-opens java.base/java.util=ALL-UNNAMED
        </argLine>
    </configuration>
</plugin>
```

If the consumer **already uses JaCoCo** (or anything else that injects a Surefire `argLine` via the `${argLine}` property), do **not** hard-overwrite it. Reference the injected value so coverage instrumentation is preserved:

```xml
<argLine>@{argLine} --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED</argLine>
```

A consumer whose tests do not perform such reflection is unaffected and needs no action.

#### Additive: Spotless is now actively configured (Google Java Format)

In `1.0.0` `spotless-maven-plugin` was only **version-managed** in `<pluginManagement>`. In `1.1.0` the parent also activates it in its top-level `<build><plugins>` with Google Java Format:

```xml
<!-- new in 1.1.0 (parent <build><plugins>) -->
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <configuration>
        <java>
            <googleJavaFormat/>
        </java>
    </configuration>
</plugin>
```

There is **no `<execution>` binding**, so Spotless does **not** run during `clean verify` and **cannot fail the build automatically**. Children simply gain working `mvn spotless:apply` / `mvn spotless:check` goals that reformat to Google Java Format. Adoption is **optional**; skipping it leaves the build behaving exactly as before.

Caveat: if the consumer **already configures Spotless itself**, Maven merges the parent's `<configuration>` into the child's, which can pull in `<googleJavaFormat/>` unexpectedly. A consumer with its own formatter (e.g. Palantir, a custom `eclipse` config) should keep its existing `<configuration>` explicit so the merge does not silently switch the formatter.

#### Additive: Git metadata stamped into the jar manifest (`-Pfull-build` only)

`1.1.0` adds `git-commit-id-maven-plugin` (`10.0.0`), version-managed in `<pluginManagement>` and wired into the **`full-build` profile only**. When a child builds with `-Pfull-build`, each jar's `MANIFEST.MF` gains `Git-Commit`, `Git-Commit-Time`, `Git-Branch`, `Git-Tag`, and `Git-Dirty` entries. No `git.properties` file is written (deliberately, to avoid classpath collisions), and the config is tuned for reproducible builds (UTC commit time, volatile properties excluded, `failOnNoGitDirectory=false`). A normal `clean verify` is unaffected. **No consumer action required.**

#### Internal (no consumer action)

These changed in the `java-parent` repository but have no consumer-facing effect: changes to the parent's own `release.yml` / `snapshot.yml` CI workflows, README updates, and `.gitignore` adjustments.

### Steps

1. In the consumer's `pom.xml`, set the `<parent>` `<version>` of `com.open-elements:java-parent` to `1.1.0`. Leave all other coordinates untouched.
2. Determine whether the consumer relies on **parameter-name retention**. Search its source for parameterless `@PathVariable` / `@RequestParam` / `@RequestHeader` / `@MatrixVariable`, Spring Data query methods, Jackson `ParameterNamesModule`, or any `Parameter#getName()` use. If found, add the `maven-compiler-plugin` `<parameters>true</parameters>` block (above) to the consumer's `<build><plugins>`.
3. Determine whether the consumer's **tests need deep reflection** into `java.base`. If the test suite previously passed only because of the inherited `--add-opens`, add the `maven-surefire-plugin` `argLine` block (above) to the consumer's `<build><plugins>` — using the `@{argLine}` form if JaCoCo is present.
4. (Optional) If you want Google Java Format, run `mvn spotless:apply`; otherwise do nothing — Spotless will not run in `verify`.
5. Run `mvn -U clean verify` (or the project's equivalent) and confirm the project compiles, **all tests pass at runtime**, and dependencies resolve before committing. Pay attention to runtime failures that a compile check would miss (the two breaking-light items surface only here).

### Guard rails

- Do **not** bump Spring Boot, Testcontainers, or any plugin version in the consumer to "match 1.1.0" — the BOM-managed versions are unchanged and the plugin set is internal to the parent.
- Add the `-parameters` block **only if** the consumer actually relies on parameter names; do not add it speculatively to every project.
- Add the Surefire `--add-opens` block **only if** the consumer's tests actually need it. If JaCoCo (or another `argLine` injector) is in play, preserve `@{argLine}` — do not clobber it with a bare `argLine`.
- Restore the two flags in the **consumer's own `pom.xml`**, not by editing `java-parent`.
- Do **not** add a Spotless `<execution>` that binds `check` to `verify` as part of this upgrade — that would turn an optional formatter into a build-breaking gate the parent never imposed.

### Don't do this

- Do not "shim" the change by pinning the consumer back to `1.0.0` in an intermediate parent so children keep inheriting the old flags — restore only the flags the consumer needs, explicitly.
- Do not blanket-add both `-parameters` and `--add-opens` to a project that demonstrably needs neither; the release intentionally stopped imposing them globally.
- Do not adopt Google Java Format reformatting (`spotless:apply`) in the same change as the version bump — a repo-wide reformat buried in an upgrade commit makes the diff unreviewable. Do it as a separate, isolated commit if at all.
- Do not bundle this upgrade with unrelated dependency bumps, plugin changes, or feature work in the same PR.
