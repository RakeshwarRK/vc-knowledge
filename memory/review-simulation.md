# Drill Results — Session 4 (2026-03-06)

## Drill 1: Framework Capability Audit

### VirtualClientComponent Base Properties (32 total)

**From ISystemInfo (read-only):** AgentId, CpuArchitecture, ExperimentId, Platform, PlatformSpecifics

**From Parameters dictionary:**
| Property | Type | Default | Parameter Key |
|----------|------|---------|---------------|
| Scenario | string | null | "Scenario" |
| PackageName | string | null | "PackageName" |
| MetricScenario | string | null | "MetricScenario" |
| MetricFilters | IEnumerable<string> | empty | "MetricFilters" or "MetricFilter" |
| Roles | IEnumerable<string> | null | "Role" or "Roles" |
| SupportedPlatforms | IEnumerable<string> | empty | "SupportedPlatforms" or "Platforms" |
| Tags | IEnumerable<string> | empty | "Tags" |
| FailFast | bool | false | "FailFast" |
| LogToFile | bool | false | "LogToFile" |
| LogTimestamped | bool | true | "LogTimestamped" |
| LogFileName | string | null | "LogFileName" |
| LogFolderName | string | null | "LogFolderName" |
| Seed | int | 777 | "Seed" |
| ProfilingEnabled | bool | false | "ProfilingEnabled" |
| ProfilingMode | ProfilingMode | None | "ProfilingMode" |
| ProfilingInterval | TimeSpan | Zero | "ProfilingInterval" |
| ProfilingPeriod | TimeSpan | Zero | "ProfilingPeriod" |
| ProfilingWarmUpPeriod | TimeSpan | Zero | "ProfilingWarmUpPeriod" |
| ProfilingScenario | string | null | "ProfilingScenario" |
| ProfileIteration | int | 0 | "ProfileIteration" (set by ProfileExecutor) |

**Mutable state:** CleanupTasks, Extensions, Metadata, MetadataContract, Logger, Dependencies, ContentPathTemplate, ClientRequestId, SupportedRoles, TypeName

### Virtual/Overridable Methods (10)
1. `ExecuteAsync(EventContext, CancellationToken)` — **abstract**, core logic
2. `InitializeAsync(EventContext, CancellationToken)` — setup, default no-op
3. `Validate()` — validation, default no-op
4. `CleanupAsync(EventContext, CancellationToken)` — runs CleanupTasks
5. `IsSupported()` — checks platforms + roles
6. `IsInRole(string)` — checks Layout role
7. `Dispose(bool)` — default no-op
8. `WaitAsync(CancellationToken)` — idle until cancelled
9. `WaitAsync(DateTime, CancellationToken)` — idle until time
10. `WaitAsync(TimeSpan, CancellationToken)` — idle for duration

### Lifecycle Order
EvaluateParameters → InitializeAsync → Validate → ExecuteAsync → LogSuccessOrFailedMetric → CleanupAsync

### Profile-Level (Top JSON) Parameters
- `MinimumExecutionInterval` — TimeSpan, gates minimum time between action rounds
- `Parameters` — merged into each component
- `Metadata` — merged into each component
- `ParametersOn` — conditional parameter sets

### Key Extension Methods (most-used)
- `ExecuteCommandAsync` — run process
- `GetPackageAsync` / `GetPlatformSpecificPackageAsync` — get registered packages
- `LoadResultsAsync` — read results files
- `LogProcessDetailsAsync` — log process output to telemetry/file
- `MakeFileExecutableAsync` — chmod +x
- `GetLayoutClientInstance` / `GetLayoutClientInstances` — client-server layout
- `ThrowIfParameterNotDefined` / `ThrowIfLayoutNotDefined` — validation helpers
- `ApplyParameters` — replace {paramName} placeholders in text
- `Combine` — OS-aware path combine
- `LogSuccessOrFailedMetric` — emit Succeeded/Failed metric

---

## Drill 2: 1-Issue Review (5 PRs, forced single-issue constraint)

| PR | My Finding | Bryan/Yang Finding | Score |
|----|-----------|-------------------|-------|
| #641 | Suspected naming mismatch | File name doesn't match class name | Partial |
| #613 | chmod 777 security | Architectural: backward compat, profile separation | Miss |
| #636 | Default verbosity breaking change | Bryan silently approved | Match (vacuous) |
| #544 | Should be SequentialExecution | Should be SequentialExecution | Match |
| #536 | Don't create parallel data model | Don't create new class, use existing Metric | Match |

**Score: 3/5** (up from ~2/5 in session 3)

**Key lesson**: Bryan reviews at architecture level (backward compat, profile composition, naming scope), not implementation detail. On #613 I flagged a security issue while Bryan flagged design: "don't break existing consumers, separate into distinct profiles, rename command to reflect broader scope."

---

## Drill 3: Blind CoreMark Parser

### What I Got Right
- Correct base class (MetricsParser)
- Identified all relevant metrics
- Iterations/Sec as HigherIsBetter

