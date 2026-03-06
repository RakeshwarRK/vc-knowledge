# VirtualClient Vision & Purpose

**Living document** — Updated as understanding deepens. Open questions flagged for resolution.

## Core Purpose

VirtualClient is the **standard testing solution for Azure cloud systems at scale — primarily for post-deployment validation and large-scale cloud customer workload simulation.**

The primary use cases:
1. **Post-deployment testing** — After hardware is deployed in Azure data centers, VC runs qualification suites to validate that systems perform correctly under realistic conditions.
2. **Customer workload simulation at scale** — VC simulates the kinds of workloads Azure customers run (database, networking, compute, storage) across thousands of machines to validate system behavior at cloud scale.

VC is also used by hardware engineers for pre-deployment validation, but a newer solution — **CRC SDK** — is being promoted specifically for that hardware engineer pre-deployment scenario.

VC is open-sourced for public use (community benchmarking), but the primary mission is Azure post-deployment validation and scale testing.

**VC answers: "Is this deployed cloud infrastructure performing correctly for customers?"**

## Core Values

1. **Highly reliable** — Runs on production systems, unsupervised, at massive scale. Must not crash on transient failures. Must produce results or report exactly why it couldn't.
2. **Easily scalable** — Same tool, same profiles, same telemetry format whether you're testing 1 machine or 10,000. No per-machine configuration needed beyond the profile.
3. **Self-contained** — Carries everything it needs. No assumption about what's installed on the target system. Critical for consistency across thousands of diverse machines.

## Design Philosophy

### 1. Execution engine, not orchestrator
VC handles **how** to test. The orchestration layer above (Azure experiment framework) decides **what** to test and **where**. The controller/agent pattern (#576) enables multi-machine test scenarios (network throughput between client/server VMs) without requiring external coordination, but VC is not trying to replace orchestration infrastructure.

### 2. Profile-driven, not code-driven
Workload configurations are JSON profiles, not hardcoded in executors:
- Same executor, different profiles → different test configurations
- Profiles are composable (multiple on command line)
- Parameters are overridable at runtime (`--parameters`)
- Non-developers can create and modify test configurations

### 3. Telemetry is a first-class output
VC isn't just "run a benchmark" — it's "run a benchmark and produce structured data that feeds automated qualification decisions." The metadata contract (#158), telemetry schema, and Event Hub integration exist because downstream data pipelines and automated pass/fail gates depend on consistent, structured output.

### 4. Platform abstraction
Executors don't think about Linux vs Windows. They use `ISystemManagement` for disk, process, firewall, package operations. Platform-specific code is pushed to the edges (numactl for Linux affinity, Win32 API for Windows affinity). Essential because Azure ships hardware with both Windows and Linux images.

### 5. Defensive execution
Post-deployment testing means running on production-grade but potentially misconfigured systems:
- Transient errors are swallowed (ErrorReason 100-399) — the test continues
- Only truly terminal conditions crash VC (ErrorReason ≥ 500)
- Processes are cleaned up on exit (SafeKill, CleanupTasks)
- Kernel-level race conditions are handled (#606)
- Retry policies with backoff for network/infrastructure flakiness

## VC vs CRC SDK — Positioning

| Aspect | VirtualClient | CRC SDK |
|--------|--------------|---------|
| **Primary use** | Post-deployment validation, scale customer simulation | Pre-deployment hardware qualification |
| **Target user** | Cloud infra teams, scale test engineers | Hardware engineers |
| **When in lifecycle** | After hardware is deployed in data centers | Before deployment, during hardware bring-up |
| **Scale** | Thousands of machines simultaneously | Individual systems or small batches |
| **Status** | Mature, open-sourced, battle-tested | Newer, being promoted for HW engineer scenarios |

*Note: CRC SDK details to be expanded as understanding develops. Keeping knowledge bases separate to avoid context mixing.*

## Strategic Direction (inferred from PR trajectory)

### Completed transitions
- **Monolithic → modular**: Single NuGet → 4 split packages (#480)
- **.NET 8 → .NET 9**: Runtime modernization (#408)
- **Single-system → multi-system**: Client/server → controller/agent (#576)
- **Hardcoded → parameterized**: Template paths (#133), parameter references, conditional sets (#569)
- **Basic → sophisticated metrics**: 3-level verbosity → 5-level with regex filtering (#636)

### Current trajectory (v2.1+)
- **Script-first extensibility**: `command`, `pwsh`, `python` subcommands (#494) lower the barrier — engineers can write Python scripts without C#
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

1. **Azure scale test / post-deployment teams** — Primary users. Run validation suites on deployed infrastructure.
2. **Azure CRC team** — Build and maintain the workloads and profiles.
3. **Azure partner teams** — Use VC for scenario-specific testing (network, storage, GPU).
4. **Downstream data systems** — Kusto queries, Power BI dashboards, automated pass/fail gates.
5. **Hardware engineers** — Pre-deployment use (overlaps with CRC SDK).
6. **Open-source community** — External users benchmarking their own systems.

## Open Questions

### RESOLVED: Orchestration boundary
VC is the execution engine. Orchestration lives above (Azure experiment framework). Controller/agent (#576) handles multi-machine test coordination for scenarios that inherently require it.

### RESOLVED: Primary use case
Post-deployment validation and scale customer simulation. Pre-deployment hardware qualification is shifting toward CRC SDK.

### OPEN: VC ↔ CRC SDK boundary
Where exactly does VC's responsibility end and CRC SDK's begin? Are there overlapping workloads? Do they share packages/profiles? To be explored separately.

### OPEN: Commercial workload licensing model
SPECcpu, SPECjbb, GeekBench require licenses. Current model is "bring your own package." Is there a plan for a managed licensing experience?

### OPEN: Extension discoverability
Extensions are DLLs dropped in `packages/`. No registry, no versioning contract. As the ecosystem grows, does this need more structure?

### OPEN: GPU workload maturity
GPU workloads are mostly Linux-x64 only. AMD vs NVIDIA creates parallel codepaths. Is there a unified GPU abstraction planned?

### OPEN: Metric classification framework
PR #636 introduced the 1-5 verbosity scale, but classification is per-parser judgment. Is there a formal framework for what constitutes a Level 1 metric?

## Core Maintainers & Their Vision Areas

| Person | Area | Vision Influence |
|--------|------|-----------------|
| **brdeyo** | Framework architecture, telemetry, REST API, controller/agent | Platform extensibility, data quality, orchestration scope |
| **yangpanMS** | .NET runtime, CI/CD, platform support, profiles | Cross-platform parity, build/release modernization |
| **ericavella** | Database workloads | Database benchmark standardization |
| **RakeshwarK** | Metrics, affinity, test infrastructure, networking | Metric sophistication, process control, test reliability |
