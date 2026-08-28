# Implementation Steps: Reproducible build timestamp

## Note on verification

This spec changes build configuration in a `packaging=pom` project with no source
code and no test framework. There is no unit test to write and — by explicit decision
recorded in `design.md` — no automated reproducibility check either; that is deferred
in [`docs/TODO.md`](../../TODO.md).

Verification is therefore of two kinds, and the steps say which applies:

- **Inspection** — the configuration is read and confirmed to say what the spec requires.
- **Execution** — a real build is run and its output measured (Step 5).

Scenarios that describe future release runs or third-party behaviour cannot be
executed here without cutting a real release; those are verified by inspection of the
mechanism plus the Step 5 measurement of the underlying property. This is stated
honestly in the coverage table rather than claimed as test coverage.

---

## Step 1: Pin the build timestamp in the parent POM

- [x] Add `project.build.outputTimestamp` to `<properties>` in `pom.xml`, seeded with
      the current UTC date at midnight (`2026-08-28T00:00:00Z`)
- [x] Add a comment explaining that the value is inherited by all children, is
      maintained by `release.sh`, and is not a build time

**Acceptance criteria:**
- [x] `./mvnw validate` succeeds
- [x] `./mvnw help:evaluate -Dexpression=project.build.outputTimestamp -q -DforceStdout`
      prints the literal
- [x] The value matches `^\d{4}-\d{2}-\d{2}T00:00:00Z$`

**Related behaviors:** A child inherits the parent's value without declaring anything;
No build-time field is introduced

---

## Step 2: Remove the command-line timestamp override from both workflows

- [ ] `.github/workflows/release.yml` — drop the `BUILD_TS` computation and the
      `-Dproject.build.outputTimestamp="$BUILD_TS"` argument
- [ ] `.github/workflows/snapshot.yml` — same
- [ ] Rewrite both step comments to explain that the timestamp now comes from the POM
      and that CI deliberately runs the same command a third party runs

**Acceptance criteria:**
- [ ] `grep -r "outputTimestamp" .github/workflows/` returns no matches
- [ ] Both workflows still parse as valid YAML
- [ ] The release build command differs from the documented public command only in
      deployment arguments

**Related behaviors:** The release workflow builds without a timestamp flag; The
snapshot workflow builds without a timestamp flag; CI and a third party run the same
command

---

## Step 3: Make `release.sh` maintain the timestamp

- [ ] After `versions:set -DnewVersion=$NEW_VERSION`, compute
      `RELEASE_DATE=$(date -u +%Y-%m-%dT00:00:00Z)`
- [ ] Rewrite the property with
      `./mvnw versions:set-property -Dproperty=project.build.outputTimestamp -DnewVersion="$RELEASE_DATE"`
- [ ] Verify the rewrite actually took effect and abort if it did not — `set-property`
      is documented as performing "no sanity checks", so a silent no-op must not pass
      unnoticed
- [ ] Place both before the local `-Pfull-build clean verify` so the verified bytes
      match what CI will publish
- [ ] Leave the next-`SNAPSHOT` bump untouched
- [ ] Comment why the value is date-granular and why it is not a build time

**Acceptance criteria:**
- [ ] `bash -n release.sh` passes
- [ ] The timestamp rewrite appears before the verification build in the script
- [ ] The next-snapshot section contains no timestamp handling
- [ ] A failed or ineffective rewrite exits non-zero

**Related behaviors:** Cutting a release rewrites the timestamp to the release date;
The release commit carries both the version and the timestamp; The timestamp is
written before the verification build; Bumping to the next snapshot leaves the
timestamp untouched; Two releases on the same day share a timestamp; A failed property
rewrite aborts the release

---

## Step 4: Document the reproducibility claim in the README

- [ ] Add a **Reproducible builds** section stating the narrow claim: same source plus
      same toolchain yields byte-identical artifacts
- [ ] Document the verification recipe: check out the tag, run
      `./mvnw -Pfull-build clean verify`, compare against Maven Central
- [ ] Warn that `-Dproject.build.outputTimestamp` or a child-POM override defeats
      external verification
- [ ] Document `Git-Commit-Time` as the deterministic answer to "when did this state
      come into being", and state that the timestamp property is not a build time
- [ ] Mention the property in the existing "Build conventions" list

**Acceptance criteria:**
- [ ] The README does not claim reproducibility across differing JDK patch versions
- [ ] The documented public build command matches what CI runs
- [ ] Links and table formatting render correctly

