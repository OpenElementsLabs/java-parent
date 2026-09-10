# Spec Index

| ID  | Spec-Folder | Name | Areas | Description | GitHub Issue | Status |
|-----|-------------|------|-------|-------------|--------------|--------|
| 001 | 001-reproducible-build-timestamp | Reproducible build timestamp | build, infrastructure, documentation | Fixed `project.build.outputTimestamp` literal in the parent POM, inherited by all child projects, maintained by `release.sh` — so a third party can rebuild any Open Elements Java artifact byte-identically | #3 | done |
| 002 | 002-pinned-line-endings | Pinned line endings | build, infrastructure, documentation | `.gitattributes` forcing LF (CRLF for `*.bat`/`*.cmd`), a matching `.editorconfig` aligned with Google Java Format, and an inherited Spotless `lineEndings=UNIX` — so a checkout produces the same text bytes on every platform | #4 | done |
| 003 | 003-openapi-stack-versions | OpenAPI stack versions | build, api, documentation | Import `springdoc-openapi-bom` and `swagger-bom` and manage `org.webjars:swagger-ui`, so the coupled OpenAPI stack resolves uniformly in every consumer instead of splitting into a `NoSuchMethodError` | — | open |
