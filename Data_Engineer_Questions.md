============================================================
DATA ENGINEER (C++ / PYTHON) - ISSP AUTOMOTIVE PROJECT
Role, Responsibilities and Interview Preparation
============================================================

Context (from the repository layout)
- Automotive embedded software (ISSP): driver monitoring (DMS), FCT
  signals, vehicle generic interface (VGI), runnables and AOS modules.
- C++ under issp/ and prototype/ follows ASIL-B rules (naming, Doxygen,
  GIVEN/WHEN/THEN unit tests, QAC/PRQA deviations).
- Targets: Linux (GCC 12) and QNX (7.1 / 8.0), built with Bazel and Conan.
- Python tooling under tools/: ims_measurement_library, rb_sxpf_replay,
  model_config_generator, smoketest, launchers, profiles, sharcc, eet, dol,
  prmfy, swqa.
- Compile_commands, compiler-warning and QAC baselines are tracked.


PART 1 - ROLES AND RESPONSIBILITIES

1. Data pipeline development (Python)
   - Build/maintain tools that ingest, transform, validate and export
     vehicle and test data (measurements, signal traces, configuration).
   - Implement replay tooling that feeds recorded signals back into the
     system or test benches (e.g., rb_sxpf_replay).
   - Parse binary and text formats and convert to analysis formats
     (CSV, Parquet, JSON, HDF5, SQL).
   - Keep scripts reproducible: pinned dependencies, deterministic output,
     clear CLI arguments and exit codes.

2. Signal and interface data handling (C++)
   - Work on components that map input messages (FCT / DMS signal lists)
     to output structures: container traversal, validity checks, state
     aggregation.
   - Handle empty/missing signal lists safely; propagate degradation and
     validity states correctly.
   - Remove or guard dead code and unused state; keep warnings at zero in
     touched modules.

3. Data quality and validation
   - Implement checks: range, timestamp monotonicity, gaps/dropouts,
     validity flags, unit consistency.
   - Create test fixtures and reference datasets for regression testing.
   - Report anomalies with evidence (plots, counts, samples).

4. Test engineering
   - Python tests (pytest) for tooling; C++ unit/component tests
     (GoogleTest, TYPED_TEST, EXPECT_DEBUG_DEATH) per repo conventions.
   - Run smoke tests and targeted Bazel test targets before merging.

5. Build, CI and tooling
   - Maintain Bazel targets (BUILD.bazel, MODULE.bazel), Conan deps and
     build scripts (build_all.sh, test_builds.sh, verify_builds.sh).
   - Keep compile_commands, compiler-warning and QAC baselines current.
   - Diagnose Linux GCC vs. QNX toolchain issues.

6. Standards, documentation, traceability
   - Follow ASIL-B / MISRA-style rules: no undefined behaviour, bounded
     resources, explicit types, const correctness, controlled dynamic
     memory, justified PRQA deviations.
   - Update docs/README affected by your changes.

7. Collaboration
   - Work with functional safety, architects, HIL/test and integration teams.
   - Take part in code reviews, design discussions and planning.
   - Explain data findings to non-software stakeholders.


PART 2 - INTERVIEW QUESTIONS
A. C++ (General)
1. Explain RAII. How is it used for resource management in embedded code?
RAII ties a resource's lifetime to an object's lifetime: the constructor acquires it and the destructor releases it. This makes cleanup automatic on every exit path, including exceptions and early returns. In embedded code it covers mutex locks (std::lock_guard), file or driver handles, and buffers.

2. Stack vs. heap allocation. Why avoid dynamic allocation in safety code?
The stack is fast and deterministic, with lifetime tied to scope. The heap is slower, can fragment, and can fail at runtime. Safety standards such as ASIL-B favour static or bounded memory so worst-case behaviour and memory use can be verified before running.

3. lvalue vs. rvalue, move semantics, std::move.
An lvalue has a name and persistent storage. An rvalue is temporary. Move semantics let an object take over another's resources instead of copying them. std::move is only a cast to an rvalue reference; it moves nothing by itself.

4. unique_ptr vs. shared_ptr?
unique_ptr has single ownership, no overhead, and is the default choice. shared_ptr shares ownership through reference counting, with extra cost and the risk of cycles. In embedded or safety code, prefer unique_ptr or plain value types, and avoid dynamic allocation where the design allows.

