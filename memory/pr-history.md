# VirtualClient PR History & Evolution

**Coverage:** PRs #1–#651 (Nov 2022 – Mar 2026), ~560 merged, ~90 unmerged/open

## Project Timeline

### Phase 1: Open-Source Setup (Nov 2022 – Jan 2023, PRs #1–#50)
- PR #2: Repository initialization (yangpanMS)
- PR #22: Sysbench OLTP client-server — first major workload (ericavella)
- PR #26: `--source` CLI flag for content/package stores (brdeyo)
- PR #35: Docker container support (yangpanMS)
- PR #43: nvidia-smi GPU monitor (yangpanMS)

### Phase 2: Workload Expansion (Jan–Jun 2023, PRs #40–#125)
- StressAppTest (#40), PostgreSQL (#46), DCGM (#55), CoreMark Pro (#64), Intel Linpack (#65)
- 3DMark + MSRA Microbenchmark (#75), NVIDIA CUDA Windows (#87)
- FIO Discovery Win-x64 (#96), Debian OS support (#104)

### Phase 3: Architecture Standardization (Jul–Dec 2023, PRs #126–#240)
- SSH Client Support (#128), CTS Traffic (#142), Charm++ HPC (#168)
- SPECviewperf (#175), MLPerf Training (#179), Blender (#197), Wrathmark (#201)
- **PR #158: Standardized metadata contract** (brdeyo) — breaking telemetry change
- **PR #214: .NET 8 migration** (yangpanMS)
- PR #240: RakeshwarK's first merged PR (OpenSSL parser)

### Phase 4: v1.14/v1.15 Maturation (Jan–Jun 2024, PRs #241–#333)
- ApacheBench (#251), Generic Script Executor (#257), Hadoop Terasort (#262)
- **PR #309: Certificate + Managed Identity support** (yangpanMS)
- Sysbench TPC-C (#303), PostgreSQL HammerDB redesign (#295)

### Phase 5: v1.16 Cross-Platform (Jul 2024 – Jan 2025, PRs #334–#432)
- **PR #360: SupportedPlatform attribute** (yangpanMS)
- DCGM ARM64 (#377), **PR #429: Parallel loop execution** (yangpanMS)
- PR #412: Metric verbosity levels (yangpanMS)

### Phase 6: v2.0 / .NET 9 (Feb–Jul 2025, PRs #433–#552)
- **PR #408: .NET 9 migration** (yangpanMS)
- **PR #477: Version bump to v2** (yangpanMS)
- **PR #480: Split into 4 runtime packages** (yangpanMS)
- **PR #494: command/pwsh/python subcommands** (brdeyo)
- **PR #513: Key Vault integration** (imadityaa)
- PR #486: Windows Event Log + Linux System Log monitors (brdeyo)

### Phase 7: v2.1 Stabilization (Aug 2025 – Mar 2026, PRs #553–#651)
- **PR #576: VC as test controller** (brdeyo)
- PR #569: Conditional parameter sets (saibulusu)
- **PR #601: All profiles use timespans** (RakeshwarK) — breaking profile change
- **PR #617: Logical processor targeting** (RakeshwarK)
- **PR #625: NCPS workload** (RakeshwarK) — replaces deprecated CPS
- **PR #629: Windows ProcessorAffinity** (RakeshwarK)
- **PR #636: Standardized metric filtering** (RakeshwarK)
- PR #647: Test framework process tracking (RakeshwarK)
- PRs #638–640: First AI-authored PRs (GitHub Copilot)

## Breaking Changes Timeline

| PR | Change | Author |
|---|---|---|
| #158 | Standardized metadata contract (telemetry) | brdeyo |
| #193 | Install path `/usr/local/bin` → `/usr/bin` | yangpanMS |
| #214 | .NET 8 migration | yangpanMS |
| #360 | SupportedPlatform attribute on components | yangpanMS |
| #408 | .NET 9 migration | yangpanMS |
| #477 | v2 version bump + CLI defaults | yangpanMS |
| #480 | Split into 4 runtime packages | yangpanMS |
| #512 | Monolithic NuGet package removed | brdeyo |
| #566 | File logger removed from defaults | yangpanMS |
| #575 | FIO discovery + multithroughput profiles removed | saibulusu |
| #576 | Controller/agent architecture | brdeyo |
| #601 | Timespan format for all profile time values | RakeshwarK |
| #617 | Logical processor targeting per process | RakeshwarK |
| #636 | Standardized metric filtering (verbosity/regex) | RakeshwarK |

## Deprecations

| What | Replaced By | PR |
|---|---|---|
| Geekbench 5 | Geekbench 6 | #224 |
| `/usr/local/bin` install | `/usr/bin` | #193 |
| Extreme Numerics library | removed | #402 |
| Monolithic NuGet package | 4 split packages | #512 |
| FIO DISCOVERY + MULTITHROUGHPUT | removed | #575 |
| CPS workload | NCPS | #625 |
| .NET 8 (main binary) | .NET 9 | #408 |
| Integer time in profiles | TimeSpan format | #601 |

## RakeshwarK Contribution Timeline (34 merged PRs)

| PR | Title | Date |
|---|---|---|
| #240 | Extended OpenSSL Metrics Parser | 2024-01 |
| #286 | Updated OpenSSL Documentation | 2024-03 |
| #327 | ASPBench Package Management | 2024-06 |
| #341 | Memtier ClientsMax optional parameter | 2024-07 |
| #345 | HPLinpack ARM Perf Libraries v24.04 | 2024-09 |
| #350 | Bind Memtier client to ALL processors | 2024-07 |
| #355 | MaxClients corner case handling | 2024-08 |
| #400 | HPL ARM Performance Library latest | 2024-11 |
| #407 | SPECcpu optional Threads + Copies | 2024-11 |
| #411 | Parameter references for SPEC profiles | 2024-12 |
| #414 | Git checkout option for GitRepoClone | 2024-12 |
| #424 | ASPNET profiles update | 2025-01 |
| #458 | SPECcpu compiler check fix Windows | 2025-03 |
| #470 | apiHost URL: localhost → `*` | 2025-03 |
| #474 | Retry policy for Networking Workload | 2025-03 |
| #491 | HPLinpack ARM Perf Libs v25 | 2025-05 |
| #524 | HPLinpack AMD + Intel Perf Libs | 2025-06 |
| #537 | Suppress wget cert check in Compression | 2025-06 |
| #588 | Conditional parameter sets bug fix | 2025-10 |
| #601 | All profiles use timespans | 2025-12 |
| #612 | Size property on Windows disks | 2026-01 |
| #617 | Logical processor targeting | 2026-01 |
| #625 | NCPS workload (CPS replacement) | 2026-01 |
| #629 | Windows ProcessorAffinity | 2026-02 |
| #636 | Standardized metric filtering | 2026-03 |
| #647 | Test framework process tracking | 2026-03 |

**Ownership areas:** OpenSSL, HPLinpack (ARM/AMD/Intel), Memtier, ASPBench, SPECcpu, Networking, test infrastructure

## Key Contributors

| Contributor | Primary Areas |
|---|---|
| **yangpanMS** | Architecture lead, .NET migrations, CI/CD, CoreMark, profiles |
| **brdeyo** | Framework core, telemetry, metadata, REST API, SSH, logging |
| **ericavella** | Database workloads: Sysbench, PostgreSQL, MySQL |
| **imadityaa** | Script executors, NVIDIA, content paths, Key Vault, parallel |
| **saibulusu** | RHEL 9, FIO, CompilerInstallation, AzLinux3, SPECcpu |
| **nmalkapuram** | PostgreSQL, Redis, Memcached, Linpack, DCGM |
| **deep1712** | MLPerf, DCGM ARM, SSH, FIO, LatMemRd |
| **RakeshwarK** | OpenSSL, HPLinpack, Memtier, ASPBench, SPECcpu, test infra |

## Architecture Decisions

- **ContentPathTemplate (#133):** Template-based blob upload paths replaced hardcoded descriptors
- **Metadata Contract (#158):** Unified system context metadata across all workloads
- **SupportedPlatform Attribute (#360):** Runtime platform enforcement via attribute
- **v2 Package Split (#480):** Monolithic NuGet exceeded size limit → 4 packages
- **Controller/Agent (#576):** Multi-VM orchestration as first-class pattern
- **Generic Script Executor (#257):** Run arbitrary scripts without full C# components
- **Conditional Parameter Sets (#569):** Runtime-conditional profile parameters
- **Parallel Loop Execution (#429):** Concurrent workload execution patterns
