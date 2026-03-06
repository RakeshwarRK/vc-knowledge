# VirtualClient Profile Catalog

**Total: 85 profiles** across 7 categories

## Profile Naming Convention
- `PERF-*` — Performance benchmarks (54 profiles)
- `MONITORS-*` — System monitoring (5 profiles)
- `EXAMPLE-*` — Reference/template (6 profiles)
- `BOOTSTRAP-*` / `GET-*` — Setup & utilities (5 profiles)
- `SETUP-*` — Hardware/OS preparation (3 profiles)
- `QUAL-*` — Qualification tests (1 profile)
- `POWER-*` — Power measurement (4 profiles)
- `UPLOAD-*` / `EXECUTE-*` — Operations (4 profiles)
- `agent/*` — Controller/agent sub-profiles (4 profiles)

## Workload Profiles by Category

### CPU Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-CPU-OPENSSL | OpenSslExecutor | linux-x64/arm64, win-x64 | CommandArguments, ciphers: md5/sha256/sha512/aes-256-cbc/rsa/ecdsa |
| PERF-CPU-OPENSSL-TLS | OpenSslTlsExecutor | linux-x64/arm64 | TLS 1.2 vs 1.3 handshake performance |
| PERF-CPU-COREMARK | CoreMarkExecutor | linux-x64/arm64, win-x64/arm64 | CompilerName=gcc, CompilerVersion=13, ThreadCount=LogicalProcessorCount |
| PERF-CPU-COREMARKPRO | CoreMarkProExecutor | linux-x64/arm64, win-x64/arm64 | CompilerFlags, Duration=00:30:00 |
| PERF-CPU-GEEKBENCH | GeekBenchExecutor | linux-x64, win-x64/arm64 | GeekBench 6 (replaced v5) |
| PERF-CPU-GEEKBENCH5 | GeekBenchExecutor | linux-x64, win-x64/arm64 | GeekBench 5 (deprecated) |
| PERF-CPU-LAPACK | LAPACKExecutor | all 4 platforms | Linear algebra (BLAS/LAPACK) |
| PERF-CPU-PRIME95 | Prime95Executor | linux-x64 | Stress/torture testing |
| PERF-CPU-HPLINPACK | HPLinpackExecutor | linux-x64/arm64 | ProblemSizeN, BlockSizeNB, CompilerName, PerfLibrary |
| PERF-CPU-HPCG | HpcgExecutor | linux-x64/arm64 | Duration=01:00:00, CompilerVersion |

### Disk I/O Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-IO-FIO | FioExecutor | linux-x64/arm64, win-x64 | 12 scenarios (random/seq read/write, 4K-1024K blocks, d1-d256), ProcessModel, DiskFilter=BiggestSize |
| PERF-IO-FIO-OLTP | FioExecutor | linux-x64/arm64, win-x64 | Database-style I/O patterns with data integrity verification |
| PERF-IO-DISKSPD | DiskSpdExecutor | win-x64/arm64 | 12 scenarios matching FIO, QueueDepth, ThreadCount |

### Memory Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-MEM-LMBENCH | LMbenchExecutor | linux-x64/arm64 | MemorySizeMB={SystemMemoryMB/4}, CompilerFlags |
| PERF-MEM-LMBENCH-SWEEP | LMbenchExecutor (LatMemRd) | linux-x64/arm64 | 4 stride sizes (128/256/512/1024 bytes), MemoryThreshold=4096 |
| PERF-MEM-STRESSAPPTEST | StressAppTestExecutor | linux-x64/arm64 | Duration=00:01:00, UseCpuStressfulMemoryCopy |

