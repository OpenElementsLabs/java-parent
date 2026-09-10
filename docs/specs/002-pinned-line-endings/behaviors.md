# Behaviors: Pinned line endings

Scenarios describe observable Git and build behaviour after `.gitattributes`,
`.editorconfig` and the Spotless `lineEndings` setting are in place. "Text file"
means any tracked file Git detects as text and that is not `*.bat` or `*.cmd`.

## Checkout

### Text file on a Windows default installation

- **Given** a Windows machine with `core.autocrlf=true`
- **When** the repository is cloned and checked out
- **Then** `pom.xml`, `README.md`, `release.sh` and every other text file are written
  to the working tree with LF line endings
- **And** `git ls-files --eol` reports `w/lf` for them

### Extensionless text file

- **Given** the same Windows machine
- **When** the repository is checked out
- **Then** `mvnw` and `.sdkmanrc` are written with LF, because the `*` baseline covers
  files that no extension rule would match
- **And** `mvnw` remains executable under Git Bash

### Windows batch file

- **Given** the same Windows machine
- **When** the repository is checked out
- **Then** `mvnw.cmd` is written with CRLF
- **And** `git ls-files --eol mvnw.cmd` reports `i/lf` and `w/crlf`
- **And** `mvnw.cmd` runs under `cmd.exe`

### Unix checkout is unchanged

- **Given** a macOS or Linux machine with `core.autocrlf=input` or unset
- **When** the repository is checked out
- **Then** every text file has LF, exactly as before the change
- **And** `mvnw.cmd` is written with CRLF, which is the only observable difference
  from the pre-change behaviour on these platforms

### Local Git configuration cannot override the pin

- **Given** a user who has explicitly set `core.autocrlf=true` on macOS
- **When** the repository is checked out
- **Then** text files still have LF, because a path-specific `eol` attribute takes
  precedence over `core.autocrlf`

### Binary assets are untouched

- **Given** any platform
- **When** the repository is checked out
- **Then** `.ttf`, `.png` and `.pdf` files are byte-identical to their index contents
- **And** their checksums match on macOS, Linux and Windows

## Committing

### A file authored with CRLF is normalized

- **Given** a Windows user who creates a new `.java` or `.md` file with CRLF endings
  in an editor that ignores `.editorconfig`
- **When** the file is staged and committed
- **Then** the blob stored in the index and in the commit uses LF
- **And** a reviewer on Linux sees no CRLF in the diff

### A batch file authored with CRLF is normalized in the index

- **Given** a new `*.cmd` file created on Windows with CRLF endings
- **When** it is staged and committed
- **Then** the index blob uses LF
- **And** a subsequent checkout writes it back with CRLF

### A binary file is committed verbatim

- **Given** a new `.png` added to the repository
- **When** it is staged and committed
- **Then** no line-ending conversion is applied and the blob equals the file on disk

## Introducing the change

### Renormalization changes nothing

- **Given** the repository immediately after `.gitattributes` is added
- **When** `git add --renormalize .` is run
- **Then** nothing is staged, because no tracked file carries CRLF in the index
- **And** the resulting commit contains only the new and modified files listed in the
  design, with no incidental whitespace changes

### The existing working tree stays valid

- **Given** the maintainer's existing macOS working tree, where `mvnw.cmd` is CRLF on
  disk and LF in the index
- **When** `.gitattributes` is added
- **Then** `git status` reports no modification for `mvnw.cmd`, because its working
  tree form now matches the declared `eol=crlf`

## Editor configuration

### Java formatting agrees with the formatter

- **Given** a child project that copies this `.editorconfig` and inherits the parent's
  Spotless configuration
- **When** a developer writes Java code following the IDE's indentation and line-length
  hints and then runs `./mvnw spotless:apply`
- **Then** Spotless makes no indentation or wrapping changes, because both are set to
  2 spaces and 100 columns

### Editor and Git agree on batch files

- **Given** an editor with EditorConfig support
- **When** a `*.cmd` file is opened and saved
- **Then** the editor writes CRLF, matching the Git attribute, and the file is not
  reported as modified afterwards

## Inherited Spotless behaviour

### Formatting on Windows without a child `.gitattributes`

- **Given** a child project that inherits this parent, has Java sources, and has no
  `.gitattributes` of its own, checked out on Windows with `core.autocrlf=true`
- **When** `./mvnw spotless:apply` is run
- **Then** every formatted Java file is written with LF, because the explicit
  `UNIX` setting overrides Spotless's `GIT_ATTRIBUTES` default

### A child can still override the setting

- **Given** a child project that sets `<lineEndings>` itself in its own Spotless
  configuration
- **When** `./mvnw spotless:apply` is run
- **Then** the child's value applies, following normal Maven plugin configuration
  inheritance

### The configuration is validated by the build

- **Given** the parent POM
- **When** `./mvnw spotless:check` is run in this repository
- **Then** the build succeeds and reports no files to format, since the repository
  contains no Java sources
- **And** an invalid `lineEndings` value would fail the plugin at configuration time

## Published artifact

### The deployed POM does not depend on the build platform

- **Given** the same commit checked out on Windows and on Linux
- **When** each build deploys `java-parent`
- **Then** the deployed `.pom` files are byte-identical
- **And** this holds because the working-tree `pom.xml` is identical on both
  platforms, not because of any build-time normalization

## Edge cases

### A new text file type nobody thought about

- **Given** a file type with no explicit rule, for example a newly added `.toml` or
  `.gradle.kts`
- **When** it is checked out on Windows
- **Then** it receives LF, because the `*` baseline governs everything Git detects as
  text

### A new binary type covered by an explicit marker

- **Given** a `.p12` keystore added to the repository
- **When** it is committed and checked out
- **Then** it is treated as binary by the explicit marker, independent of whether the
  content heuristic would have classified it correctly

### A new binary type not covered by any marker

- **Given** a binary file type absent from the marker list
- **When** it is committed
- **Then** Git's content heuristic decides, exactly as it does today
- **And** if the heuristic misclassifies it, the file may be corrupted by
  normalization — an accepted residual risk that the marker list narrows but does
  not eliminate

### A file with mixed line endings

- **Given** an existing text file containing both LF and CRLF
- **When** it is staged
- **Then** all line endings are normalized to LF in the index
- **And** `git ls-files --eol` reports `i/lf` rather than `i/mixed`

### A file with no line endings at all

- **Given** a single-line file with no trailing newline, of which the repository has
  several
- **When** it is checked out on any platform
- **Then** its content is unchanged and no newline is inserted

## Regression

### Weakening the attributes silently restores the old behaviour

- **Given** someone later changes the baseline to bare `* text=auto`
- **When** a Windows user with `core.autocrlf=true` checks the repository out
- **Then** text files are written with CRLF again
- **And** nothing in the build or CI fails, because no automated guard exists — this
  is the accepted consequence of the "no guard" decision and is recorded in
  `docs/TODO.md`
