# VC Review Simulation Results

## Session 5 Cumulative Results (2026-03-06)

### Bryan Prediction Accuracy
| Round | Method | Score |
|-------|--------|-------|
| Session 3 | Pattern matching | ~70% |
| Session 4a | 1-issue drill (5 PRs) | 60% (3/5) |
| Session 4b | Taxonomy derivation | Taxonomy complete |
| Session 5 | Live blind review (6 scoreable PRs) | **83% (4 strong + 2 partial)** |

### Yang Prediction Accuracy
| Round | Method | Score |
|-------|--------|-------|
| Session 3 | Ad hoc | ~38% |
| Session 4b | Convention checklist | Checklist complete |
| Session 5 | Deep calibration (10 PRs, 95+ comments) | **Decision tree complete** |

### Profile Design Trajectory
| Profile | Score | Key Gap |
|---------|-------|---------|
| HPCG | 2/11 (18%) | Everything wrong |
| FIO | 6.5/10 (65%) | DataIntegrity, Linux pkgs |
| Redis | 4.5/10 (45%) | CommandLine, source compilation |
| Memcached v1 | 3.7/10 (23%) | JSONPath, warmup, scenarios |
| DiskSpd | ~45/100 (45%) | Parser format (-Rtext not -Rxml) |
| Memcached v2 | 5.5/10 (55%) | Threading model, scenario naming |
| SockPerf (network) | 1.9/10 (19%) | Wrong architecture (combined suite, meta-orchestrator) |
| CoreMark | 3.7/10 (37%) | Over-applied OpenSSL patterns, wrong dep types |
| OpenSSL | 5.0/10 (50%) | Algorithm coverage perfect, missed Tags/MetricScenario/threading |
| Prime95 | 6.9/10 (69%) | Naming rules + calculate expression perfect |
| LMBench | 7.1/10 (71%) | First >70%, per-config RecommendedMin |
| Ping | 7.3/10 (73%) | v1→v2 network evolution |
| PostgreSQL OLTP | **8.5/10 (85%)** | Database mastery — reference translation |
| StressNg | 4.4/10 (44%) | Used blob install for system package |
| GeekBench | 7.2/10 (72%) | Executor-first, decent |
| LAPACK | 4.3/10 (43%) | Chocolatey chain unknown, platform 171 violation |
| LZBench | 6.0/10 (60%) | CompilerInstallation missed, wrong param architecture |
| **PostgreSQL TPCC** | **9.5/10 (95%)** | **Database translation pattern validated — near-perfect** |
| LZBench | 6.0/10 (60%) | CompilerInstallation missed, wrong param architecture |
| **SPECjbb** | **8.3/10 (83%)** | **First non-database >80%! Dep catalog working** |

### Domain Knowledge Coverage
| Workload | WHY Understood | Key Insight |
|----------|---------------|-------------|
| FIO | Complete | 512 QD invariant, FileSize > RAM, steady-state SSD |
| DiskSpd | Complete | Symmetric with FIO for cross-platform comparison |
| Redis | Complete | Single-threaded scaling, key space sizing, TLS propagation |
| Memcached | Complete | Multi-threaded vs Redis single-threaded |
| CoreMark | Complete | Compile-time threading, auto-calibration |
| OpenSSL | Complete | 13 algorithms cover real crypto needs, -multi Linux only |
| Geekbench | Complete | Black-box, version-pinned, license required |

### Edge Case Coverage
| Category | Patterns Learned | Key Pattern |
|----------|-----------------|-------------|
| Process lifecycle | No-output, hung process, non-zero success | SafeKill in finally, success codes list |
| Multi-process | WhenAny vs WhenAll | Servers=WhenAny, workloads=WhenAll |
| State persistence | DiskFill, server state via API | Save after completion, check before start |
| Platform logic | Binary paths, env vars, command flags | Conditional in InitializeAsync |
| Cleanup | CleanupTasks + finally + Dispose | Three layers for different lifecycles |
| Retry | Two-layer retry, exclude cancellation | Flow-level + instance-level |

