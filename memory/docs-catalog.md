# VirtualClient Documentation & Issues Catalog

## Documentation Structure

```
website/docs/
├── overview/          # Platform overview, features, design, roadmap, FAQ
├── guides/            # Getting started, CLI, profiles, client/server, telemetry
├── workloads/         # Per-workload docs + profile docs (40+ workloads)
├── dependencies/      # 21 dependency component docs
├── monitors/          # 7 monitor docs
└── developing/        # Developer guide, extensions, onboarding, CI/CD, testing
```

## Workload Documentation Summary

### CPU (9 workloads)
| Workload | Platforms | License | Key Metrics |
|----------|-----------|---------|-------------|
| CoreMark | linux-x64/arm64 | Free | Iterations/sec |
| CoreMark Pro | all 4 | Free | Multi/single-core scores |
| OpenSSL | linux-x64/arm64, win-x64 | Free | Bytes/sec per algorithm |
| GeekBench 6 | linux-x64, win-x64/arm64 | Free (basic) | Single/multi-core scores |
| SPEC CPU 2017 | linux-x64/arm64 | Commercial | SPECrate/SPECspeed int/fp |
| SPEC JVM 2008 | all 4 | Commercial | Operations/min |
| SPEC JBB 2015 | all 4 | Commercial | max-jOPS, critical-jOPS |
| Prime95 | linux-x64 | Free | Pass/fail, stress test |
| LAPACK | all 4 | Free | Computation times |

### Disk I/O (2 workloads)
| Workload | Platforms | Key Metrics |
|----------|-----------|-------------|
| FIO | linux-x64/arm64, win-x64 | IOPS, bandwidth, latency percentiles |
| DiskSpd | win-x64/arm64 | IOPS, throughput, latency |

### Memory (1 workload)
| Workload | Platforms | Key Metrics |
|----------|-----------|-------------|
| LMbench | linux-x64/arm64 | Bandwidth (MB/s), latency (ns) |

### Network (5 workloads, all client/server)
| Workload | Platforms | Key Metrics |
|----------|-----------|-------------|
| NTttcp | all 4 | Bandwidth (Gbps), retransmits |
| NCPS (was CPS) | all 4 | Connections/sec |
| Latte | win-x64/arm64 | Latency (us) |
| SockPerf | linux-x64/arm64 | Latency (us) |
| Network Ping | all 4 | Ping latency, packet loss |

### Database (4 workloads, all client/server)
| Workload | Platforms | Key Metrics |
|----------|-----------|-------------|
| Sysbench (MySQL) | linux-x64/arm64 | TPS, QPS, latency |
| HammerDB (PostgreSQL) | linux-x64/arm64, win-x64 | TPM, NOPM |
| Redis | linux-x64/arm64 | Ops/sec, latency |
| Memcached | linux-x64/arm64 | Ops/sec, latency |

### Compression (4 workloads)
7zip, Gzip, Pbzip2 (linux only), LZBench (all platforms)

### GPU/ML (6 workloads)
MLPerf, SuperBenchmark, DCGMI, Blender, 3DMark, SPECviewperf

### HPC (5 workloads, linux only)
HPLinpack, HPCG, NAS Parallel, Graph500, OpenFOAM

### Stress (2 workloads, linux only)
Stress-ng, StressAppTest

### Web/Microservice (2 workloads)
AspNetBench (all platforms), DeathStarBench (linux-x64)

### Power (1 workload)
SPEC Power 2008 (all platforms, commercial)

## Dependency Components (21 types)

| Component | Purpose |
|-----------|---------|
| DependencyPackageInstallation | Download VC packages from Azure Blob |
| LinuxPackageInstallation | apt/dnf/yum/zypper package install |
| ChocolateyInstallation | Windows Chocolatey setup |
| ChocolateyPackageInstallation | Install via Chocolatey |
| CompilerInstallation | GCC, Charm++, Cygwin |
| CudaAndNvidiaGPUDriverInstallation | CUDA + NVIDIA drivers |
| AmdGpuDriverInstallation | AMD GPU drivers |
| DockerInstallation | Docker CE |
| DotNetInstallation | .NET SDK |
| JavaDevelopmentKitInstallation | OpenJDK 11/16/17/21 |
| MsmpiInstallation | Microsoft MPI |
| MySQLServerInstallation | MySQL server |
| MySQLServerConfiguration | MySQL config/start/users |
| PostgreSQLInstallation | PostgreSQL server |
| FormatDisks | Partition/format disks |
| MountDisks | Mount disk volumes |
| GitRepoClone | Clone git repos |
| WgetPackageInstallation | Download from URIs |
| ExecuteCommand | Run arbitrary commands |
| SetEnvironmentVariable | Set env vars |
| WaitExecutor | Wait between steps |

## Monitor Components

| Monitor | Purpose | Platforms |
|---------|---------|-----------|
| LinuxPerformanceCounterMonitor | CPU/memory/disk/network via Atop | linux |
| WindowsPerformanceCounterMonitor | Windows perf counters | win |
| NvidiaSmiMonitor | GPU metrics, ECC, temps, C2C | linux/win |
| AmdSmiMonitor | AMD GPU metrics | linux/win |
| LspciMonitor | PCI device info | linux/win |
| WindowsEventLogMonitor | System/Application event logs | win |
| LinuxEventLogMonitor | journalctl system logs | linux |
| FileUploadMonitor | Upload files to blob | all |

## Developer Guide Key Points

1. **Extension model:** NuGet package `VirtualClient.Framework`, DLL must contain "VirtualClient" in name
2. **Onboarding steps:** Research → Package → Parser+Tests → PackageManager → Profile → Executor+Tests
3. **Component base:** All derive from `VirtualClientComponent`
4. **Test framework:** `VirtualClient.TestFramework` NuGet
5. **CI/CD:** GitHub Actions PR builds, MSFT signing for releases
6. **Coding standards:** Enforced via PR review, documented in `0080-coding-standards.md`

## GitHub Issues Summary

**Total: 10 issues found (all closed)** — Most tracking is internal to Microsoft.

| Category | Count | Notable |
|----------|-------|---------|
| Bugs | 4 | Parser crashes, build inconsistencies, Docker image issues |
| Feature Requests | 2 | Ubuntu 24.04 packaging, MySQL OLTP |
| Documentation | 2 | Getting-started gaps, commercial workload docs |
| Questions | 2 | Metrics collection customization |

Common themes: parser fragility with unexpected output, build SDK sensitivity, docs missing critical params.
