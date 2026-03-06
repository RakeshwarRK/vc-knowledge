# VirtualClient Vision & Purpose

**Living document** — Updated as understanding deepens. Open questions flagged for resolution.

## Core Purpose

VirtualClient exists to answer one question at Azure scale: **"Is this hardware ready for customers?"**

It's the defacto standard workload execution platform for Azure Cloud hardware validation — qualifying CPUs, GPUs, disks, NICs, and memory across every VM SKU before customers see them. Every Azure region, every new chip generation, every firmware update runs through VC.

## Why It Exists

Azure partner teams (CRC — Cloud Readiness Certified) needed a single tool that could:
1. Run the same benchmarks identically across thousands of machines
2. Produce structured, comparable telemetry at scale
3. Be entirely self-contained (no external dependencies on the system under test)
4. Support both single-machine and multi-machine (client/server) topologies
5. Run on every platform Azure ships: Windows/Linux × x64/ARM64

Before VC, teams had ad-hoc scripts, manual benchmark runs, inconsistent data formats. VC standardized this into a repeatable, automatable pipeline.

## Design Philosophy

### 1. Self-contained execution
VC carries everything it needs. Dependency packages are downloaded once, cached, and reused. No assumption about what's installed on the target system. This is critical for bare-metal qualification where the OS may be freshly imaged.

### 2. Profile-driven, not code-driven
Workload configurations are JSON profiles, not hardcoded in executors. This means:
- Same executor, different profiles → different test configurations
- Profiles are composable (multiple on command line)
- Parameters are overridable at runtime (`--parameters`)
- Non-developers can create and modify test configurations

### 3. Telemetry is a first-class output
VC isn't just "run a benchmark" — it's "run a benchmark and produce structured data that feeds automated decision systems." The metadata contract (#158), telemetry schema, and Event Hub integration exist because the data pipeline downstream is as important as the execution.

### 4. Platform abstraction
Executors don't think about Linux vs Windows. They use `ISystemManagement` for disk, process, firewall, package operations. Platform-specific code is pushed to the edges (numactl for Linux affinity, Win32 API for Windows affinity).

### 5. Defensive execution
Hardware qualification means running on potentially broken hardware. VC must:
- Not crash on transient errors (ErrorReason 100-399 = swallowed)
- Only crash on truly terminal conditions (ErrorReason ≥ 500)
- Clean up processes on exit (SafeKill, CleanupTasks)
- Handle kernel-level race conditions (#606)

## Strategic Direction (inferred from PR trajectory)

### Completed transitions
- **Monolithic → modular**: Single NuGet → 4 split packages (#480)
- **.NET 8 → .NET 9**: Runtime modernization (#408)
- **Single-system → multi-system**: Client/server → controller/agent (#576)
- **Hardcoded → parameterized**: Template paths (#133), parameter references, conditional sets (#569)
- **Basic → sophisticated metrics**: 3-level verbosity → 5-level with regex filtering (#636)

### Current trajectory (v2.1+)
- **VC as orchestrator**: Controller/agent pattern means VC can drive tests across multiple machines via SSH, not just run locally
- **Script-first extensibility**: `command`, `pwsh`, `python` subcommands (#494) lower the barrier — you don't need C# to use VC
- **Metric sophistication**: Verbosity levels + regex filters give downstream consumers fine-grained control over data volume
- **Platform breadth**: ARM64 parity, AzureLinux/Mariner support, GPU workloads growing

## Strengths

1. **Battle-tested at Azure scale** — Runs on thousands of machines daily. Edge cases are found and fixed in production.
2. **Comprehensive workload coverage** — 40+ workloads across CPU, disk, memory, network, database, GPU, HPC, compression, web.
3. **Structured telemetry pipeline** — Data flows from VC → Event Hub → Kusto/ADX → automated qualification decisions.
4. **Self-contained packaging** — `.vcpkg` format with platform-specific folders means no "works on my machine" issues.
5. **Extension model** — Teams can build private workloads without forking the repo.

## Gaps & Tensions

### OPEN QUESTION: What's the boundary between VC and orchestration?
The controller/agent pattern (#576) pushes VC toward being an orchestrator. But VC was designed as a single-machine execution engine. How far does orchestration go? Does VC replace Ansible/Terraform for test infrastructure, or does it stay focused on workload execution?

### OPEN QUESTION: Commercial workload licensing model
SPECcpu, SPECjbb, GeekBench require licenses. The current model is "bring your own package." Is there a plan for a managed licensing experience, or does VC stay agnostic?

### OPEN QUESTION: Extension discoverability
Extensions are DLLs dropped in `packages/`. There's no registry, no versioning contract beyond "must target .NET 9 and contain VirtualClient in the assembly name." As the extension ecosystem grows, does this need more structure?

### OPEN QUESTION: GPU workload maturity
GPU workloads (MLPerf, SuperBench, DCGM) are growing but are mostly Linux-x64 only. The AMD vs NVIDIA split creates parallel codepaths. Is there a unified GPU abstraction planned?

### OPEN QUESTION: What makes a metric "critical" vs "verbose"?
PR #636 introduced the 1-5 scale, but the classification is per-parser, per-contributor judgment. Is there a formal framework for what constitutes a Level 1 metric? Or does each workload owner decide?

## Who VC Serves

1. **Azure CRC engineers** — Primary users. Run qualification suites on new hardware.
2. **Azure partner teams** — Use VC for scenario-specific testing (network, storage, GPU).
3. **External community** — Open-source users benchmarking their own systems.
4. **Downstream data systems** — Kusto queries, Power BI dashboards, automated pass/fail gates that consume VC telemetry.

## Core Maintainers & Their Vision Areas

| Person | Area | Vision Influence |
|--------|------|-----------------|
| **brdeyo** | Framework architecture, telemetry, REST API, controller/agent | Platform extensibility, data quality, orchestration |
| **yangpanMS** | .NET runtime, CI/CD, platform support, profiles | Cross-platform parity, build/release modernization |
| **ericavella** | Database workloads | Database benchmark standardization |
| **RakeshwarK** | Metrics, affinity, test infrastructure, networking | Metric sophistication, process control precision |
