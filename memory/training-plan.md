# VC PR Training Plan — Full Sweep

## Strategy
- **Deep study**: Framework/architecture PRs (fetch code, blind attempt, compare, extract patterns)
- **Rapid scan**: Workload onboarding PRs (check patterns, flag deviations, batch lessons)
- **Review mining**: Extract Bryan/Yang review comments on other people's PRs
- **Skip**: Docs-only, version bumps, dependabot, typo fixes, <10 line changes

## Progress Summary
- **Total PRs in repo**: 536 merged
- **PRs studied (deep + rapid + review-mined)**: ~35
- **Patterns extracted**: 52 + Bryan's review meta-patterns
- **Sessions**: 2 (initial 11 PRs + current sweep)
- **Last updated**: 2026-03-06

## Phase 1: Bryan (brdeyo) — DONE for high-value PRs

### Completed (deep study)
| PR | Title | Patterns |
|----|-------|----------|
| #576 | Controller/Agent Architecture | 11-15 |
| #595 | Status Code Registry | error-handling.md |
| #606 | Kernel Race Condition Fix | error-handling.md |
| #549 | FIO SingleProcessAggregated | 16-20 |
| #494 | Script Subcommands | 21-23 |
| #547 | SSH Client Refactor for SCP | 24-25 |
| #627 | Path Handling & ScriptExecutor | 26-28 |
| #565 | Disk Mount Permissions | 29-33 |
| #158 | Metadata Contract | 34-35 |
| #486 | Event Log Monitors | 36 |
| #163 | --fail-fast option | 37 |
| #503 | Extension propagation bug | 38 |
| #28 | FIO grouped job results | 52 |
| #33 | Custom API ports | 51 |
| #236 | Logical processors fix | (scanned) |

### Completed (rapid scan / review mining)
| PR | Title | Lessons |
|----|-------|---------|
| #618 | Geekbench kill -9 0 | 39 |
| #631 | Core directories race | 40 |
| #481 | Serilog rolling fix | (scanned) |
| #54 | MockFixture design flaws | (scanned) |
| #29 | DBNull handling | (scanned) |

### Bryan review comments mined from other people's PRs
| PR | Author | Comments | Key lessons |
|----|--------|----------|-------------|
| #55 | nmalkapuram | 15 comments | 43, 45, 49 — consolidate executors, naming, pin versions |
| #65 | nmalkapuram | 12 comments | 44, 45, 46 — least privilege, naming, test precision |
| #46 | nmalkapuram | 4 comments | 44, 45 — naming, least privilege |
| #168 | deep1712 | 8 comments | 47, 48, 45 — state tracking, IsSupported, naming |
| #562 | saibulusu | 4 comments | Scenario naming significance, Duration pattern |
| #575 | saibulusu | 3 comments | Block size consistency, Duration in CLI |
| #543 | imadityaa | 1 comment | Plural naming |
| #544 | imadityaa | 1 comment | SequentialExecution naming |

### Remaining (lower priority)
- #120 (file upload, 16K lines — co-authored, may scan later)
- #563 (log upload schema, 7K lines — dedicated session needed)
- #579 (.NET 8 targeting — build config, low pattern value)
- #41, #525, #162, #502, #564 (minor fixes)

## Phase 2: Yang (yangpanMS) — IN PROGRESS

### Completed
| PR | Title | Patterns |
|----|-------|----------|
| #412 | Metric verbosity (original) | 42 |
| #429 | ParallelLoopExecution | 41 |
| #309 | Certificate + managed identity | 50 |
| #449 | --logger CLI customize | (scanned) |
| #519 | Summary logger formatting | (scanned) |
| #190 | DiskSpd parser redesign | (scanned) |
| #325 | Long file paths Windows | (scanned) |
| #318 | ASP.NET server/client | (scanned) |

### TODO
| PR | Title | Priority |
|----|-------|----------|
| #36 | Commercial workloads (SPEC, 30K lines) | HIGH — dedicated session |
| #214 | .NET 8 migration | MEDIUM |
| Remaining ~130 Yang PRs | Workload onboarding, config | Batch scan in next session |

## Phase 3: Other Contributors — TODO

### Prioritized for Bryan review mining
| Author | PR Count | Strategy |
|--------|----------|----------|
| ericavella | 35 | Mine Bryan/Yang review comments |
| saibulusu | 42 | Mine Bryan/Yang review comments |
| nmalkapuram | 43 | DONE (PRs #46, #55, #65 mined) |
| imadityaa | 28 | Partially done (#543, #544) |
| deep1712 | 26 | Partially done (#168) |
| RakeshwarK | 32 | Self-review (done #636, #625) |

## Phase 4: Future Training (after PR sweep)

### Immediate next opportunities
1. **Bryan's rejected/closed-without-merge PRs** — What approaches were abandoned and why?
2. **GitHub Issues** — Open issues reveal design intent and feature direction
3. **Code review simulation** — Pick random new PRs, write review before seeing actual reviews
4. **Blind workload onboarding** — Design executor/parser/profile for a tool not in VC

### Longer-term
5. **CRC SDK comparison** — When user is ready, create separate knowledge repo
6. **Cross-cutting refactor practice** — E.g., migrate all parsers to new verbosity (real task)
7. **Performance PRs** — Any PRs focused on caching, parallelism, memory optimization
8. **Architecture decision records** — Document why VC chose X over Y for major decisions
