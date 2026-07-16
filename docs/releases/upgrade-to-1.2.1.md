# Upgrade prompt: `com.open-elements:java-parent` 1.2.0 → 1.2.1

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its `<parent>`, moving from `1.2.0` to `1.2.1`. This is a **patch release with a single breaking-light change**: the parent now enables the Java compiler's `-parameters` flag for every child module, so formal parameter names are retained in the compiled bytecode. There are **no API/code changes**, **no BOM-managed version changes** (Spring Boot stays `3.5.14`, Testcontainers stays `2.0.5`), and **no new managed dependencies or plugins**. Consumer source compiles unchanged; the only observable difference is in the emitted bytecode.

Your job is to bump the `<parent>` version and rebuild. Apply exactly the change below and nothing outside this scope.

### What changed in 1.2.1

#### Dependencies

Bump only the `<parent>` version of `com.open-elements:java-parent` to `1.2.1`. **Nothing else changes**: `spring-boot-dependencies` stays at `3.5.14` and `testcontainers-bom` stays at `2.0.5`, both still imported via the parent's `<dependencyManagement>`. Do **not** bump those, and do **not** bump any Maven / JReleaser / CycloneDX / Spotless / git-commit-id / pomchecker plugin version in the consumer — the plugin set is unchanged from `1.2.0` and is internal to the parent with no consumer-facing version effect.

#### Breaking-light: `-parameters` is now enabled for all child modules

The parent's `<properties>` now sets:

```xml
<!-- new in 1.2.1 (parent <properties>) -->
<maven.compiler.parameters>true</maven.compiler.parameters>
```

This property is consumed by the Maven compiler plugin and adds `-parameters` to `javac`, so **formal parameter names survive into the `.class` files** (instead of being erased to `arg0`, `arg1`, …). In `1.2.0` this flag was **not** set, so parameter names were dropped. Every module that inherits from the parent picks this up automatically on the next compile — **no consumer `pom.xml` edit is required** beyond the version bump.

Why it matters and why it is only *breaking-light*:

- **Source is unaffected.** No signatures change; the consumer's code compiles exactly as before. This is purely a `javac` output-detail change.
- **This is a fix, not a new feature.** Frameworks that resolve things by parameter name now work reliably. In particular Spring uses parameter names to disambiguate constructor/method injection of same-type beans and to bind `@RequestParam` / `@PathVariable` / `@ConfigurationProperties` **without an explicit name**. On `1.2.0` such code could fail at runtime (e.g. `IllegalArgumentException: Name for argument of type [...] not specified`) unless the name was spelled out; on `1.2.1` it resolves from the retained parameter name. Spring Boot's own parent enables `-parameters` for exactly this reason.
- **The bytecode changes.** `.class` files now contain a `MethodParameters` attribute and are marginally larger. Two consequences to be aware of:
  - A **test that asserts parameter names are absent/synthetic** (e.g. checks `Parameter#isNamePresent()` is `false`, or that a name equals `"arg0"`) will now fail. This is the only realistic way this upgrade breaks a build. Fix the test to expect the real parameter name; do not disable the flag.
  - If the consumer compares build artifacts **byte-for-byte against a jar produced under `1.2.0`** (reproducible-build diffing), the class files will differ. A clean rebuild under `1.2.1` is still deterministic — re-baseline the comparison against a `1.2.1` build.

A consumer that explicitly set `<maven.compiler.parameters>false</maven.compiler.parameters>` in its own `pom.xml` keeps that override (the child property wins over the inherited one). Only remove such an override if you actually want `-parameters` — that is the recommended state and the reason for this release.

#### Internal (no consumer action)

The only other diff between `1.2.0` and `1.2.1` is release version churn in the parent's own `pom.xml` (`1.2.0` → `1.3.0-SNAPSHOT` → `1.2.1`) and the explanatory `<!-- ... -->` comment added next to the new property. Neither has any consumer-facing effect.

### Steps

1. In the consumer's `pom.xml`, set the `<parent>` `<version>` of `com.open-elements:java-parent` to `1.2.1`. Leave all other coordinates untouched.
2. Run a plain `mvn -U clean verify` and confirm the project compiles, tests pass, and dependencies resolve.
3. If a test fails because it asserted parameter names were **absent** or equal to `"arg0"`/`"arg1"`/…, update that test to expect the real parameter name. Do **not** disable `-parameters`.
4. If the consumer has an explicit `<maven.compiler.parameters>false</maven.compiler.parameters>` and *wants* the fix, remove that override so the inherited `true` applies. Otherwise leave the consumer's `pom.xml` unchanged.
5. If (and only if) the consumer performs byte-for-byte reproducible-build comparisons, re-baseline the reference artifact against a `1.2.1` build — the class files legitimately differ from `1.2.0`.
6. Confirm the build is green before committing.

### Guard rails

- Do **not** bump Spring Boot, Testcontainers, or any plugin version in the consumer to "match 1.2.1" — nothing in the BOM or plugin set changed.
- Do **not** add `<maven.compiler.parameters>false</maven.compiler.parameters>` to the consumer to "restore old behavior" — the retained parameter names are the intended fix; suppressing them re-introduces the Spring name-resolution bug this release fixes.
- Do **not** pass `-Dmaven.compiler.parameters=false` on the command line to work around a failing test — fix the test's assumption instead.
- Make any test fix in the **consumer's own test sources**, not by editing `java-parent`.

### Don't do this

- Do not disable, override, or command-line-suppress `-parameters` to make a name-asserting test pass — correct the assertion.
- Do not treat differing `.class` bytes versus a `1.2.0` build as a regression; it is the expected effect of this release.
- Do not bundle this upgrade with unrelated dependency bumps, plugin changes, or feature work in the same PR.