5. What is undefined behaviour? Give examples.
Undefined behaviour means the standard places no requirements on the outcome, so the program may crash, produce wrong values, or appear to work. Examples are out-of-bounds access, use-after-free, dangling references, signed integer overflow, and reading uninitialised variables.

6. const vs. constexpr.
const means the value cannot change after initialisation, and it may be computed at runtime. constexpr means the value must be computable at compile time, which allows its use in array sizes and static asserts. Use constexpr for compile-time constants.

7. Virtual functions, vtables, and the virtual destructor.
A virtual call goes through a vtable pointer, which costs an indirection and prevents some inlining. A base class needs a virtual destructor so that deleting through a base pointer calls the derived destructor. Without it, the behaviour is undefined.

8. Function vs. class templates; what is TYPED_TEST used for?
A function template generates overloads for different types, and a class template generates types. TYPED_TEST (GoogleTest) runs one test body for several types, which is useful for testing the same interface across implementations.

9. std::vector vs. std::array vs. C array for a fixed-size signal list.
std::array is the best fit for a fixed size: it knows its size, supports range-for and bounds-checked .at(), and has no heap allocation. std::vector is for dynamic size and allocates. A C array decays to a pointer and loses its size.

10. Iterator invalidation rules for std::vector.
Growing the vector (push_back beyond capacity) reallocates and invalidates all iterators, pointers and references. Insert and erase also invalidate iterators at and after the affected position. Reserve capacity up front or avoid holding iterators across modifications.

11. Avoiding implicit conversions and narrowing.
Use brace initialisation, which rejects narrowing. Use static_cast explicitly, avoid mixing signed and unsigned in comparisons, and enable -Wconversion and -Wsign-compare. Scoped enum class prevents implicit conversion to int.

12. static_cast vs. reinterpret_cast.
static_cast converts between related types with compile-time checks. reinterpret_cast reinterprets the bit pattern and is almost always a red flag. It is acceptable only for specific low-level cases such as hardware registers, and should be documented.

13. One Definition Rule; header structure and forward declarations.
The ODR requires each entity to have exactly one definition across the program. Put declarations in headers and definitions in source files, use include guards or #pragma once, and forward-declare types where the full definition is not needed. This reduces coupling and compile time.

B. C++ (Concurrency, Performance, Tooling)
14. Data races and how to prevent them.
A data race is concurrent access to the same memory where at least one access is a write and there is no synchronisation. Prevent it with mutexes, atomics, message passing, or by making data immutable or thread-confined.

15. std::atomic vs. mutex.
std::atomic gives lock-free operations on a single value and is cheap for simple counters and flags. A mutex protects a larger critical section with multiple variables. Use atomics for single values and mutexes for invariants spanning several fields.

16. How would you profile a slow data-processing function?
Measure first with a representative input, using perf, gprof, or the QNX profiler. Check for allocations in loops, copies of large containers, cache-unfriendly access, and repeated lookups. Fix the hotspot, then measure again to confirm the gain.

17. Which warnings do you enable, and how do you handle them?
Use -Wall -Wextra -Werror as a baseline, plus -Wconversion, -Wshadow and -Wunused where the project allows. Fix the root cause rather than suppressing the warning, and keep the warning baseline clean.

18. A warning you cannot remove (variable used only in commented-out code). Removal or [[maybe_unused]]?
Prefer removal, because dead state misleads readers. Use [[maybe_unused]] only when the variable is intentionally kept for a planned re-enable and that intent is documented. Earlier in this session I recommended removal for hasFailureDegradationState.

19. What is static analysis (QAC / PRQA), and why do deviations need justification?
Static analysis checks code against rule sets such as MISRA and project coding standards without running it. Some rules must be broken in justified cases, and each deviation is recorded with a reason so auditors can see the risk was assessed. In this repo, deviations appear as PRQA S ... comments.

C. Python (General)
20. list vs. tuple vs. set vs. dict for signal data?
Use a list for ordered, mutable sequences such as samples in time order. Use a tuple for fixed records. Use a set for membership and unique values, and a dict for lookups by signal name or ID.

21. Generators and why they help with large recordings.
A generator yields items lazily, so only one item is in memory at a time. For large recordings this keeps memory flat and lets processing start before the file is fully read.

