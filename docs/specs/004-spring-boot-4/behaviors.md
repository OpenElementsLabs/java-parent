# Behaviors: Spring Boot 4

Scenarios describe a project inheriting `java-parent` 2.0.0 ("consumer").

## Version management

### Spring Boot 4 is managed

- **Given** a consumer that declares `spring-boot-starter-webmvc` without a `<version>`
- **When** the project resolves its dependencies
- **Then** Spring Boot 4.1.2 and Spring Framework 7.0.10 are used

### The OpenAPI stack matches Boot 4

- **Given** a consumer that declares `springdoc-openapi-starter-webmvc-ui` without a
  `<version>`
- **When** `mvn dependency:tree -Dincludes=org.springdoc,io.swagger.core.v3,org.webjars:swagger-ui`
  is run
- **Then** springdoc resolves to 3.2.0
- **And** `swagger-annotations-jakarta`, `swagger-models-jakarta` and
  `swagger-core-jakarta` all resolve to 2.2.55
- **And** `swagger-ui` resolves to 5.34.0

### Testcontainers is unchanged

- **Given** a consumer that declares `testcontainers-junit-jupiter` without a `<version>`
- **When** the project resolves its dependencies
- **Then** Testcontainers 2.0.5 is used

### jspecify is not pinned below Boot

- **Given** a consumer that declares `org.jspecify:jspecify` without a `<version>`
- **When** the project resolves its dependencies
- **Then** jspecify 1.0.1 is used — the version Boot 4.1.2 manages — with `compile`
  scope

## Java baseline

### Sources compile for Java 25

- **Given** a consumer with one Java source file and no compiler configuration
- **When** it is built on JDK 25
- **Then** the class file has major version 69 (Java 25)

### An older JDK fails fast

- **Given** the same consumer
- **When** it is built on JDK 21
- **Then** the build fails in `validate` with the `requireJavaVersion` enforcer rule
- **And** no compilation is attempted

### A child may still raise the baseline

- **Given** a consumer that sets `enforcer.requiredJavaVersion` to a higher version
- **When** it is built on a JDK below that version
- **Then** the enforcer fails, as in 1.x

## The parent itself

### The parent builds and validates on JDK 25

- **Given** this repository on JDK 25
- **When** `./mvnw -Pfull-build clean verify` is run
- **Then** the build passes, including the enforcer, pomchecker and the CycloneDX SBOM

### Reproducible-build settings are unchanged

- **Given** the 2.0.0-SNAPSHOT POM
- **When** `project.build.outputTimestamp` is evaluated
- **Then** it is the literal inherited from the 1.3.0 release, untouched by this change