**Related behaviors:** The README states only the measured claim; Git-Commit-Time
remains available and distinct; Deferred work is discoverable

---

## Step 5: Acceptance run — measure reproducibility with a fixture child

- [ ] `./mvnw -N install` the modified parent
- [ ] Create a throwaway child project outside the repository, inheriting the parent,
      with one Java source file
- [ ] Build it twice with `-Pfull-build clean verify`, no flags, with time between runs
- [ ] Compare jar, sources jar, javadoc jar, `bom.json` and `bom.xml`
- [ ] Confirm the inherited value appears in the child's archive entries
- [ ] Confirm a CLI `-D` override and a child-POM property each win over the inherited
      value
- [ ] Confirm a build with no `.git` directory succeeds and is stamped identically
- [ ] Confirm `Git-Commit-Time` is present in the manifest and independent of the
      timestamp property
- [ ] Empirically determine whether `versions:set-property` fails on a missing
      property; if it does not, confirm the Step 3 guard catches it
- [ ] Record the measured results for the pull request description

**Acceptance criteria:**
- [ ] All five artifacts are byte-identical across the two runs
- [ ] Override precedence is confirmed as `CLI -D` > child POM > parent POM
- [ ] The no-`.git` build succeeds and produces the same stamp
- [ ] The throwaway project is removed afterwards and the repository is left clean

**Related behaviors:** A child project builds byte-identically twice; The parent's own
build is deterministic; A command-line override wins over the inherited value; A child
POM property wins over the inherited value; A build without a Git directory still
succeeds and is stamped; Git-Commit-Time remains available and distinct

---

## Behavior Coverage

All 23 scenarios from `behaviors.md`. "Layer" is Build/CI configuration throughout —
this project has no backend or frontend. "Verification" states honestly how each
scenario is confirmed.

| Scenario | Layer | Verification | Covered in Step |
|----------|-------|--------------|-----------------|
| A child project builds byte-identically twice | Build | Execution | 5 |
| The parent's own build is deterministic | Build | Execution | 5 |
| A child inherits the parent's value without declaring anything | Build | Execution | 1, 5 |
| A child pinning an older parent is unaffected | Build | Inspection — documents the accepted rollout gap; no change to make | 4 |
| A command-line override wins over the inherited value | Build | Execution | 5 |
| A child POM property wins over the inherited value | Build | Execution | 5 |
| A verifier reproduces a release with no insider knowledge | CI | Inspection — needs a real published release; mechanism verified via Steps 2 and 5 | 2, 5 |
| A build without a Git directory still succeeds and is stamped | Build | Execution | 5 |
| Locale and timezone do not affect the stamp | Build | Inspection — the property is an absolute UTC instant; cross-environment measurement is a TODO | 1 |
| Cutting a release rewrites the timestamp to the release date | Release | Inspection | 3 |
| The release commit carries both the version and the timestamp | Release | Inspection | 3 |
| The timestamp is written before the verification build | Release | Inspection | 3 |
| Bumping to the next snapshot leaves the timestamp untouched | Release | Inspection | 3 |
| Two releases on the same day share a timestamp | Release | Inspection — follows from date granularity | 3 |
| A failed property rewrite aborts the release | Release | Execution — the `set-property` no-op question is settled in Step 5 | 3, 5 |
| The release workflow builds without a timestamp flag | CI | Inspection | 2 |
| The snapshot workflow builds without a timestamp flag | CI | Inspection | 2 |
| CI and a third party run the same command | CI | Inspection | 2, 4 |
| A child on a SNAPSHOT parent inherits a moving value | Build | Inspection — accepted non-goal, documented | 4 |
| Git-Commit-Time remains available and distinct | Build | Execution | 5 |
| No build-time field is introduced | Build | Inspection | 1, 4 |
| The README states only the measured claim | Docs | Inspection | 4 |
| Deferred work is discoverable | Docs | Inspection | 4 |

No scenario is unassigned. Six are confirmed by real measurement in Step 5; the rest
are configuration or documentation facts confirmed by reading the result. Three
scenarios describe conditions that cannot be exercised without cutting a real release
or a differing machine, and say so.

---

## Out of scope for these steps

- `CLAUDE.md` does not exist in this repository. Creating one is unrelated to this
  spec and is not done here.
- No automated reproducibility check is added — deferred by decision, tracked in
  `docs/TODO.md`.
