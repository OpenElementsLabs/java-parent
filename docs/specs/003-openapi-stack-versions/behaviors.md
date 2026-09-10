# Behaviors: OpenAPI stack versions

Scenarios describe dependency resolution in a project inheriting `java-parent` after
`springdoc-openapi-bom` and `swagger-bom` are imported and `org.webjars:swagger-ui`
is managed. "Consumer" means a project whose parent is `java-parent`.

## Version management

### springdoc resolves without a version

- **Given** a consumer that declares `springdoc-openapi-starter-webmvc-ui` with no
  `<version>`
- **When** the project resolves its dependencies
- **Then** springdoc 2.8.17 is used, as before the change
- **And** the version comes from `springdoc-openapi-bom`, not from an explicit
  dependency entry in the parent

### Other springdoc starters are now usable without a version

- **Given** a consumer that declares `springdoc-openapi-starter-webflux-ui` with no
  `<version>`
- **When** the project resolves its dependencies
- **Then** it resolves to 2.8.17, because the BOM manages every springdoc starter
- **And** before this change the build would have failed with a missing version

### The Swagger stack is uniform

- **Given** a consumer that declares only `springdoc-openapi-starter-webmvc-ui`
- **When** `mvn dependency:tree -Dincludes=io.swagger.core.v3` is run
- **Then** `swagger-annotations-jakarta`, `swagger-models-jakarta` and
  `swagger-core-jakarta` all resolve to 2.2.47

### The Swagger UI webjar is pinned

- **Given** the same consumer
- **When** `mvn dependency:tree -Dincludes=org.webjars:swagger-ui` is run
- **Then** `swagger-ui` resolves to 5.32.2

## The bug this fixes

### A library contributing an older annotations artifact no longer splits the stack

- **Given** a consumer that declares `springdoc-openapi-starter-webmvc-ui` without a
  version
- **And** a library dependency that declares `swagger-annotations-jakarta` 2.2.29
  with an explicit version
- **When** the project resolves its dependencies
- **Then** `swagger-annotations-jakarta` resolves to 2.2.47, because management is
  consulted before nearest-wins mediation
- **And** the application serves `/v3/api-docs` without a `NoSuchMethodError`

### The same graph fails on the previous parent version

- **Given** the identical consumer built against the parent version preceding this
  change
- **When** the project resolves its dependencies
- **Then** `swagger-annotations-jakarta` resolves to 2.2.29 while
  `swagger-core-jakarta` resolves to 2.2.47
- **And** schema resolution throws
  `NoSuchMethodError: io.swagger.v3.oas.annotations.media.Schema.$dynamicRef()`
- **And** this scenario exists to prove that the acceptance probe reproduces the
  problem rather than passing vacuously

### Depth of the conflicting declaration does not matter

- **Given** a consumer whose dependency graph contains `swagger-models-jakarta` 2.2.29
  four levels deep
- **When** the project resolves its dependencies
- **Then** it still resolves to 2.2.47, because `dependencyManagement` applies at any
  depth

## Limits of the fix

### A direct pin in the consumer still wins

- **Given** a consumer that declares `swagger-annotations-jakarta` 2.2.29 **directly
  with an explicit version** in its own POM
- **When** the project resolves its dependencies
- **Then** 2.2.29 is used, because management does not override a direct declaration
- **And** the split stack persists, with no warning from the parent

### Dropping the direct pin activates the fix

- **Given** the same consumer
- **When** the `<version>` is removed from its direct
  `swagger-annotations-jakarta` declaration
- **Then** it resolves to 2.2.47 from the managed set

### A consumer may still override deliberately

- **Given** a consumer that declares its own `swagger.version` property
- **When** the project resolves its dependencies
- **Then** the whole Swagger stack moves to that version, because property
  interpolation happens in the child's effective POM and therefore reaches the
  parent's BOM import
- **And** one property in the child retargets all 14 coordinates at once

### Overriding the Swagger version below 2.2.47 fails the build

- **Given** a consumer that sets `swagger.version` to a release older than 2.2.47
- **When** the project is built
- **Then** the build fails at model-building time with a non-resolvable import POM,
  because `io.swagger.core.v3:swagger-bom` was first published at 2.2.47
- **And** the failure is explicit rather than silent
- **And** declaring the artifact directly with a `<version>` is unaffected and remains
  the way to use an older Swagger

## Side effects

### The javax Swagger line becomes managed

- **Given** a consumer that declares `io.swagger.core.v3:swagger-annotations` — the
  non-jakarta artifact — with no version
- **When** the project resolves its dependencies
- **Then** it resolves to 2.2.47, a version selected for springdoc compatibility
  rather than for that consumer
- **And** the build succeeds, so the side effect is silent

### A consumer using the JAX-RS artifacts is affected the same way

- **Given** a consumer that declares `swagger-jaxrs2-jakarta` with no version
- **When** the project resolves its dependencies
- **Then** it resolves to 2.2.47, because `swagger-bom` manages all 14 coordinates

### No overlap with the Spring Boot BOM

- **Given** the parent's existing `spring-boot-dependencies` 3.5.14 import
- **When** the effective POM is computed
- **Then** no coordinate is managed by both: `spring-boot-dependencies` 3.5.14
  contains no `io.swagger` and no `org.springdoc` entry at all, and its only
  `org.webjars` entries are `webjars-locator-core` and `webjars-locator-lite`
- **And** the relative order of the imports has no effect on the resolved versions

## Maintenance

### Bumping springdoc alone reintroduces the failure

- **Given** a maintainer who raises `springdoc.version` to a release built against a
  newer Swagger, leaving `swagger.version` unchanged
- **When** a consumer builds against the resulting parent release
- **Then** the split stack returns, now for every consumer at once
- **And** nothing in the parent's own build fails, because the parent has no Java
  sources and never exercises the stack
- **And** this is the accepted cost of documenting the lockstep instead of enforcing
  it, recorded in `docs/TODO.md`

### The lockstep values are discoverable

- **Given** a maintainer bumping `springdoc.version` to a new release
- **When** they follow the comment on the property in `pom.xml`
- **Then** opening `org/springdoc/springdoc-openapi/<version>/springdoc-openapi-<version>.pom`
  and reading `<swagger-api.version>` and `<swagger-ui.version>` yields the two
  matching values
- **And** the README maintenance section describes the same path

## Parent build

### The parent still builds and validates

- **Given** the modified parent
- **When** `./mvnw clean verify` is run
- **Then** the build succeeds
- **And** `./mvnw -Pfull-build clean verify` passes pomchecker, which validates the
  POM against Maven Central's publishing rules

### An unresolvable BOM fails fast

- **Given** a typo in one of the three version properties
- **When** the parent or any child is built
- **Then** resolution of the imported BOM fails with a clear missing-artifact error
  at model-building time, before compilation
