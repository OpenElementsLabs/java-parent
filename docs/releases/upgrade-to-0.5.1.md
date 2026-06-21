# Upgrade prompt: `com.open-elements:java-parent` 0.5.0 → 0.5.1

## Prompt

You are upgrading a Maven project that uses `com.open-elements:java-parent` as its `<parent>`, moving from `0.5.0` to `0.5.1`. This release is a **cleanup release**: it contains exactly one consumer-relevant change — the parent POM no longer declares the `central-portal-snapshots` snapshot `<repositories>` block, which child projects previously inherited. There are **no dependency-version changes, no plugin-version changes, and no API changes**. For the typical consumer that resolves only released artifacts from Maven Central, this is a **do-nothing-and-it-still-works** bump. Only one case requires action (see the Breaking-light change below). Do not change anything outside this scope.

### What changed in 0.5.1

#### Dependencies

Bump only the `<parent>` version of `com.open-elements:java-parent` to `0.5.1`. **No managed dependency or plugin versions changed** between 0.5.0 and 0.5.1 — Spring Boot stays at `3.5.14`, Testcontainers stays at `2.0.5`, Slack API client stays at `1.45.3`, JSpecify stays at `1.0.0`, WireMock stays at `3.10.0`, and all `maven-*` / JReleaser / CycloneDX plugin versions are unchanged. Do **not** bump any of those coordinates as part of this upgrade.

#### Breaking-light: inherited `central-portal-snapshots` repository removed

In 0.5.0 the parent POM declared a `<repositories>` block that child projects inherited:

```xml
<!-- present in 0.5.0, REMOVED in 0.5.1 -->
<repositories>
    <repository>
        <id>central-portal-snapshots</id>
        <name>Central Portal Snapshots</name>
        <url>https://central.sonatype.com/repository/maven-snapshots/</url>
        <releases>
            <enabled>false</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>
```

In 0.5.1 this block is gone. The parent no longer contributes any `<repositories>` entry to its children.

This still compiles and the project model stays valid — that is why it is **breaking-light, not breaking**. The only consumer affected is one that **relied on inheriting this snapshot repository to resolve `-SNAPSHOT` dependencies** (for example, pre-release Open Elements artifacts published to the Central Portal snapshot repository). After upgrading, such a build will fail at dependency resolution with a "could not resolve" / "not found in any repository" error for those snapshot artifacts.

- **If the consumer only uses released versions** (the normal case): no action is required. Released artifacts come from Maven Central, which Maven resolves by default; nothing changes.
- **If the consumer genuinely needs Central Portal snapshots**: declare the repository explicitly in the consumer's own `pom.xml` (paste the block above), rather than relying on the parent to provide it.

### Steps

1. In the consumer's `pom.xml`, set the `<parent>` `<version>` of `com.open-elements:java-parent` to `0.5.1`. Leave all other coordinates untouched.
2. Determine whether the consumer depends on any `-SNAPSHOT` artifact that was previously resolved through the inherited `central-portal-snapshots` repository (search the build for `-SNAPSHOT` dependency versions, and check whether the consumer declares its own snapshot repository already).
3. Only if step 2 found such a dependency **and** no equivalent repository is declared in the consumer: add the `central-portal-snapshots` `<repositories>` block (above) directly to the consumer's `pom.xml`.
4. Run `./mvnw -version`-equivalent resolution — e.g. `mvn -U dependency:resolve` (or a clean build) — and confirm all dependencies resolve.
5. Build and run the test suite; confirm green before committing.

### Guard rails

- Do **not** bump Spring Boot, Testcontainers, Slack API client, JSpecify, WireMock, or any Maven/JReleaser/CycloneDX plugin version as part of this upgrade — none of them changed in 0.5.1.
- Do **not** add the `central-portal-snapshots` repository to the consumer unless the consumer actually resolves a `-SNAPSHOT` dependency that depended on it; adding it speculatively pulls snapshot metadata into a build that should use only releases.
- Do **not** add the removed repository back to `java-parent` itself — the removal is intentional; consumers that need it declare it locally.
- Do **not** treat this as an API or behavior change: there are no changed signatures, defaults, or message strings to fix.

### Don't do this

- Do not "fix" snapshot-resolution failures by enabling snapshots on Maven Central or on an unrelated repository — declare the specific Central Portal snapshot repository instead.
- Do not pin or upgrade managed dependency versions in the consumer to "match 0.5.1" — the BOM-managed versions are unchanged.
- Do not bundle this upgrade with unrelated dependency bumps or feature work in the same PR.
