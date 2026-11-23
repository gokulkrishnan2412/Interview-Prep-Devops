# Interview Question Bank: Azure DevOps CI/CD Role (Conan + CMake)

Prepared around an automotive C++ CI/CD pipeline (Bazel, QAC, GoogleTest, Coverage, Coverity, Docker, Conan).

---

## 0. Read this first: your experience vs the role

| The role asks for | Your real experience | How to bridge honestly |
|---|---|---|
| Azure DevOps Pipelines | Jenkins / GitHub Actions style pipelines, AWS-hosted infrastructure | "The concepts map directly. I have built multi-stage pipelines with gating, parallel builds and quality stages. In Azure DevOps the same ideas are stages, jobs, steps and templates." |
| CMake | Bazel as the build system | "I have worked with Bazel for C++ builds. I understand CMake concepts: targets, toolchains, presets and how Conan integrates with it." Only claim CMake hands-on work if you have done it. |
| Conan | Conan dependency management | This is your strong point. Know it well (Section 3). |

**Rules for tomorrow:**
- Say "I" only for what you built or maintained. Say "we" for what the team did.
- If you have not used something, say: "I have not used it directly, but I have used X, which is similar, and this is how I would approach it."
- Do not claim Azure DevOps or CMake project work you have not done. A follow-up question will expose it.

---

## 1. Questions about your project

**Q1. Walk me through your CI/CD pipeline.**
Pre-checks (format, naming, PR size, test tags) block early. Then parallel builds for Linux GCC and QNX with Bazel. Then quality analysis: QAC (delta on PRs, full nightly), GoogleTest unit tests, coverage with SquishCOCO, Coverity. Then validation, reporting, baseline comparison and artifact upload.

**Q2. Why do pre-checks run first?**
They are the cheapest and fastest. Failing early saves build time and gives developers feedback in minutes.

**Q3. How do you speed up pull request builds?**
Build only changed targets, run QAC in delta mode on changed files, cache dependencies and build outputs, run independent jobs in parallel, and keep full builds and coverage for nightly runs.

**Q4. What is the difference between QAC delta and full analysis?**
Delta analyses only changed files against the target branch for fast PR feedback. Full analyses the whole codebase and is run on a schedule.

**Q5. What is a baseline and why do you use one?**
A stored reference of known findings. New violations are measured against it, so legacy issues do not block new work, but quality cannot get worse.

**Q6. QAC vs Coverity?**
QAC checks coding-standard compliance (for example MISRA). Coverity does deep defect analysis: memory errors, concurrency problems and security issues.

**Q7. What was the hardest problem you solved in this pipeline?**
Use your real story. A solid option from your resume: intermittent build failures caused by differences in dependencies, toolchains and environments. Fixed by standardising on Docker (environment), Conan (pinned dependencies) and Bazel (repeatable builds).

**Q8. What would you improve in the pipeline?**
Examples: better caching, a dependency lockfile for stricter reproducibility, clearer dashboards for quality trends, faster feedback on quality stages, pipeline templates to avoid duplication.

---

## 2. Azure DevOps Pipelines

**Q9. What are the main building blocks of an Azure DevOps YAML pipeline?**
Trigger -> **stages** -> **jobs** -> **steps** (tasks or scripts). Stages run in sequence by default, jobs inside a stage can run in parallel, and steps run in order within a job.

**Q10. Microsoft-hosted vs self-hosted agents?**
Microsoft-hosted: managed, clean VM per job, easy, but limited customisation and no access to private networks or licence servers by default. Self-hosted: you control tools, hardware, network access (needed for QNX or commercial licences, large caches), but you maintain and patch them.

**Q11. How do you pass variables and secrets securely?**
Variable groups, optionally linked to **Azure Key Vault**. Mark secrets as secret so they are masked in logs. Never hard-code credentials in YAML. Use **service connections** for access to Azure, registries and artifact repositories.

**Q12. What are pipeline templates and why use them?**
Reusable YAML (steps, jobs or stages) shared across repositories. They reduce duplication, standardise builds and let you change a process in one place.

**Q13. How do you trigger pipelines for PRs?**
For Azure Repos, use **branch policies** with build validation. For GitHub or Bitbucket repos, use the `pr:` trigger. Use `trigger:` for pushes and `schedules:` (cron) for nightly runs.