### Network Benchmarks (Client/Server)
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-NETWORK | NetworkingWorkloadExecutor | all 4 | NTttcp (9 scenarios: TCP 4K/64K/256K × T1/T32/T256) + SockPerf/Latte (3 each) + CPS (1) |
| PERF-NETWORK-2 | NCPSExecutor + NetworkingWorkloadExecutor | all 4 | NCPS replaces CPS, NTttcp + SockPerf/Latte |
| PERF-NETWORK-NTTTCP | NetworkingWorkloadExecutor | all 4 | NTttcp only (11 TCP scenarios: 4K/64K/256K × T1/T32/T256 + UDP) |
| PERF-NETWORK-CTSTRAFFIC | CtsTrafficExecutor | win-x64/arm64 | Windows-only, Duplex pattern, Connections=20 |
| PERF-NETWORK-DEATHSTARBENCH | DeathStarBenchExecutor | linux-x64 | 3 services: SocialNetwork, MediaMicroservices, HotelReservation |
| PERF-NETWORK-PING | NetworkPingExecutor | all 4 | ICMP ping latency |

### Database Benchmarks (Client/Server)
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-MYSQL-SYSBENCH-OLTP | SysbenchExecutor | linux-x64/arm64 | 10 OLTP workloads, InnodbBufferPoolSize={80% RAM}, Duration=00:05:00 |
| PERF-MYSQL-SYSBENCH-TPCC | SysbenchExecutor | linux-x64/arm64 | TPC-C benchmark, Duration=00:30:00 |
| PERF-POSTGRESQL-HAMMERDB-TPCC | HammerDBExecutor | linux-x64/arm64, win-x64 | TPC-C on PostgreSQL, TPM/NOPM metrics |
| PERF-POSTGRESQL-SYSBENCH-OLTP | SysbenchExecutor | linux-x64/arm64 | OLTP on PostgreSQL |
| PERF-POSTGRESQL-SYSBENCH-TPCC | SysbenchExecutor | linux-x64/arm64 | TPC-C on PostgreSQL |
| PERF-REDIS | RedisServer + MemtierClient | linux-x64/arm64 | 9 memtier scenarios (8/16/32 threads × 32B/1KB/10KB), TLS support, BindToCores=true |
| PERF-MEMCACHED | MemcachedServer + MemtierClient | linux-x64/arm64 | 9 memtier scenarios, ServerPort=6379 |

### Compression Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-COMPRESSION | CompressionExecutor | linux-x64/arm64 | 7zip, gzip, pbzip2 |
| PERF-COMPRESSION-LZBENCH | LZBenchExecutor | all 4 | Multi-algorithm compression speed |

### GPU/ML Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-GPU-MLPERF | MLPerfExecutor | linux-x64 | Inference on A100/A30/A2 |
| PERF-GPU-MLPERF-NVIDIA | MLPerfExecutor | linux-x64 | NVIDIA-specific inference |
| PERF-GPU-MLPERF-TRAINING-NVIDIA | MLPerfExecutor | linux-x64 | BERT training benchmark |
| PERF-GPU-SUPERBENCH | SuperBenchmarkExecutor | linux-x64 | GPU compute microbenchmarks |
| PERF-GPU-SUPERBENCH-NVIDIA | SuperBenchmarkExecutor | linux-x64 | NVIDIA-specific superbench |
| PERF-GPU-3DMARK-AMD | 3DMarkExecutor | win-x64 | AMD GPU 3D graphics |
| PERF-GPU-SPECVIEW-AMD | SPECviewperfExecutor | win-x64 | AMD GPU viewset FPS |
| PERF-GPU-FLOPS-AMD | AmdSmiMonitor + FLOPSExecutor | linux-x64 | AMD GPU FLOPS |
| PERF-BLENDER-AMD | BlenderBenchmarkExecutor | win-x64 | Scenes: monster, junkshop, classroom |

### HPC Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-GRAPH500 | Graph500Executor | linux-x64/arm64 | BFS + SSSP graph traversal |
| PERF-HPC-NASPARALLELBENCH | NASParallelBenchExecutor | linux-x64/arm64 | MPI parallel kernels |
| PERF-OPENFOAM | OpenFoamExecutor | linux-x64/arm64 | CFD simulation |

### Web/App Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-ASPNETBENCH | AspNetBenchExecutor | all 4 | JSON serialization, single-system |
| PERF-ASPNETBENCH-MULTI | AspNetBenchServer/Client | all 4 | Client/server, wrk -t256 -c256 |

### Stress Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-STRESSNG | StressNgExecutor | linux-x64/arm64 | Multi-stressor bogo-ops |

