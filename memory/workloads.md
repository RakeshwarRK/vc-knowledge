# VC Memory — Workloads

## CPU Benchmarks
| Workload | Profile(s) | Platform | Notes |
|---------|-----------|---------|-------|
| OpenSSL Speed | `PERF-CPU-OPENSSL.json` | linux, win | Crypto perf; SHA, AES, RSA |
| OpenSSL TLS | `PERF-CPU-OPENSSL-TLS.json` | linux, win | TLS handshake benchmark; client/server |
| CoreMark | `PERF-CPU-COREMARK.json` | linux, win | EEMBC CPU benchmark |
| CoreMark Pro | `PERF-CPU-COREMARKPRO.json` | linux, win | Multi-threaded CoreMark |
| GeekBench 5/6 | `PERF-CPU-GEEKBENCH.json`, `PERF-CPU-GEEKBENCH5.json` | linux, win | Commercial; requires license |
| HPLinpack | `PERF-CPU-HPLINPACK.json` | linux | HPC floating-point benchmark |
| HPCG | `PERF-CPU-HPCG.json` | linux | Conjugate gradient; HPC |
| LAPACK | `PERF-CPU-LAPACK.json` | linux | Linear algebra |
| NAS Parallel Bench | `PERF-HPC-NASPARALLELBENCH.json` | linux | HPC benchmark suite |
| Graph500 | `PERF-GRAPH500.json` | linux | Graph traversal benchmark |
| Prime95 | `PERF-CPU-PRIME95.json` | win | Stress test |
| StressNG | `PERF-STRESSNG.json` | linux | Stress testing |
| StressAppTest | `PERF-STRESSAPPTEST.json` | linux | Memory stress |
| LMbench | — | linux | Memory/latency micro-benchmarks |
| LatMemRd | — | linux | Memory latency sweep |

## I/O Benchmarks
| Workload | Profile(s) | Platform | Notes |
|---------|-----------|---------|-------|
| FIO | `PERF-IO-FIO.json`, `PERF-IO-FIO-OLTP.json` | linux | Flexible I/O; OLTP job files |
| DiskSpd | `PERF-IO-DISKSPD.json` | win | Windows disk I/O |
| Lzbench | `PERF-COMPRESSION-LZBENCH.json` | linux | Compression benchmark |
| 7-zip | `PERF-COMPRESSION.json` | linux, win | Compression |
| Pbzip2 | `PERF-COMPRESSION.json` | linux | Parallel bzip2 |
| OpenFOAM | `PERF-OPENFOAM.json` | linux | CFD simulation |

## Network Benchmarks (client/server)
| Workload | Profile(s) | Platform | Notes |
|---------|-----------|---------|-------|
| NTTTCP | `PERF-NETWORK-*.json` | win | Windows network throughput |
| SockPerf | `PERF-NETWORK-*.json` | linux | Latency benchmark |
| Latte | `PERF-NETWORK-*.json` | win | Windows latency |
| CPS | `PERF-NETWORK-*.json` | linux, win | Connections per second (deprecated → NCPS) |
| NCPS | — | linux, win | New connections per second (replaces CPS, merged Jan 2026) |
| CtsTraffic | `PERF-NETWORK-*.json` | win | Windows network traffic |

## Database Benchmarks (client/server)
| Workload | Profile(s) | Platform | Notes |
|---------|-----------|---------|-------|
| Redis | `GET-STARTED-REDIS.json` | linux | Memtier load generator |
| Memcached | `PERF-MEMCACHED.json` | linux | Memtier load generator |
| Sysbench | `PERF-SYSBENCH-*.json` | linux | MySQL/PostgreSQL OLTP |
| HammerDB | `PERF-HAMMERDB-*.json` | linux, win | DB benchmark suite |
| PostgreSQL | — | linux | via Sysbench |

## Web Server Benchmarks
| Workload | Profile(s) | Platform | Notes |
|---------|-----------|---------|-------|
| ASPNetBench | `PERF-ASPNETBENCH.json`, `PERF-ASPNETBENCH-MULTI.json` | linux, win | .NET web server; uses Bombardier |
| DeathStarBench | `PERF-DEATHSTARBENCH-*.json` | linux | Microservices benchmark |

## GPU Benchmarks
| Workload | Profile(s) | Platform | Notes |
|---------|-----------|---------|-------|
| MLPerf Inference | `PERF-GPU-MLPERF-NVIDIA.json`, `PERF-GPU-MLPERF.json` | linux | NVIDIA GPU |
| MLPerf Training | `PERF-GPU-MLPERF-TRAINING-NVIDIA.json` | linux | NVIDIA GPU |
| SuperBenchmark | `PERF-GPU-SUPERBENCH-NVIDIA.json`, `PERF-GPU-SUPERBENCH.json` | linux | NVIDIA GPU micro-benchmarks |
| 3DMark (AMD) | `PERF-GPU-3DMARK-AMD.json` | win | AMD GPU |
| DX Flops (AMD) | `PERF-GPU-FLOPS-AMD.json` | win | AMD GPU FP throughput |
| SpecView (AMD) | `PERF-GPU-SPECVIEW-AMD.json` | win | AMD GPU |
| Blender (AMD) | `PERF-BLENDER-AMD.json` | win | GPU rendering |
| DCGMI | — | linux | NVIDIA GPU diagnostics/monitoring |

## Monitor Profiles
| Profile | Purpose |
|---------|---------|
| `MONITORS-DEFAULT.json` | Standard perf counters |
| `MONITORS-COUNTERS.json` | Detailed perf counters |
| `MONITORS-GPU-NVIDIA.json` | NVIDIA GPU monitoring |
| `MONITORS-GPU-AMD.json` | AMD GPU monitoring |
| `MONITORS-FILE-UPLOAD.json` | Upload log files to blob storage |

## Bootstrap/Utility Profiles
| Profile | Purpose |
|---------|---------|
| `BOOTSTRAP-DEPENDENCIES.json` | Install common dependencies |
| `BOOTSTRAP-CERTIFICATE.json` | Install certificates from Key Vault |
| `BOOTSTRAP-PACKAGE.json` | Install a single package |
| `GET-ACCESS-TOKEN.json` | Fetch auth token (Key Vault) |
| `EXECUTE-COMMAND.json` | Run arbitrary command |
| `EXECUTE-COMMAND-ON-REMOTE.json` | Run command on remote system |

## Workload Onboarding Steps (summary)
1. Research workload; understand output format
2. Create `.vcpkg` package + upload to storage
3. Create `{Workload}ResultsParser.cs` + unit tests with example outputs
4. Create `PERF-{CAT}-{NAME}.json` profile
5. Create `{Workload}Executor.cs` + unit tests
6. Add dependency handler if needed
7. Run on Azure VMs and verify telemetry
8. Write docs: overview page + profiles page + metrics page