### What I Missed
1. **DataTable pattern** — actual uses `ConvertToDataTable()` + `GetMetrics()`, not individual regex
2. **Metric names preserved verbatim** from tool output (don't rename)
3. **Units are plain strings** ("bytes", "ticks"), not MetricUnit constants
4. **Verbosity must be set** — primary=1, secondary=5
5. **Conservative with Relativity** — only HigherIsBetter for true perf metrics, Undefined for config
6. **Preprocess filters lines** — common pattern: filter to delimiter-matching lines, parse as table

---

## Drill 4: Profile Design (HPCG — Blind Attempt)

**Score: 2/11**

### Key Misses
| Aspect | My Attempt | Actual | Lesson |
|--------|-----------|--------|--------|
| MinimumExecutionInterval | 00:30:00 | 00:01:00 | It's a minimum gap, not duration |
| SupportedOperatingSystems | Missing | CBL-Mariner,CentOS,etc | Separate metadata field |
| CompilerVersion | "11" | "" (empty) | Default empty = use system |
| Scenario | "ExecuteHpcg" | "Gflops" | Metric-focused, not action |
| Duration | In profile | Not in profile | HPCG has fixed internal duration |
| Install method | Blob download | Spack (GitRepoClone) | HPC workloads use Spack |
| CompilerInstallation | Missing | Present | Own dependency type |

### 6 Installation Archetypes (from 55+ profiles)
1. **Download & run** — DependencyPackageInstallation only (Geekbench, OpenSSL)
2. **Build from source** — CompilerInstallation + GitRepoClone (CoreMark, HPCG)
3. **Windows cross-platform** — Chocolatey + Compiler + DependencyPackage (Lapack)
4. **Client-server** — DependencyPackage + ApiServer (Redis, Memcached)
5. **IO workload** — FormatDisks + MountDisks + DependencyPackage (FIO, DiskSpd)
6. **GPU/Container** — FormatDisks + Docker + NvidiaCuda + GitRepoClone (MLPerf)

---

## Drill 5: Scenario Naming Taxonomy

| Pattern | Example | When Used |
|---------|---------|-----------|
| Metric/output name | Gflops, SHA256, RSA4096 | Single-metric workloads |
| Operation_Parameters | RandomWrite_4k_BlockSize | IO with varying config |
| Algorithm_Mode | 7zLZMAUltraMode | Compression variants |
| Protocol_Pattern | SockPerf_TCP_Ping_Pong | Network workloads |
| Benchmark class | bt.D.x, sp.D.x | NAS Parallel (matches tool naming) |
| Functional role | Server, warmup_10kb_pool | Client-server components |

**Rule**: Names describe WHAT is measured, never the action verb. Never "Execute" or "Run".

---

## Drill 6: Unit Test Patterns (from SequentialExecutionTests)

1. MockFixture in [SetUp], not per-test
2. Test naming: `ClassName_WhatItDoes_WhenCondition`
3. TestComponent inner class with delegate + execution counter
4. Test wrapper class exposes protected methods via `new` keyword
5. Four coverage areas always present: happy path, cancellation, error propagation, unsupported skip
6. Assert messages always descriptive
7. Delegate-based test doubles, no mocking framework
8. Direct assertions (Assert.AreEqual), not fluent

---

## Drill 7: ErrorReason Ranges (Corrected)

| Range | Category | Examples |
|-------|----------|---------|
| 0 | Undefined | Default |
| 100-199 | Monitor errors | PerformanceCounterNotFound, MonitorUnexpectedAnomaly |
| 300-399 | Workload execution | WorkloadFailed (315), WorkloadResultsNotFound (314), HttpErrors (320-325) |
| 400-499 | Serious but transient | InvalidResults (400), ApiStatePollingTimeout (410), WorkloadUnexpectedAnomaly (430) |
| 500+ | Terminal/non-recoverable | ProfileNotFound (500), PlatformNotSupported (503), DependencyNotFound (507) |

---

## Updated Weakness Assessment

| Weakness | Previous | After Drills | Status |
|----------|----------|-------------|--------|
| Naming hierarchy | Critical gap | Resolved — full taxonomy mapped | Fixed |
| Yang's operational knowledge | Critical gap | Major improvement — 32 properties + 30 extensions mapped | Mostly fixed |
| Over-reviewing | Calibration issue | 1-issue constraint working (3/5 score) | Improving |
| Premature generalization | Pattern mismatch | Aware but needs practice | Ongoing |
| Parser DataTable pattern | Unknown gap | Now identified — use ConvertToDataTable + GetMetrics | New learning |
| Installation archetypes | Unknown gap | 6 patterns mapped | New learning |
| Profile design | Unknown gap | Major gap (2/11), needs more practice | New weakness |
| Scenario naming | Minor gap | Full taxonomy mapped | Fixed |

## Recommended Next Drills
1. **Profile Design Round 2** — try IO workload (FIO) and client-server (Redis) profiles blind
2. **DataTable Parser Drill** — rewrite CoreMark parser using correct pattern, try NTttcp
3. **Architecture Review Drill** — practice Bryan's architecture-level reviewing (backward compat, profile composition)
4. **Extension Method Usage Drill** — given a scenario, pick the right extension methods
