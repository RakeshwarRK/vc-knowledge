# VirtualClient Vision & Purpose

**Living document** — Updated as understanding deepens. Open questions flagged for resolution.

## Core Purpose

VirtualClient is the **standard testing solution for Azure cloud hardware validation at data center scale.**

Hardware engineers use VC to run tests after firmware updates, hardware improvements, and new hardware validations — before that hardware is deployed to data centers serving cloud customers. Every new chip generation, every firmware revision, every NIC qualification runs through VC.

**VC answers: "Is this hardware ready for Azure customers?"**

It is also open-sourced for public use — community members can benchmark their own systems — but the primary mission is Azure hardware qualification at scale.

## Core Values

1. **Highly reliable** — Runs on potentially broken hardware, in production data centers, unsupervised. Must not crash on transient failures. Must produce results or report exactly why it couldn't.
2. **Easily scalable** — Same tool, same profiles, same telemetry format whether you're testing 1 machine or 10,000. No per-machine configuration needed beyond the profile.
3. **Self-contained** — Carries everything it needs. No assumption about what's installed on the target system. Critical for bare-metal qualification where the OS may be freshly imaged.

## Design Philosophy

### 1. Execution engine, not orchestrator
VC handles **how** to test. The orchestration layer above (Azure experiment framework) decides **what** to test and **where**. The controller/agent pattern (#576) enables multi-machine test scenarios (network throughput between client/server VMs) without requiring external coordination, but VC is not trying to replace orchestration infrastructure.

### 2. Profile-driven, not code-driven
Workload configurations are JSON profiles, not hardcoded in executors:
- Same executor, different profiles → different test configurations
- Profiles are composable (multiple on command line)
- Parameters are overridable at runtime (`--parameters`)
- Hardware engineers (non-developers) can create and modify test configurations

### 3. Telemetry is a first-class output
VC isn't just "run a benchmark" — it's "run a benchmark and produce structured data that feeds automated qualification decisions." The metadata contract (#158), telemetry schema, and Event Hub integration exist because downstream data pipelines and automated pass/fail gates depend on consistent, structured output.

### 4. Platform abstraction
Executors don't think about Linux vs Windows. They use `ISystemManagement` for disk, process, firewall, package operations. Platform-specific code is pushed to the edges (numactl for Linux affinity, Win32 API for Windows affinity). This is essential because Azure ships hardware with both Windows and Linux images.

### 5. Defensive execution
Hardware qualification means running on potentially broken hardware:
- Transient errors are swallowed (ErrorReason 100-399) — the test continues
- Only truly terminal conditions crash VC (ErrorReason ≥ 500)
- Processes are cleaned up on exit (SafeKill, CleanupTasks)
- Kernel-level race conditions are handled (#606)
- Retry policies with backoff for network/infrastructure flakiness

## Strategic Direction (inferred from PR trajectory)

### Completed transitions
- **Monolithic → modular**: Single NuGet → 4 split packages (#480)
- **.NET 8 → .NET 9**: Runtime modernization (#408)
- **Single-system → multi-system**: Client/server → controller/agent (#576)
- **Hardcoded → parameterized**: Template paths (#133), parameter references, conditional sets (#569)
- **Basic → sophisticated metrics**: 3-level verbosity → 5-level with regex filtering (#636)

### Current trajectory (v2.1+)
- **Script-first extensibility**: `command`, `pwsh`, `python` subcommands (#494) lower the barrier — hardware engineers can write Python scripts without C#
- **Metric sophistication**: Verbosity levels + regex filters give downstream consumers fine-grained control over data volume
- **Platform breadth**: ARM64 parity, AzureLinux/Mariner support, GPU workloads growing
- **Test infrastructure**: Process tracking, assertion helpers (#647) make it easier to write reliable tests

## Strengths

1. **Battle-tested at Azure scale** — Runs on thousands of machines daily in production data centers
2. **Comprehensive workload coverage** — 40+ workloads across CPU, disk, memory, network, database, GPU, HPC, compression, web
3. **Structured telemetry pipeline** — Data flows VC → Event Hub → Kusto/ADX → automated qualification gates
4. **Self-contained packaging** — `.vcpkg` format with platform-specific folders means no "works on my machine"
5. **Extension model** — Teams can build private workloads without forking

## Who VC Serves (in priority order)

1. **Azure hardware engineers** — Primary users. Run qualification suites on new/updated hardware in data centers.
2. **Azure CRC team** — Build and maintain the workloads and profiles.
3. **Azure partner teams** — Use VC for scenario-specific testing (network, storage, GPU).
4. **Downstream data systems** — Kusto queries, Power BI dashboards, automated pass/fail gates that consume VC telemetry.
5. **Open-source community** — External users benchmarking their own systems.

## Open Questions

### RESOLVED: Orchestration boundary
~~What's the boundary between VC and orchestration?~~ **Answer:** VC is the execution engine. Orchestration lives above (Azure experiment framework). Controller/agent (#576) handles multi-machine test coordination for scenarios that inherently require it (networking, databases), not general orchestration.

### OPEN: Commercial workload licensing model
SPECcpu, SPECjbb, GeekBench require licenses. Current model is "bring your own package." Is there a plan for a managed licensing experience?

### OPEN: Extension discoverability
Extensions are DLLs dropped in `packages/`. No registry, no versioning contract beyond ".NET 9 + VirtualClient in assembly name." As the extension ecosystem grows, does this need more structure?

### OPEN: GPU workload maturity
GPU workloads (MLPerf, SuperBench, DCGM) are mostly Linux-x64 only. AMD vs NVIDIA creates parallel codepaths. Is there a unified GPU abstraction planned?

### OPEN: Metric classification framework
PR #636 introduced the 1-5 verbosity scale, but classification is per-parser judgment. Is there a formal framework for what constitutes a Level 1 metric, or does each workload owner decide?

## Core Maintainers & Their Vision Areas

| Person | Area | Vision Influence |
|--------|------|-----------------|
| **brdeyo** | Framework architecture, telemetry, REST API, controller/agent | Platform extensibility, data quality, orchestration scope |
| **yangpanMS** | .NET runtime, CI/CD, platform support, profiles | Cross-platform parity, build/release modernization |
| **ericavella** | Database workloads | Database benchmark standardization |
| **RakeshwarK** | Metrics, affinity, test infrastructure, networking | Metric sophistication, process control, test reliability |