**Q14. How do you control deployments and approvals?**
**Environments** with approvals and checks, and deployment jobs. Gates can require manual approval, a work-item check or an Azure Monitor check before a stage proceeds.

**Q15. How do you cache dependencies in Azure Pipelines?**
The `Cache@2` task, keyed on a hash of the dependency definition (for example `conanfile.py` or the lockfile), caching the Conan cache directory.

**Q16. How do you publish build outputs?**
`PublishPipelineArtifact` for pipeline artifacts, **Azure Artifacts** (universal packages) or an external repository such as Artifactory. `PublishTestResults` publishes test results so they show in the Tests tab.

**Q17. How do you run stages in parallel and control dependencies?**
`dependsOn` in stages and jobs. Jobs without dependencies run in parallel. Use `strategy: matrix` to build several configurations (for example Debug/Release, GCC/Clang).

**Q18. Classic pipelines vs YAML pipelines?**
YAML is code in the repository: versioned, reviewable and reusable. Classic uses the UI. YAML is the recommended approach.

**Q19. What are conditions and how do you use them?**
`condition:` on stages, jobs or steps, for example run reporting even if tests fail (`succeededOrFailed()`), or only on the main branch.

**Q20. How do you monitor and troubleshoot a failing Azure pipeline?**
Check the failed step's logs, enable `system.debug`, rerun the job, compare with the last successful run, check agent capabilities, and reproduce the same commands locally or in the same container.

### Sample Azure DevOps pipeline (know how to read and write this)

```yaml
trigger:
  branches:
    include: [ develop ]

pr:
  branches:
    include: [ develop ]

pool:
  name: SelfHostedLinux      # or: vmImage: ubuntu-latest

variables:
  - group: build-secrets      # secrets via variable group / Key Vault

stages:
  - stage: PreChecks
    jobs:
      - job: Lint
        steps:
          - script: ./tools/check_format.sh
            displayName: Formatting and naming checks

  - stage: Build
    dependsOn: PreChecks
    jobs:
      - job: BuildRelease
        steps:
          - task: Cache@2
            inputs:
              key: 'conan | "$(Agent.OS)" | conanfile.py'
              path: $(Pipeline.Workspace)/.conan2
          - script: |
              pip install conan
              conan profile detect --force
              conan install . --build=missing
              cmake --preset conan-release
              cmake --build --preset conan-release
            displayName: Conan install + CMake build
          - script: ctest --preset conan-release --output-on-failure
            displayName: Run unit tests
          - task: PublishTestResults@2
            condition: succeededOrFailed()
            inputs:
              testResultsFormat: JUnit
              testResultsFiles: '**/test-results/*.xml'
          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: build/Release
              artifact: release-binaries
```

---

## 3. Conan (strong area, prepare well)

> The commands below use **Conan 2** syntax. If your project used Conan 1, mention the version and key differences.

**Q21. What is Conan and why use it for C++?**
A C/C++ package manager. It gives versioned, reproducible dependencies, binary packages (so you do not rebuild everything), and consistent builds across developers and CI.

**Q22. What does `conanfile.py` contain?**
Package metadata, `settings` (OS, compiler, build type, architecture), `options`, `requires` (dependencies), `build_requirements` (tools), and methods like `generate()`, `build()` and `package()`. For consumers there can be a `conanfile.txt` or `conanfile.py`.

**Q23. What are Conan profiles?**
Files describing the build and host environment: OS, architecture, compiler and version, C++ standard, build type. Used for cross compilation with two profiles: `-pr:b` (build machine) and `-pr:h` (host/target).

**Q24. How do you cross-compile (for example for QNX or embedded) with Conan?**
A **host profile** for the target (compiler, OS, architecture) and a **build profile** for the machine running the build, plus a toolchain file or compiler settings. Tools needed on the build machine go in `tool_requires`.

**Q25. What are the key commands?**
`conan install` (resolve and fetch/build dependencies), `conan create` (build and test a package in the cache), `conan upload` (push to a remote), `conan remote add/login`, `conan list`, `conan lock create` (lockfile), `conan search`.

**Q26. How do you make dependencies reproducible?**
Pin versions, use **lockfiles**, fixed profiles, a locked Docker image for the toolchain, and an internal remote as the only source of packages.

