# Implementation Steps: OpenAPI stack versions

Spec: [`design.md`](design.md) · [`behaviors.md`](behaviors.md) · Issue #6

`java-parent` has `packaging=pom` and no Java sources, so the fix cannot be exercised
by building the parent alone. The executable form of these scenarios is Maven
dependency resolution against a throwaway probe child, per
[Acceptance](design.md#acceptance).

---

## Step 1: Declare the coupled version properties

- [x] Add `swagger.version` and `swagger-ui.version` next to the existing
      `springdoc.version`
- [x] Write the lockstep comment above them: the three are coupled, mixing Swagger
      versions produces `NoSuchMethodError` inside Swagger itself, and the matching
      values are read from `<swagger-api.version>` / `<swagger-ui.version>` in
      `org/springdoc/springdoc-openapi/<version>/springdoc-openapi-<version>.pom`

**Acceptance criteria:**
- [x] `./mvnw help:evaluate -Dexpression=swagger.version -q -DforceStdout` prints `2.2.47`
- [x] `./mvnw help:evaluate -Dexpression=swagger-ui.version -q -DforceStdout` prints `5.32.2`
- [x] The values match springdoc 2.8.17's aggregator POM

**Related behaviors:** The lockstep values are discoverable

---

## Step 2: Import the BOMs and manage the Swagger UI webjar

- [x] Import `org.springdoc:springdoc-openapi-bom` at `${springdoc.version}`
- [x] Import `io.swagger.core.v3:swagger-bom` at `${swagger.version}`
- [x] Manage `org.webjars:swagger-ui` at `${swagger-ui.version}`, with a comment
      noting no BOM covers it
- [x] Remove the explicit `springdoc-openapi-starter-webmvc-ui` entry the BOM replaces

**Acceptance criteria:**
- [x] `./mvnw -Pfull-build clean verify` passes, including pomchecker
- [x] The effective POM manages every springdoc starter and all 14 Swagger coordinates
- [x] No coordinate is managed twice

**Related behaviors:** springdoc resolves without a version · Other springdoc starters
are now usable without a version · No overlap with the Spring Boot BOM · An
unresolvable BOM fails fast

---

## Step 3: Prove the fix against a probe child

- [x] Install the modified parent locally with `./mvnw -N install`
- [x] Create a throwaway child inheriting it that declares
      `springdoc-openapi-starter-webmvc-ui` without a version, plus a dependency
      that contributes `swagger-annotations-jakarta` 2.2.29
- [x] Resolve the tree and record the Swagger and Swagger UI versions
- [x] Repeat against the **unmodified** parent to confirm the probe reproduces the
      split — otherwise the check passes vacuously and proves nothing
- [x] Record both trees for the pull request description

**Acceptance criteria:**
- [x] Against the modified parent: `swagger-annotations-jakarta`,
      `swagger-models-jakarta` and `swagger-core-jakarta` all resolve to 2.2.47, and
      `swagger-ui` to 5.32.2
- [x] Against the unmodified parent: `swagger-annotations-jakarta` resolves to 2.2.29
      while `swagger-core-jakarta` resolves to 2.2.47

**Related behaviors:** The Swagger stack is uniform · The Swagger UI webjar is pinned ·
A library contributing an older annotations artifact no longer splits the stack · The
same graph fails on the previous parent version · A direct pin in the consumer still
wins · Dropping the direct pin activates the fix

---

## Step 4: Document the managed stack and the maintenance rule

- [x] Add the OpenAPI stack to the README's managed-versions section
- [x] Add a maintenance subsection describing the coupling, the failure it prevents,
      and the concrete lookup path for a future springdoc bump

**Acceptance criteria:**
- [x] The lookup path names the exact artifact and the exact properties
- [x] The consequence of getting it wrong is stated: the parent would publish a split
      stack to every child at once

**Related behaviors:** The lockstep values are discoverable · Bumping springdoc alone
reintroduces the failure

---

## Step 5: Final verification

- [x] `./mvnw -Pfull-build clean verify`
- [x] `docs/specs/INDEX.md` status updated

**Acceptance criteria:**
- [x] Build passes
- [x] The probe project is removed and leaves no trace in the repository

**Related behaviors:** The parent still builds and validates

---

## Behavior Coverage

18 scenarios. Layer is dependency resolution throughout. All rows marked *Measured*
were confirmed against throwaway probe projects during implementation; the probes
were removed afterwards and left no artifacts behind.

| Scenario | Verification | Step |
|---|---|---|
| springdoc resolves without a version | Measured — probe child resolves 2.8.17 | 2, 3 |
| Other springdoc starters are now usable without a version | Measured — `webflux-ui` resolves to 2.8.17 without a version | 3 |
| The Swagger stack is uniform | Measured — annotations, models and core all 2.2.47 | 3 |
| The Swagger UI webjar is pinned | Measured — 5.32.2; note the probe did not create a competing webjar version, so the pin is preventive | 3 |
| A library contributing an older annotations artifact no longer splits the stack | Measured — 2.2.29 contributor is overridden to 2.2.47 | 3 |
| The same graph fails on the previous parent version | Measured — pre-change parent yields annotations 2.2.29 next to core 2.2.47 | 3 |
| Depth of the conflicting declaration does not matter | Follows from management semantics; the probe covers depth 2 | 3 |
| A direct pin in the consumer still wins | Measured — direct 2.2.29 declaration survives the fix | 3 |
| Dropping the direct pin activates the fix | Measured — removing `<version>` yields 2.2.47 | 3 |
| The javax Swagger line becomes managed | Measured — `swagger-annotations` and `swagger-jaxrs2-jakarta` resolve to 2.2.47 | 3 |
| A consumer using the JAX-RS artifacts is affected the same way | Measured together with the row above | 3 |
| No overlap with the Spring Boot BOM | Verified against `spring-boot-dependencies` 3.5.14 | 2 |
| Bumping springdoc alone reintroduces the failure | Not executable — documented risk | 4 |
| The lockstep values are discoverable | Lookup path followed once during implementation | 1, 4 |
| The parent still builds and validates | `./mvnw -Pfull-build clean verify` | 2, 5 |
| An unresolvable BOM fails fast | Measured — a child setting `swagger.version=2.2.38` fails at model building | 3 |
| A consumer may still override deliberately | Measured — `swagger.version=2.2.50` moves all Swagger coordinates | 3 |
| Overriding the Swagger version below 2.2.47 fails the build | Measured — `swagger-bom` does not exist below 2.2.47 | 3 |