22. List comprehension vs. generator expression: memory implications?
A list comprehension builds the whole list in memory. A generator expression produces values on demand. Use the generator form when you only iterate once, such as passing it to sum().

23. Reference counting and the garbage collector.
CPython frees an object when its reference count reaches zero. The cyclic garbage collector handles reference cycles. You can avoid problems by breaking cycles with weakref and by releasing large objects promptly.

24. The GIL; parallelising CPU-bound work.
The Global Interpreter Lock lets only one thread run Python bytecode at a time, so threads do not speed up CPU-bound code. Use multiprocessing or concurrent.futures.ProcessPoolExecutor for CPU-bound work, and threads for I/O-bound work.

25. is vs. ==.
== compares values, and is compares identity. Use is only for singletons such as None, and == for values.

26. Exception handling in pipelines; custom exceptions.
Catch specific exceptions, not bare except. Define custom exception classes for domain errors such as invalid frames or missing signals. Log the context, and either recover or fail with a clear exit code.

27. Decorators and context managers.
A decorator wraps a function to add behaviour such as timing or logging. A context manager, used with with, guarantees setup and teardown, such as closing files or releasing locks. It is implemented with __enter__/__exit__ or contextlib.

28. Type hints; mypy / Pylance.
Type hints document the expected types and let static checkers catch errors before runtime. Pylance in VS Code and mypy in CI are common choices. Hints are not enforced at runtime.

29. Package structure (init.py, entry points, argparse CLI).
A package is a directory with modules and an __init__.py. Expose the command-line entry point through if __name__ == "__main__": or a console-script entry in packaging. Use argparse for typed, documented options and return meaningful exit codes.

D. Python Data Engineering
30. Reading a file too large for memory.
Read in chunks or stream line by line, use pandas.read_csv(chunksize=...), or use mmap for random access. For binary data, read fixed-size records with struct, or use numpy.memmap.

31. pandas vs. numpy vs. polars.
NumPy provides fast arrays for numeric work. pandas adds labelled tabular data and time-series tools. Polars is a faster, multi-threaded DataFrame library with lazy evaluation, suited to larger tables.

32. Time-series: resampling, interpolation, aligning signals with different rates.
Resample to a common grid, using forward-fill for status signals and interpolation for continuous values. Check timestamp units and time zones, and use merge_asof for alignment on the nearest earlier sample.

33. Detecting missing data, duplicates and outliers.
Count NaNs and gaps against the expected sample period. Drop or flag duplicate timestamps. Use range limits from the signal specification, and z-score or IQR for statistical outliers, with the decision documented.

34. ETL vs. ELT.
ETL transforms data before loading it into the target store. ELT loads raw data first and transforms it inside the store. For test-data pipelines, ETL-style validation before storage keeps bad data out of the reference set.

35. Designing idempotent pipelines.
Running the same input twice must produce the same output. Achieve this with deterministic transforms, overwriting outputs by key, and no hidden state, so reruns and partial failures are safe.

36. Versioning datasets and transformation logic.
Version the code with git and pin dependencies. Store dataset versions with checksums and metadata that record the code version and parameters used to produce them.

37. Testing transformations.
Use pytest with small fixtures and @pytest.mark.parametrize for edge cases. Use golden files for expected output, and compare floats with a tolerance such as pytest.approx.

38. File formats for measurements: CSV, Parquet, HDF5, MDF/ASAM.
CSV is simple and human-readable but large and untyped. Parquet is columnar, compressed and typed, good for analytics. HDF5 suits hierarchical numeric data. MDF (ASAM) is the standard automotive measurement format and is common in vehicle tooling.

39. Parsing a binary protocol.
Describe the layout exactly, then use struct.unpack with an explicit byte-order prefix such as < for little-endian. Check lengths before unpacking, and handle padding and alignment explicitly.

40. Calling C++ from Python: pybind11, ctypes, CLI wrappers.
pybind11 gives the cleanest typed bindings but requires a build step. ctypes needs no compilation but has no type safety and needs a C interface. A CLI wrapper is the most decoupled and simplest to test, but it adds process overhead and data-passing cost.

E. Automotive / Domain
41. What is ASIL and how does it change the way you write code?
ASIL (A to D) is the Automotive Safety Integrity Level from ISO 26262, based on hazard severity, exposure and controllability. Higher levels require stricter coding rules, deterministic behaviour, bounded resources, full test coverage and documented deviations.