### Session 5 Drill Summary
| # | Drill | Score/Result | Patterns Added |
|---|-------|-------------|----------------|
| 1 | Memcached v2 blind design | 5.5/10 (up from 3.7) | 101-105 |
| 2 | Redis/Memcached comparison | Structural analysis | 106-108 |
| 3 | Live PR blind review (8 PRs) | 83% Bryan match | 109-111 |
| 4 | Cross-cutting refactor drill | IO profile analysis | (lessons) |
| 5 | SockPerf executor deep dive | Architecture analysis | 112-120 |
| 6 | Yang calibration (10 PRs) | Decision tree built | (yang-taxonomy.md) |
| 7 | Domain knowledge (6 workloads) | WHY for all params | 121-127 |
| 8 | Edge case patterns (6 categories) | Process/retry/cleanup | 128-132 |
| 9 | Calculate expression deep dive | Roslyn CSharpScript, 3-phase eval | 133-136 |
| 10 | SockPerf network profile blind design | 1.9/10 — meta-orchestrator discovered | 137-143 |
| 11 | OpenSSL profile blind design | 5.0/10 — algorithm coverage perfect | 144-148 |
| 12 | CoreMark profile blind design | 3.7/10 — over-applied patterns | 149-153 |
| 13 | Scenario naming synthesis (15 profiles) | 6 naming rules derived | 154 |
| 14 | Prime95 profile blind design | 6.9/10 — best score yet | 155-156 |
| 15 | Command line pattern analysis (7 profiles) | Two models identified | 157-158 |
| 16 | Dependency catalog drill (18 profiles) | 21 types, 7 combinations | 159-160 |
| 17 | LMBench profile blind design | **7.1/10 — first >70%!** | 161-164 |
| 18 | Ping profile blind design | 7.3/10 | 165-166 |
| 19 | PostgreSQL OLTP blind design | **8.5/10 — new high, database mastery!** | 167 |
| 20 | StressNg blind design | 4.4/10 — system pkg mistake | 168-171 |
| 21 | GeekBench blind design | 7.2/10 — executor-first decent | (lessons) |
| 22 | LAPACK blind design | 4.3/10 — Chocolatey chain miss | 172-175 |
| 23 | LZBench blind design | 6.0/10 — CompilerInstallation miss | 176-179 |
| 24 | PostgreSQL TPCC blind design | **9.5/10 — database translation confirmed!** | (validation) |
| 25 | LZBench blind design | 6.0/10 — CompilerInstallation miss | 176-179 |
| 26 | **SPECjbb blind design** | **8.3/10 — first non-DB >80%!** | 187-189 |
| 27 | PR #651 blind review | 5 issues found (code dup, breaking change, shared type) | (review drill) |
| 28 | Dependency catalog sweep (30+ profiles) | Full co-occurrence patterns | 180-186, 190 |
| 29 | Parser architecture analysis (5 parsers) | 4 approaches by format | 191 |
| 30 | Contributor review patterns (5 contributors) | Prediction rules derived | 192-194 |
| 31 | **PR #648 blind description + review (iter 1)** | Blind: 65%, Review: 2/5 Bryan issues | 195-202 |
| 32 | **PR #648 re-test (iter 2)** | Blind: 88%, Review: 5/5 Bryan issues | 202a, 203-206 |
| 33 | **PR #648 deep extraction (iter 3)** | Blind: ~95%, all 16 files predicted | 207-212 |

### Pattern Count Growth
| Session | Added | Total |
|---------|-------|-------|
| 1 (initial) | 35 | 35 |
| 2 (sweep 1-2) | 16 | 51 |
| 3 (sweep 2-3) | 28 | 79 |
| 4a (drills) | 8 | 88 |
| 4b (deep drills) | 10 | 98 |
| 5 (mastery drills) | 38 | 136 |
| 5c (continued) | 28 | 164 |
| 6 (executor-first+catalog) | 30 | **194** |
| 7 (PR #648 training) | 18 | **212** |