**Q27. What is the package ID and why does it matter?**
A hash computed from the settings and options that affect the binary. If any of these differ, Conan needs a different binary. This is why a compiler or build-type mismatch triggers a rebuild or "package not found".

**Q28. What does `--build=missing` do?**
Uses available binaries and builds from source only those packages without a matching binary.

**Q29. How do you integrate Conan with CMake?**
Use the generators **`CMakeDeps`** (creates `find_package` files) and **`CMakeToolchain`** (creates the toolchain file and presets), with `cmake_layout`. Then CMake uses `find_package(<lib>)` and the toolchain file.

**Q30. How do you handle private packages and a remote repository?**
Host an internal remote (for example Artifactory). Add it with `conan remote add`, authenticate with `conan remote login` using CI secrets, and upload built packages from CI only after the pipeline passes.

**Q31. Packages fail to resolve in CI. How do you debug?**
Check the remote and credentials, the profile (compiler, build type), version ranges that changed, the package ID mismatch, the Conan cache state, and compare the Conan version used locally and on CI. Run `conan install` with higher verbosity (`-vtrace` in Conan 2).

**Q32. How do you version and release your own Conan packages from CI?**
Build with `conan create` on the CI agent, run package tests (`test_package`), upload to the remote on the main/release branch only, and use a clear versioning scheme (semantic versions, with channels or branches handled by CI).

**Q33. Conan cache: where does it live and how do you manage it on CI?**
In the user's Conan home (`~/.conan2`). On CI, cache it between runs keyed on the dependency definition, and clean it periodically to avoid stale or corrupted packages.

---

## 4. CMake

> Only answer from experience if you have used it. Otherwise lead with: "My builds were Bazel-based, but I understand CMake as follows."

**Q34. What is a "target" in modern CMake?**
A buildable unit (executable or library) created by `add_executable` or `add_library`. Properties such as include paths, definitions and link dependencies are attached to targets with `target_include_directories`, `target_compile_definitions` and `target_link_libraries`, using `PUBLIC`, `PRIVATE` or `INTERFACE` to control propagation.

**Q35. PUBLIC vs PRIVATE vs INTERFACE?**
PRIVATE: only this target uses it. PUBLIC: this target and anything linking to it. INTERFACE: only consumers, not this target.

**Q36. What is an out-of-source build?**
Building in a separate directory (for example `build/`) so the source tree stays clean: `cmake -S . -B build`.

**Q37. What is a toolchain file and when do you use it?**
A file that sets the compiler, sysroot and target system for **cross compilation** (for example QNX or ARM). Passed with `-DCMAKE_TOOLCHAIN_FILE=...`. Conan generates one with `CMakeToolchain`.

**Q38. What are CMake presets?**
`CMakePresets.json` defines configure, build and test presets (generator, build type, toolchain file, cache variables), so CI and developers use identical commands.

**Q39. How do you find and use dependencies?**
`find_package(<Name> REQUIRED)` finds a library, and `target_link_libraries(app PRIVATE Name::Name)` links it. With Conan, `CMakeDeps` generates the files that make `find_package` work.

**Q40. How do you run tests with CMake?**
`enable_testing()` and `add_test()` (or `gtest_discover_tests()` with GoogleTest), then `ctest --output-on-failure`. Output can be converted to JUnit XML for the CI test report.

**Q41. Debug vs Release, single-config vs multi-config generators?**
Single-config (Makefiles, Ninja): set `CMAKE_BUILD_TYPE` at configure time. Multi-config (Visual Studio, Ninja Multi-Config): choose with `--config` at build time.

**Q42. How do you speed up CMake builds in CI?**
Ninja generator, `ccache` or `sccache`, parallel builds (`--parallel`), caching the Conan cache and build directory, and building only changed targets.

**Q43. How do you generate coverage with CMake builds?**
Compile with coverage flags (for example `--coverage`), run the tests, then collect with `gcov/lcov` or `gcovr` and publish an HTML or Cobertura report in the pipeline.

**Q44. Bazel vs CMake: how would you compare them?**
Bazel: hermetic, strongly cached, scales to large monorepos, strict dependency declarations, steeper learning curve. CMake: de-facto standard for C++, broad tool and IDE support, flexible, but reproducibility depends more on the environment, which is why Conan and Docker are used with it.

---

## 5. Quality, testing and security

