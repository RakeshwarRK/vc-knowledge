# VC Memory — Recent Activity

*Auto-updated weekly by GitHub Actions. Last scan: 2026-03-05*

## Recent Commits (last 30, as of 2026-03-05)

| SHA | Date | Author | Summary |
|-----|------|--------|---------|
| f8a337cc | 2026-03-05 | brdeyo | Refactor LMbench workloads — clearer metrics across memory sweeps; fix SPEC CPU profiles |
| 6ff0a27e | 2026-03-04 | **Rakesh** | Simplify test framework with process tracking and assertion helpers (#647) |
| fccc4a76 | 2026-03-04 | erica | Sysbench: formalize database population design (#651) |
| 3ba552b0 | 2026-03-02 | brdeyo | Fix eccentricities with SockPerf and Latte workloads |
| 4ff0499b | 2026-03-03 | **Rakesh** | Standardized MetricFilters: verbosity, inclusion regex, exclusion regex (#636) |
| 4b76bdd6 | 2026-03-02 | Alex | Added check for bashrc path existence (#649) |
| 1373da0a | 2026-02-28 | brdeyo | SpecCPU: parse metrics from CSV results |
| cbfd372c | 2026-02-26 | brdeyo | Fix range of bugs: metrics CSV flush on exit, network firewall scripts, profile package refs |
| 4b1108af | 2026-02-27 | Nirjan | Enable get-token + bootstrap certificate from Key Vault (#613) |
| e0610a67 | 2026-02-23 | **Rakesh** | Enabled Windows ProcessorAffinity support (#629) |
| ca6bab4b | 2026-02-20 | deep1712 | Bump version to 2.1.57 (#646) |
| 4814000a | 2026-02-20 | deep1712 | Update LatMemRd profile and executor (#642) |
| 4f35e599 | 2026-02-20 | manuelrod | Use QueueDepth in DiskSpd MetricScenario (#645) |
| 0c8476a5 | 2026-02-19 | v-safilho | Consolidate Sysbench command arguments (#570) |
| 8ac1445b | 2026-02-18 | imadityaa | Add stateDir in profile expression evaluators (#643) |
| e1657ba2 | 2026-02-17 | Nirjan | New VirtualClientComponent: ResponseFile component (#641) |
| 880320f1 | 2026-02-11 | deep1712 | Log process details file upload; fix FolderName; bump version (#635) |
| 8914b5ed | 2026-02-05 | deep1712 | Add LatMemRd (Lat Mem Read) workload (#630) |
| e2b00302 | 2026-02-03 | Bryan | Create core directories (logs, packages) up front to avoid read/write issues (#631) |
| ef4ce85a | 2026-01-28 | Sai | SuperBench NVIDIA A100 setup profile, ansible-core, disk space (#626) |
| fa567e34 | 2026-01-27 | Sai | User-defined Benchmarks parameter for SPEC CPU profiles (#628) |
| 85a3c8af | 2026-01-26 | **Rakesh** | Onboard NCPS workload (CPS being deprecated) (#625) |
| 1364dbff | 2026-01-24 | Bryan | Better path type handling (abs/relative); nested quotes in PowerShell executor (#627) |
| 00f4f5f8 | 2026-01-23 | **Rakesh** | Enable defining specific logical processors to run a process against (#617) |
| cc392eab | 2026-01-22 | Sai | Upversion to 2.1.51 (#624) |
| 6ae53c35 | 2026-01-22 | Sai | FIO verify file path (#623) |
| e3347919 | 2026-01-17 | Sai | SPEC CPU limitations on Windows; specifying benchmarks in SPECCpuExecutor (#619) |
| 47650fff | 2026-01-15 | Bryan | Bug fix: Geekbench process kill logic (#618) |

## Notable Recent Changes

### New Features (2025–2026)
- **MetricFilters** — verbosity levels + regex inclusion/exclusion on metrics (PR #636, Rakesh)
- **Windows ProcessorAffinity** — bind workloads to specific CPU cores on Windows (PR #629, Rakesh)
- **NCPS workload** — replaces deprecated CPS for connections-per-second benchmarks (PR #625, Rakesh)
- **Logical processor targeting** — specify exact logical processors for a process (PR #617, Rakesh)
- **Key Vault integration** — get-token and bootstrap certificate from Key Vault (PR #613, Nirjan)
- **ResponseFile component** — new VirtualClientComponent type (PR #641, Nirjan)
- **LatMemRd workload** — memory latency sweep benchmarks (PR #630, deep1712)
- **stateDir in profile expressions** — `{stateDir}` well-known parameter (PR #643)

### Active Contributors
| GitHub | Real Name | Focus |
|--------|-----------|-------|
| brdeyo | Bryan DeYoung | Core platform, network workloads, bug fixes |
| **RakeshwarRK** | **Rakesh** | **CPU affinity, NCPS, metrics, test framework** |
| deep1712 | — | Version management, monitoring |
| Nirjan Chapagain | Nirjan | Key Vault, new components |
| Sai Bulusu | Sai | GPU workloads, SPEC CPU, FIO |
| erica / ericavella | Erica | Sysbench |

## Current Version Line
- **2.1.x** (currently at 2.1.57+, as of Mar 2026)

## Deprecated / Replaced
- **CPS** → replaced by **NCPS** (connections per second workload)
