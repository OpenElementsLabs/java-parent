# Implementation Steps: Pinned line endings

Spec: [`design.md`](design.md) · [`behaviors.md`](behaviors.md) · Issue #4

This spec changes build and repository configuration in a `packaging=pom` project
with no Java sources and no test framework. "Tests" are therefore Git and Maven
verification commands, not JUnit cases — the checks below are the executable form of
the behaviour scenarios. Scenarios that can only be observed on Windows are recorded
as specified-but-unverified, consistent with the design's
[no-guard decision](design.md#acceptance).

---

## Step 1: Add `.gitattributes`

- [x] Create `.gitattributes` with the `* text=auto eol=lf` baseline
- [x] Add the `*.bat` / `*.cmd` → `eol=crlf` exception with its rationale comment
- [x] Add explicit `binary` markers for the asset types present plus the binary types
      a Java repository predictably acquires
- [x] Run `git add --renormalize .` and confirm it stages nothing

**Acceptance criteria:**
- [x] `git check-attr text eol -- pom.xml` reports `text: auto` and `eol: lf`
- [x] `git check-attr eol -- mvnw.cmd` reports `eol: crlf`
- [x] `git check-attr text -- <a .ttf asset>` reports `text: unset`
- [x] `git ls-files --eol` shows no `i/crlf` and no `i/mixed`
- [x] `git add --renormalize .` produces no staged changes
- [x] `git status` reports no modification for `mvnw.cmd`

**Related behaviors:** Text file on a Windows default installation · Extensionless
text file · Windows batch file · Unix checkout is unchanged · Local Git configuration
cannot override the pin · Binary assets are untouched · A file authored with CRLF is
normalized · A batch file authored with CRLF is normalized in the index · A binary
file is committed verbatim · Renormalization changes nothing · The existing working
tree stays valid · A new text file type nobody thought about · A new binary type
covered by an explicit marker · A new binary type not covered by any marker · A file
with mixed line endings · A file with no line endings at all

---

## Step 2: Add `.editorconfig`

- [x] Create `.editorconfig` from the Open Elements standard
      (`.claude/skills/project-setup/references/editorconfig.md`)
- [x] Correct `[*.java]` to Google Java Format: `indent_size = 2`,
      `ij_continuation_indent_size = 4`, `max_line_length = 100`
- [x] Add a `[*.{cmd,bat}]` block with `end_of_line = crlf` so the editor agrees with
      the Git attribute
- [x] Comment the deviation from the org standard at the point of deviation

**Acceptance criteria:**
- [x] `[*] end_of_line = lf` and `charset = utf-8` are present
- [x] The `[*.java]` block specifies 2 spaces and 100 columns, matching
      `googleJavaFormat`
- [x] The `[*.{cmd,bat}]` block specifies CRLF, matching `.gitattributes`
- [x] The `ij_java_*` rules from the org standard are carried over unchanged

**Related behaviors:** Java formatting agrees with the formatter · Editor and Git
agree on batch files

---

## Step 3: Force UNIX line endings in the inherited Spotless configuration

- [x] Add `<lineEndings>UNIX</lineEndings>` to the Spotless plugin configuration in
      `pom.xml`, above the existing `<java>` block
- [x] Comment why the `GIT_ATTRIBUTES` default is insufficient for child projects

**Acceptance criteria:**
- [x] `./mvnw spotless:check` passes — an invalid enum value would fail the plugin at
      configuration time
- [x] `./mvnw clean verify` passes
- [x] `./mvnw -Pfull-build clean verify` passes, including pomchecker

**Related behaviors:** Formatting on Windows without a child `.gitattributes` · A
child can still override the setting · The configuration is validated by the build

---

## Step 4: Document the pin in the README

- [x] Add a **Line endings** section describing what is pinned and why the parent's
      own published `.pom` is affected
- [x] Include a copy-paste `.gitattributes` block for child projects
- [x] State what the inherited Spotless setting does and does not cover
- [x] Reference the section from the existing Build conventions list

**Acceptance criteria:**
- [x] The section states the narrow claim: line endings are eliminated as a variance
      source; cross-OS reproducibility is not claimed
- [x] The copy-paste block matches the repository's own `.gitattributes`
- [x] No promise is made that the parent delivers `.gitattributes` to children

**Related behaviors:** The lockstep between editor and Git is documented (supporting
Java formatting agrees with the formatter)

---

## Step 5: Final verification

- [x] `git ls-files --eol` audit across the whole tree
- [x] `git add --renormalize .` is a no-op
- [x] `./mvnw -Pfull-build clean verify` passes
- [x] `docs/specs/INDEX.md` status is correct

**Acceptance criteria:**
- [x] All of the above pass
- [x] The diff contains no incidental whitespace changes to existing files

**Related behaviors:** Weakening the attributes silently restores the old behaviour
(documented, deliberately unguarded)

---

## Behavior Coverage

23 scenarios. Layer is "Repo/Build" throughout — there is no application code.

| Scenario | Verifiable here | Covered in Step |
|---|---|---|
| Text file on a Windows default installation | No — Windows only | 1 (attribute asserted) |
| Extensionless text file | Partly — attribute asserted | 1 |
| Windows batch file | Partly — attribute asserted | 1 |
| Unix checkout is unchanged | Yes | 1 |
| Local Git configuration cannot override the pin | Yes — `core.autocrlf` override test | 1 |
| Binary assets are untouched | Yes | 1 |
| A file authored with CRLF is normalized | Yes — CRLF probe file | 1 |
| A batch file authored with CRLF is normalized in the index | Yes — CRLF probe file | 1 |
| A binary file is committed verbatim | Yes | 1 |
| Renormalization changes nothing | Yes | 1, 5 |
| The existing working tree stays valid | Yes | 1 |
| Java formatting agrees with the formatter | Yes — by inspection, no Java sources here | 2 |
| Editor and Git agree on batch files | Yes — by inspection | 2 |
| Formatting on Windows without a child `.gitattributes` | No — needs a child project | 3 (config asserted) |
| A child can still override the setting | No — needs a child project | 3 |
| The configuration is validated by the build | Yes | 3 |
| The deployed POM does not depend on the build platform | No — needs two platforms | 1 (follows from the attribute) |
| A new text file type nobody thought about | Yes — probe file | 1 |
| A new binary type covered by an explicit marker | Yes — probe file | 1 |
| A new binary type not covered by any marker | Yes — documented residual risk | 1 |
| A file with mixed line endings | Yes — probe file | 1 |
| A file with no line endings at all | Yes | 1 |
| Weakening the attributes silently restores the old behaviour | No — regression, unguarded by decision | 5 (documented) |

Scenarios marked "No" are the Windows- and child-project-dependent ones the design
accepted as unverified; both gaps are recorded in `docs/TODO.md`.