**Q45. How do you integrate static analysis into the pipeline?**
Run it as a stage, publish reports (SARIF/HTML), compare with a baseline, and set gating rules: block on new high-severity findings, warn on trends.

**Q46. How do you handle code coverage and what do you gate on?**
Instrumented build, run tests, collect and publish reports, track trends, and enforce thresholds required by the project's safety level.

**Q47. What is MISRA / why does it matter in automotive?**
A coding standard for safe C/C++ use in safety-related software. Tools like QAC check compliance, and violations need to be fixed or formally justified.

**Q48. How do you manage secrets in pipelines?**
Key Vault or the platform secret store, masked variables, least-privilege service accounts, rotating tokens, never in source code or logs.

**Q49. How do you secure the supply chain for dependencies?**
Use only an internal, controlled package remote, pin and lock versions, scan dependencies for known vulnerabilities, and review any new dependency.

---

## 6. Docker, Linux and scripting

**Q50. Why use Docker for builds?**
Identical toolchain and dependencies on every developer machine and CI agent. It removes "works on my machine" problems and makes toolchain upgrades and rollbacks easy.

**Q51. How do you keep Docker build images small and fast?**
Multi-stage builds, minimal base image, ordering layers so rarely changed layers are cached, `.dockerignore`, and one tool-versioned image per toolchain.

**Q52. What do you automate with Python and Bash?**
Pre-check scripts (naming, size limits), test execution and validation wrappers, report aggregation and parsing build logs.

**Q53. Linux commands for debugging a failing build agent?**
`df -h` (disk), `free -m` and `top` (memory/CPU), `journalctl` and log files, `ps`, `lsof`, `env` (environment), `ldd` (shared libraries), `strace` for system calls.

---

## 7. Troubleshooting scenarios

**Q54. A build passes locally but fails in CI. What do you do?**
Compare environments: compiler and tool versions, Conan profile, dependency versions, environment variables and OS. Reproduce inside the same Docker image. Check for hidden dependence on local files.

**Q55. A build that worked yesterday fails today with no code changes.**
Check what changed outside the code: dependency version ranges, base image or toolchain update, agent change, expired credentials or licences, remote repository outage, cache corruption.

**Q56. Pipeline suddenly became much slower.**
Compare stage timings with previous runs, check cache hit rate, agent resources, queue time, new large dependencies or a new tool stage, and network access to package remotes.

**Q57. Flaky tests in CI.**
Rerun to confirm flakiness, isolate and quarantine the test with a ticket, find the cause (timing, shared state, ordering, resources), fix it, then restore it to the gating suite.

**Q58. A dependency version introduced a bug in production-bound builds.**
Roll back to the last known good version using the lockfile, notify affected teams, then analyse and fix forward, and add a check that would have caught it.

**Q59. License server unreachable for a commercial tool.**
Verify network and VPN access from the agent, check the licence server status and the number of licences in use, retry with backoff, notify the tool owner, and keep a documented fallback.

**Q60. Two teams need different compiler versions.**
Separate Docker images and Conan profiles per toolchain, selected with a pipeline variable or matrix, so builds do not interfere.

---

## 8. Behavioural questions (use STAR: Situation, Task, Action, Result)

- Tell me about a time you fixed a major pipeline outage.
- Tell me about a time you disagreed with a developer or lead. How did you resolve it?
- Describe a time you automated a manual task. What was the impact?
- How do you prioritise when several teams need help at once?
- How do you document and share knowledge with the team?
- Describe a mistake you made in a pipeline or release, and what you learned.
- Why are you moving to this role / company?

---

## 9. Questions you can ask the interviewer

- Which build system and package management approach does the team use, and are Conan and CMake used together?
- What does the current Azure DevOps setup look like: self-hosted or Microsoft-hosted agents, templates, environments?
- How is the team structured, and who owns pipeline reliability?
- What are the biggest pain points in the current CI/CD process?
- What does success look like in the first 3 to 6 months?

---

## 10. Last-minute revision checklist

- [ ] I can explain my pipeline in 60 seconds.
- [ ] I know the difference between stage, job and step in Azure DevOps.
- [ ] I can explain Conan profiles, lockfiles and package ID.
- [ ] I can explain how Conan connects to CMake (CMakeDeps, CMakeToolchain, presets).
- [ ] I can explain Bazel vs CMake honestly.
- [ ] I know what I have done hands-on and what I am still learning.
