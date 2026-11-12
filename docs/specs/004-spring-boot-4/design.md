# Design: Spring Boot 4

## GitHub Issue

[#8 — Upgrade managed Spring Boot to 4.1](https://github.com/OpenElementsLabs/java-parent/issues/8)

## Summary

`java-parent` 1.x manages Spring Boot 3.5.14. Downstream projects (first of all
spring-services, OpenElementsLabs/spring-services#58) are moving to Spring Boot 4 and
deliberately do not override the Boot version themselves: a `spring-boot.version`
property in a child would shadow the parent's and silently pin its BOM import. A parent
release that manages Boot 4 is therefore a prerequisite for that move.

Every consumer of such a parent must be on Boot 4, so this is a new major version:
**`java-parent` 2.0.0**. The version lines are:

| `java-parent` | Spring Boot | Java baseline |
|---|---|---|
| 1.x | 3.5 | 21 |
| 2.x | 4.1 | 25 |

Projects that stay on Boot 3.5 keep using 1.x. No further 1.x release is planned by
this spec.

## Goals

- The parent manages Spring Boot 4.1.2 and an OpenAPI stack built against it.
- A child moves from 1.x to 2.0.0 by changing only `<parent><version>` and its JDK —
  the parent makes the Java baseline decision, children make none of their own.
- The managed versions stay aligned with what Boot 4.1.2 itself manages, so the parent
  never pins a coordinate *below* Boot's choice.

## Non-goals

- **No migration of consumer code.** Boot 4's own breaking changes (renamed starters,
  Jackson 3, Spring Framework 7, Hibernate 7) are the consumer's concern and are
  handled in the consumer's own upgrade (e.g. spring-services spec 025).
- **No management of `spring-boot-maven-plugin`.** A `scope=import` BOM contributes
  `dependencyManagement` only, never `pluginManagement`, so the plugin was never
  inherited through `spring-boot-dependencies` — in 1.x neither. Children declare it
  with an explicit version, as before. Unchanged by this spec.
- **No maintenance branch for 1.x.** Created only if a 1.x fix is actually needed.

## Version changes

| Property | 1.x | 2.0.0 | Source of truth |
|---|---|---|---|
| `spring-boot.version` | 3.5.14 | **4.1.2** | Brings Spring Framework 7.0.10, Hibernate 7.4.13.Final, Jackson 3.1.7 |
| `springdoc.version` | 2.8.17 | **3.2.0** | springdoc 2.x supports Boot 3 only; 3.2.0's aggregator POM has `spring-boot-starter-parent` 4.1.2 as parent |
| `swagger.version` | 2.2.47 | **2.2.55** | `<swagger-api.version>` in `springdoc-openapi-3.2.0.pom` |
| `swagger-ui.version` | 5.32.2 | **5.34.0** | `<swagger-ui.version>` in `springdoc-openapi-3.2.0.pom` |
| `testcontainers.version` | 2.0.5 | 2.0.5 | Identical to the version Boot 4.1.2 manages |
| `jspecify.version` | 1.0.0 | **1.0.1** | Boot 4.1.2 manages 1.0.1 |
| `maven.compiler.source` / `target` | 21 | **25** | Java baseline decision, see below |
| `enforcer.requiredJavaVersion` | 21 | **25** | Follows the baseline |

### jspecify: not in the issue, but required by the third goal

The issue lists jspecify as unchanged at 1.0.0. Boot 4.1.2's BOM manages
`org.jspecify:jspecify` at **1.0.1**, and the parent's explicit `dependencyManagement`
entry takes precedence over any imported BOM. Keeping 1.0.0 would therefore actively
*downgrade* jspecify below what Spring Framework 7 and Boot 4 were built and tested
with. 1.0.1 is a patch release; the explicit entry is kept because it also sets
`scope=compile`.

### Java baseline: 25

Boot 4 requires Java 17+; both 21 and 25 are LTS. 25 was chosen so that the major
version that already forces every consumer through a framework migration also carries
the JDK move, instead of a second breaking parent release later. The cost: consumers
must upgrade their JDK together with Boot 4. The decision lives in the parent — children
inherit `maven.compiler.*` and `enforcer.requiredJavaVersion` and do not set them.

Consequences inside this repository: the CI workflows (`build.yml`, `snapshot.yml`,
`release.yml`) and `.sdkmanrc` move to Java 25, because the enforcer rule would
otherwise fail this project's own build.

## Consumer notes (for the release notes)

- **2.x = Spring Boot 4, 1.x = Spring Boot 3.5.**
- **Java 25 is required** (enforced in `validate`).
- **Jackson 2 next to Jackson 3.** `swagger-core` 2.2.55, used by springdoc 3.2.0,
  still depends on `com.fasterxml.jackson.core:jackson-databind` (Jackson 2), while
  Boot 4 uses Jackson 3 (`tools.jackson`). Applications using springdoc have both on
  the runtime classpath. Libraries using only `swagger-annotations-jakarta` are not
  affected — it has no dependencies.

## Files changed

| File | Change |
|---|---|
| `pom.xml` | Version `2.0.0-SNAPSHOT`; properties per the table above |
| `README.md` | Version lines table, updated managed versions, Java 25, Boot 4 pairing in the OpenAPI reference list |
| `.github/workflows/*.yml` | `java-version: "25"` |
| `.sdkmanrc` | `java=25` |

## Regression risk

- **Every consumer breaks on upgrade by design.** Boot 4 and Java 25 are both breaking.
  The major version number is the signal; 1.x stays available.
- **The OpenAPI lockstep from spec 003 applies unchanged.** The three values were read
  from springdoc 3.2.0's aggregator POM; a future springdoc bump needs the same lookup.
- **No overlap with the Boot BOM.** Boot 4.1.2 manages neither springdoc nor Swagger, so
  the OpenAPI imports do not compete with it.

## Acceptance

`java-parent` has no Java sources, so acceptance is a manual check against a throwaway
probe child (same approach as specs 001 and 003):

1. `./mvnw -Pfull-build clean verify` passes on JDK 25 (enforcer, pomchecker, SBOM).
2. `./mvnw -N install`, then a probe child inheriting `2.0.0-SNAPSHOT` with
   `spring-boot-starter-webmvc`, `springdoc-openapi-starter-webmvc-ui`, `jspecify` and
   `testcontainers-junit-jupiter` (all without `<version>`) plus one Java source file.
3. `dependency:tree` shows Boot 4.1.2, Spring Framework 7.0.10, springdoc 3.2.0, all
   Swagger jakarta artifacts at 2.2.55, `swagger-ui` 5.34.0, jspecify 1.0.1,
   Testcontainers 2.0.5.
4. The probe compiles with release 25 (class file major version 69).
5. Running the probe on JDK 21 fails in `validate` with the enforcer message.