### SPEC Commercial (License Required)
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| PERF-SPECCPU-INTRATE | SpecCpuExecutor | linux-x64/arm64 | SPECrate integer |
| PERF-SPECCPU-INTSPEED | SpecCpuExecutor | linux-x64/arm64 | SPECspeed integer |
| PERF-SPECCPU-FPRATE | SpecCpuExecutor | linux-x64/arm64 | SPECrate floating point |
| PERF-SPECCPU-FPSPEED | SpecCpuExecutor | linux-x64/arm64 | SPECspeed floating point |
| PERF-SPECJBB | SpecJbbExecutor | all 4 | max-jOPS, critical-jOPS |
| PERF-SPECJVM | SpecJvmExecutor | all 4 | JVM operations/min |

### Power Benchmarks
| Profile | Executor | Platforms | Key Parameters |
|---------|----------|-----------|----------------|
| POWER-SPEC30 | SpecPowerExecutor | all 4 | 30% load level |
| POWER-SPEC50 | SpecPowerExecutor | all 4 | 50% load level |
| POWER-SPEC70 | SpecPowerExecutor | all 4 | 70% load level |
| POWER-SPEC100 | SpecPowerExecutor | all 4 | 100% load level |

## Monitor Profiles

| Profile | Monitors | Platforms |
|---------|----------|-----------|
| MONITORS-DEFAULT | PerfCounters (Win+Linux) + EventLogs + system info commands (12 monitors) | all 4 |
| MONITORS-COUNTERS | PerfCounters only (2 monitors) | all 4 |
| MONITORS-GPU-NVIDIA | NvidiaSmiMonitor | linux-x64/arm64, win-x64 |
| MONITORS-GPU-AMD | AmdSmiMonitor + LspciMonitor | linux-x64, win-x64 |
| MONITORS-FILE-UPLOAD | FileUploadMonitor | all 4 |

## Setup / Utility Profiles

| Profile | Purpose |
|---------|---------|
| BOOTSTRAP-CERTIFICATE | Install certificate from Key Vault |
| BOOTSTRAP-DEPENDENCIES | Install a VC dependency package |
| BOOTSTRAP-PACKAGE | Install a VC package from blob storage |
| SETUP-FORMAT-DISKS | Format unformatted disks |
| SETUP-NVIDIA-A100 | Install NVIDIA A100 drivers + CUDA |
| SETUP-AMD-GPU-DRIVER | Install AMD GPU drivers |
| QUAL-GPU-DCGMI | GPU qualification diagnostics |
| GET-ACCESS-TOKEN | Retrieve Key Vault access token |
| EXECUTE-COMMAND | Run arbitrary command |
| EXECUTE-COMMAND-ON-REMOTE | Run command on remote via SSH |
| UPLOAD-FILES | Upload files to blob storage |
| UPLOAD-TELEMETRY | Upload telemetry data |

## Common Profile Parameters

| Parameter | Default | Used By |
|-----------|---------|---------|
| Duration | varies (00:01:00 to 01:00:00) | Most workloads |
| DiskFilter | BiggestSize | FIO, DiskSpd |
| ProcessModel | SingleProcess | FIO, DiskSpd |
| CompilerName | gcc | CoreMark, LMbench, HPLinpack |
| CompilerVersion | 13 | CoreMark, HPLinpack |
| ProfilingEnabled | False | Most profiles |
| ProfilingMode | None/Interval | Most profiles |
| MonitorFrequency | 00:05:00 | Monitor profiles |
| MonitorWarmupPeriod | 00:05:00 | Monitor profiles |
| ServerPort | varies | Client/server workloads |
| ClientInstances | varies (1-8) | Client/server workloads |
| BindToCores | true | Redis, Memcached |

## Profile Parameter Reference Syntax
- `$.Parameters.X` — Reference another profile parameter
- `{calculate({SystemMemoryBytes} * 80 / 100)}` — Runtime calculation
- `{SystemMemoryMegabytes}` — System info variable
- `{LogicalProcessorCount}` — CPU count variable