42. ISO 26262 and examples of safety mechanisms.
ISO 26262 is the functional-safety standard for road vehicles. Examples of safety mechanisms are plausibility checks, watchdogs, redundant computation, signal validity checks, and safe-state transitions.

43. What is a signal validity or degradation state, and why must it be propagated?
A validity or degradation state tells consumers whether a value can be trusted. If it is dropped silently, a consumer may act on bad data. In this codebase, computeDrivingStackStatus maps degradation states to the DMS state, and the commented-out failure path shows how a missing propagation can hide errors.

44. CAN / Ethernet at a high level; why signal timing matters.
CAN is a low-bandwidth, multi-master bus with message IDs and arbitration. Automotive Ethernet provides higher bandwidth for camera and sensor data. Timing matters because control and safety functions rely on fresh data, and late or missing messages must be detected.

45. What is a driver monitoring system (DMS), and what signals might it produce?
A DMS uses a camera and algorithms to assess driver state. Its outputs include drowsiness level, distraction warnings, microsleep and sleep warnings, unresponsive-driver warnings, and driver presence, each with a validity and degradation state.

46. How do you test behaviour when an input is missing or stale?
Write component tests that feed empty lists, old timestamps, and invalid states through the public API. Assert the expected degradation or fallback state, and confirm that no stale value is treated as valid.

47. HIL testing, and how replay data supports it.
Hardware-in-the-loop testing runs the software against real or simulated hardware and signal inputs. Replay data reproduces recorded real-world signals on a test bench, so behaviour can be checked repeatably.

F. Build, Tools and Process
48. Bazel vs. CMake.
Bazel is hermetic and incremental, with explicit dependencies and remote caching, which suits large multi-target builds. CMake generates build files for native tools and is widely used but less strict about dependencies.

49. Why Conan for C++ dependencies?
Conan manages versioned, reproducible third-party packages across platforms and compilers. It makes builds on Linux and QNX use the same dependency versions.

50. A build works on Linux but fails on QNX. How do you debug it?
Compare toolchains, compiler flags and target definitions. Check for POSIX or Linux-only APIs, endianness and alignment assumptions, and differing standard-library versions. Reproduce the error with the QNX toolchain's own compile command from compile_commands.json.

51. What belongs in a CI pipeline for this kind of project?
Build all targets on each supported platform, run unit and component tests, enforce zero new compiler warnings, run static analysis, compare baselines, and run Python linting and tests.

52. Git: rebase vs. merge; resolving conflicts.
Merge preserves history and shows integration points. Rebase rewrites your local commits onto a new base for a linear history, and should not be used on shared branches. For conflicts, understand both changes, resolve them, then build and test.

53. How do you review a PR for correctness and standards compliance?
Check the requirement and tests first, then the logic and edge cases, then the naming and documentation rules. Confirm that warnings and baselines are unchanged or justified, and that deviations are documented.

G. Scenario / Behavioural
54. Tell me about a data pipeline you built.
Use the STAR format: describe the situation, your task, the actions you took, and the measurable result, such as data volume processed, time saved, or defects found. Be honest about what went wrong and what you changed afterwards.

55. A recording replays differently on two runs. How do you investigate?
Check the inputs first, including checksums, timestamps and the replay configuration. Then look for non-determinism such as threading, uninitialised memory, time-dependent logic, and floating-point ordering. Reproduce it reliably before fixing.

56. A unit test passes locally but fails in CI. What do you check first?
Compare the environment: compiler version, build flags, OS, locale, timezone, and dependency versions. Look for test-order dependence, shared global state, and uninitialised variables. Run the test in isolation and repeatedly.

57. A warning appears after a refactor. How do you fix it without hiding a real issue?
Find the root cause: an unused variable, a sign conversion, or a shadowed name. Fix the code rather than suppressing the warning. Only use a suppression when it is justified and documented.

58. How do you explain a data anomaly to a non-technical manager?
Lead with the impact, then give a simple visual such as a plot or count, state what is known and not known, and propose the next step with a time estimate.

59. How do you prioritise when several teams request tooling at once?
Rank by impact on safety and release commitments, then by the number of people affected, and by effort. Make the trade-offs visible to the requesters and agree the order with the team lead.

60. Describe a code review disagreement and how you handled it.
Describe a concrete technical disagreement, how you ba
