# Automotive C++ CI/CD Pipeline: Stages & Quality Gates

A generic overview of a CI/CD pipeline for safety-oriented automotive C++ software, built around **Bazel**, **static analysis (QAC, Coverity)**, **unit testing (GoogleTest)** and **code coverage**.

> **Note:** This is a sanitized, tool-level description written for learning and interview preparation. It contains no proprietary code, hostnames, credentials or customer-specific details.

---

## Table of Contents

1. [Pipeline at a glance](#pipeline-at-a-glance)
2. [Workflows and triggers](#workflows-and-triggers)
3. [Stage 1: Pre-checks](#stage-1-pre-checks)
4. [Stage 2: Build](#stage-2-build)
5. [Stage 3: Quality analysis](#stage-3-quality-analysis)
6. [Stage 4 and 5: Validation and reporting](#stage-4-and-5-validation-and-reporting)
7. [Gating policy](#gating-policy)
8. [PR pipeline vs nightly pipeline](#pr-pipeline-vs-nightly-pipeline)
9. [Config-driven builds](#config-driven-builds)
10. [Baseline management](#baseline-management)
11. [Failure handling](#failure-handling)
12. [Secrets and licences](#secrets-and-licences)
13. [Skills demonstrated](#skills-demonstrated)

---

## Pipeline at a glance

```
 TRIGGER: push / pull request / nightly schedule
                      |
                      v
 +--------------------------------------------------+
 | STAGE 1: PRE-CHECKS (gating)                     |
 |  - Formatting and style checks                   |
 |  - PR size validation                            |
 |  - Naming and test-tag validation                |
 |  - Dependency-rule checks                        |
 +--------------------------------------------------+
                      | pass
          +-----------+-----------+
          v                       v
 +------------------+   +------------------+
 | STAGE 2A: BUILD  |   | STAGE 2B: BUILD  |
 | Linux GCC 12     |   | QNX 7.1          |
 | release / debug  |   | release / debug  |
 +------------------+   +------------------+
          |                       |
          +-----------+-----------+
                      v
 +--------------------------------------------------+
 | STAGE 3: QUALITY ANALYSIS                        |
 |  3A. QAC static analysis (full and delta)        |
 |  3B. Unit tests (GoogleTest)                     |
 |  3C. Code coverage (SquishCOCO)                  |
 |  3D. Coverity SAST                               |
 +--------------------------------------------------+
                      |
                      v
 +--------------------------------------------------+
 | STAGE 4: VALIDATION AND PACKAGING                |
 +--------------------------------------------------+
                      |
                      v
 +--------------------------------------------------+
 | STAGE 5: REPORTING                               |
 |  metrics, baseline comparison, artifact upload,  |
 |  build-status notification                       |
 +--------------------------------------------------+
```

---

## Workflows and triggers

| Workflow | Trigger | Purpose |
|---|---|---|
| **Build workflow** | Push to development branches, pull request to the development branch | Pre-checks plus build for each change |
| **Quality workflow** | Same as above | Static analysis (QAC) on every change |
| **Nightly workflow** | Daily scheduled run on the development branch | Full release and debug builds, unit tests, coverage |

**Skipped runs:** draft pull requests, selected temporary branches and commits marked as drafts.

---

## Stage 1: Pre-checks

Fast checks that run before any compilation. Blocking checks stop the pipeline early.

| Check | Tool | Purpose | Severity |
|---|---|---|---|
| Code formatting | clang-format | Enforce a consistent code format | Warning |
| C++ best practices | clang-tidy | Static analysis for common issues | Warning |
| Code style | Custom Python | Naming conventions and member prefixes | Warning |
| PR size | Custom Python | Limit file size and total change size | **Blocking** |
| Naming consistency | Custom Python | Interface and component naming rules | **Blocking** |
| Test tags | Custom Python | Valid unit and component test tags | **Blocking** |
| Dependency rules | Custom Python | Prevent illegal imports between code areas | **Blocking** |
| Repository object size | Custom Python | Git object size limits by file type | **Blocking** |

**Why it matters:** cheap checks run first, so a developer gets feedback in minutes and CI capacity is not wasted on changes that were never going to pass.

---

## Stage 2: Build

Builds run **in parallel** for each target platform after the pre-checks pass.

```bash
# Generic form of the build command
bazel build --config=<variant>_<platform>_<buildtype> //<package>:<target>
```

| Platform | Toolchain | Build types |
|---|---|---|
| Linux | GCC 12 | Release, Debug |
| QNX (real-time OS) | QNX 7.1 | Release, Debug |

**Outputs:** compiled binaries and libraries, a build-event log for compiler warnings, and cached build outputs for faster incremental builds.

**Why Bazel:** reproducible, incremental builds with explicit dependency tracking, so the same inputs give the same outputs on a developer machine and on a CI agent.

---

## Stage 3: Quality analysis

### 3A. Static code analysis: Helix QAC

| Mode | When | What it analyses |
|---|---|---|
| **Full analysis** | Scheduled / full runs | The whole codebase against the configured rule set |
| **Delta analysis** | Pull requests | Only changed files compared with the target branch, for fast feedback |

- Rule set: MISRA C++ together with project-specific safety rules.
- Results are compared against a stored **baseline**, so *new* violations are detected as regressions without being buried by legacy findings.
- Output formats: CSV, XML, SARIF, HTML and JSON.

### 3B. Unit tests: GoogleTest

- Test types: `TEST`, `TEST_F` and `TYPED_TEST`, built and run through Bazel.
- Tests follow a GIVEN / WHEN / THEN structure for traceability.
- Output: JUnit-style XML for CI, plus console summaries.

```cpp
/// GIVEN: <preconditions>
/// WHEN:  <test steps>
/// THEN:  <expected results>
TEST(MySuite, myTestName)
{
    // test implementation
}
```

### 3C. Code coverage: SquishCOCO

- Instrumented build, run with the test suite, then coverage data is merged and analysed.
- Metrics: line, branch and function coverage, plus MC/DC where the safety level requires it.
- Output: HTML dashboards, CSV, XML and LCOV; trends are tracked against a reference baseline.

### 3D. Static application security testing: Coverity

- Build capture followed by deep analysis (`cov-build`, `cov-analyze`).
- Finds path-dependent defects, memory errors, concurrency issues and security problems.
- High-severity findings gate the merge.

---

## Stage 4 and 5: Validation and reporting

**Validation:** artifact generation and packaging, additional validation builds and component-test validation.

**Reporting:**
- Aggregated quality metrics
- Baseline comparison for regression detection
- Artifact upload to the artifact repository
- Build-status notification to the team

---

## Gating policy

| Stage | Tool | Gate |
|---|---|---|
| Pre-check | Formatting / lint checks | Warning |
| Pre-check | PR size, naming, tags, dependency rules | **Blocking** |
| Build | Bazel + GCC / QNX | **Blocking** |
| Static analysis | QAC | Regression against baseline |
| Unit tests | GoogleTest | **Blocking** |
| Coverage | SquishCOCO | Trend / minimum thresholds |
| SAST | Coverity | **High severity blocks** |
| Reporting | Custom scripts | Informational |

---

## PR pipeline vs nightly pipeline

```
PULL REQUEST (fast feedback)
  Pre-checks --+--> Build changed targets
               +--> QAC delta analysis        (run in parallel)

NIGHTLY (full confidence)
  Full builds (GCC + QNX, release + debug)
        |
        +--> Unit tests ------+
        |                     +--> QAC full, Coverity, reporting
        +--> Code coverage ---+
```

| | Pull request | Nightly |
|---|---|---|
| Goal | Fast feedback on a change | Full verification of the branch |
| Scope | Changed targets and files | Everything, all variants |
| Analysis | QAC **delta** | QAC full, coverage, Coverity |

---

## Config-driven builds

Build, analysis and coverage commands are defined in one central configuration file, so pipelines stay generic and new variants are added by configuration, not by editing workflows.

```json
{
  "sw_build_targets": {
    "linux_gcc12_release": {
      "variants": { "<variant>": ["bazel build --config=<variant>_linux_gcc12_release //<target>"] },
      "unit_test": {},
      "component_test": {}
    }
  },
  "sca_build_targets": {},
  "sca_delta_build_targets": {},
  "coco_build_targets": {},
  "cov_build_targets": {}
}
```

---

## Baseline management

Quality baselines are tracked in version control so every change is compared with an approved reference.

1. A change is committed and CI generates new quality metrics.
2. If the results are acceptable, the baseline is updated through a normal reviewed commit.
3. Later pull requests are compared against the new baseline.

Benefit: teams are not blocked by historical findings, but cannot make quality worse.

---

## Failure handling

| Failure | Result | Typical fix |
|---|---|---|
| Pre-check | PR rejected within minutes | Apply the formatter, reduce PR size, correct names |
| Build | Merge blocked | Fix compile errors and rebuild locally first |
| Static analysis | Warning or blocking by severity | Fix the root cause, or suppress with a documented justification |
| Unit test | Merge blocked | Debug the test or the code under test |

---

## Secrets and licences

- Credentials for registries, source control and artifact repositories are stored as **CI secrets**, never in code or in the repository.
- Commercial tools (QNX, SquishCOCO, Coverity) use **floating licences** served from a licence server that the build runners can reach.
- Access follows the principle of least privilege.

---

## Skills demonstrated

`CI/CD` · `Bazel` · `Docker` · `Linux` · `QNX` · `C++` · `Static analysis (QAC, Coverity)` · `GoogleTest` · `Code coverage` · `Quality gates` · `Baseline / regression management` · `Pipeline as code` · `Python and Bash automation`
