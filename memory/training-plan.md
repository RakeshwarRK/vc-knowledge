# VC PR Training Plan — Full Sweep

## Strategy
- **Deep study**: Framework/architecture PRs (fetch code, blind attempt, compare, extract patterns)
- **Rapid scan**: Workload onboarding PRs (check patterns, flag deviations, batch lessons)
- **Review mining**: Extract Bryan/Yang review comments on other people's PRs
- **Skip**: Docs-only, version bumps, dependabot, typo fixes, <10 line changes

## Progress Summary
- **Total PRs in repo**: 536 merged
- **PRs studied (deep + rapid + review-mined)**: ~130
- **Patterns extracted**: 79 + Bryan's review meta-patterns
- **Sessions**: 3 (initial 11 PRs + sweep phase 1-2 + sweep phase 2-3)
- **Last updated**: 2026-03-06

## Phase 1: Bryan (brdeyo) — DONE

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
| #562 | saibulusu | 4 comments | 66 — scenario naming significance |
| #575 | saibulusu | 3 comments | Block size consistency, Duration in CLI |
| #543 | imadityaa | 1 comment | Plural naming |
| #544 | imadityaa | 1 comment | SequentialExecution naming |
| #300 | saibulusu | 6 comments | 64, 67 — parameter lockdown, intent-driven docs |
| #271 | ericavella | 5 comments | 63 — no logic in getters, static shared helpers |
| #303 | ericavella | 5 comments | 64 — lock down params, throw on unsupported |
| #246 | ericavella | 4 comments | 65 — MetricScenario override pattern |
| #127 | ericavella | 2 comments | Naming precision ("DatabaseScenario") |
| #123 | deep1712 | 2 comments | 77 — smart default parameter design |
| #414 | RakeshwarK | 3 comments | Domain naming, unit test coverage |
| #601 | RakeshwarK | 3 comments | Timespan naming, backward compat tests |
| #647 | RakeshwarK | 2 comments | 71 — method overloads, hide impl details |
| #491 | RakeshwarK | 1 comment | 70 — parameter cohesion |
| #471 | imadityaa | 1 comment | 80 — place checks naturally |
| #392 | saibulusu | 1 comment | Don't regress coverage |

### Remaining (lower priority)
- #120 (file upload, 16K lines — co-authored, may scan later)
- #563 (log upload schema, 7K lines — dedicated session needed)

## Phase 2: Yang (yangpanMS) — DONE

### Completed (deep study)
| PR | Title | Patterns |
|----|-------|----------|
| #412 | Metric verbosity (original) | 42 |
| #429 | ParallelLoopExecution | 41 |
| #309 | Certificate + managed identity | 50 |
| #360 | SupportedPlatform attribute | 53 |
| #408 | .NET 9 migration | 54 |
| #480 | Split into 4 runtime packages | (scanned) |
| #498 | CLI default changes for v2 | 55 |
| #500 | --timeout=never | 55 |
| #433 | MetricVerbosity to logging | (scanned) |
| #410 | DiskSpd skip read/write-only | 56 |
| #406 | File upload race condition | 57 |
| #484 | State manager invalid JSON | 58 |
| #434 | EventHub null ref pythonnet | (scanned) |

### Completed (rapid scan)
| PR | Title | Lessons |
|----|-------|---------|
| #449 | --logger CLI customize | (scanned) |
| #519 | Summary logger formatting | (scanned) |
| #190 | DiskSpd parser redesign | (scanned) |
| #325 | Long file paths Windows | (scanned) |
| #318 | ASP.NET server/client | (scanned) |

### Completed (bug fix study)
| PR | Title | Patterns |
|----|-------|----------|
| #48 | VC deletes mount path on raw disk | 60 |
| #50 | .NET process output missing | 58 |
| #226 | Incomplete remote profile download | 59 |
| #199 | Disk extension compare path | (copy-paste boolean bug) |
| #62 | Dependency install with disk ref | (access vs device path) |
| #66 | lspci retry rapidly | 61 |
| #374 | Duplicate metrics in memtier | (reset state per execution) |
| #313 | EventHub connection string hotfix | 62 |
| #501 | Proxy conflicts with default URI | (context-dependent defaults) |

### Completed (workload onboarding study)
| PR | Title | Patterns |
|----|-------|----------|
| #36 | Commercial workloads (SPEC, 30K lines) | 72 |
| #64 | CoreMark Pro | 72 |
| #43 | nvidia-smi GPU monitor | (monitor pattern) |
| #58 | lspci monitor | (monitor pattern) |
| #116 | Windows coremark/coremark-pro | (platform extension) |
| #238 | OpenSSL sign verify parsing | (parser enhancement) |
| #354 | SPECcpu iterations override | (parameter extension) |
| #375 | FIO custom installed binary | (flexibility) |
| #559 | Geekbench process kill wait | (process lifecycle) |

### Skipped (docs/version/infra — 60+ PRs)
PRs #1-16, #19-21, #27, #30, #39, #59, #92, #95, #104, #118, #139, #141, #149, #165, #172, #178, #182, #184, #186, #188, #193, #195, #209, #210, #216, #223, #225, #228, #230, #234, #260-270, #272-275, #283-284, #291, #297-298, #305, #307-308, #315, #321, #323, #326, #328-331, #351, #359, #361, #370-371, #378, #388, #397, #402, #404, #416, #444, #467, #469, #477, #489, #530, #545, #559, #561, #566-567, #571, #573

## Phase 3: Other Contributors — DONE

### ericavella (27 PRs)
| PR | Reviewer | Key lessons |
|----|----------|-------------|
| #22 | Yang (24 comments) | First workload review thoroughness |
| #69 | Bryan (1) + Yang (10) | Unit test coverage, param docs |
| #127 | Bryan (2) + Yang (6) | 65 — naming "DatabaseScenario", disk formatting |
| #134 | Yang (11) | Parameter referencing, naming consistency |
| #169 | Yang (6) | Mockable APIs, document parameters |
| #174 | Yang (4) | Hide internal state from users |
| #189 | Yang (3) | Avoid magic strings |
| #246 | Bryan (4) | 65 — MetricScenario override pattern |
| #271 | Bryan (5) | 63 — no getter logic, static helpers, clean/reset |
| #303 | Bryan (5) | 64, 67 — lock down params, intent docs, throw on invalid |
| Other 17 PRs | Minor/approved | (scanned) |

### saibulusu (42 PRs)
| PR | Reviewer | Key lessons |
|----|----------|-------------|
| #300 | Bryan (6) + Yang (2) | 64, 67 — profile defaults, doc intent, alphabetical params |
| #392 | Bryan (1) + Yang (5) | Don't regress coverage, runtime config injection |
| #562 | Bryan (4) | 66 — scenario names functional, fix bugs not profiles |
| #575 | Bryan (3) | Block size coverage, runtime on CLI not in jobfiles |
| #534 | Yang (2) | Use PollForHeartbeatAsync, realistic timeouts |
| #423 | Yang (4) | Test overwrite scenarios |
| Other 36 PRs | Minor/approved | (scanned) |

### deep1712 (24 PRs)
| PR | Reviewer | Key lessons |
|----|----------|-------------|
| #123 | Bryan (2) + Yang (5) | Smart null defaults, parameter grouping |
| #128 | Yang (6) | Library choice, naming, spacing |
| #179 | Yang (18) | Documentation completeness, data prep, test data integrity |
| #377 | Yang (3) | Null vs empty string precision, version freshness |
| #386 | Yang (1) | Null reference prevention |
| #168 | Bryan (8) | Already captured (patterns 47-48) |
| Other 18 PRs | Minor/approved | (scanned) |

### imadityaa (28 PRs)
| PR | Reviewer | Key lessons |
|----|----------|-------------|
| #40 | Yang (multi) | Parser design, packaging, MIT headers |
| #47 | Yang | 79 — API defaults reflect common case |
| #87 | Yang | Parameter hygiene, dead code |
| #133 | Yang | 68 — parameter isolation across components |
| #153 | Yang | Naming clarity, default placement |
| #257 | Yang | Documentation, YAGNI |
| #471 | Bryan | 80 — place checks naturally |
| #513 | Yang | Abstraction justification |
| #536 | Yang | 69 — don't create parallel types |
| #543, #544 | Bryan | Naming (already captured) |
| Other 18 PRs | Minor/approved | (scanned) |

### RakeshwarK (30 PRs)
| PR | Reviewer | Key lessons |
|----|----------|-------------|
| #414 | Bryan (3) + Yang (3) | Domain naming, unit tests, use framework utilities |
| #601 | Bryan (3) | Timespan naming, backward compat tests |
| #647 | Bryan (2) | 71 — method overloads, hide impl details |
| #491 | Bryan (1) | 70 — parameter cohesion |
| #240 | Yang (4) | Method consistency, naming nits |
| #341 | Yang (1) | Rename MaxClients |
| #350 | Yang (5) | Pascal casing, docs, encoding |
| #355 | Yang (1) | Avoid tribal knowledge |
| #407 | Yang (1) | Apply to all related profiles |
| #458 | Yang (2) | Windows fault tolerance |
| #524 | Yang (2) | Case-insensitive comparison, error messages |
| Other 19 PRs | Approved (no comments) | Clean approvals indicate growth |

### Smaller Contributors (mined for Bryan/Yang reviews)
| Author | PRs | Bryan/Yang comments found |
|--------|-----|---------------------------|
| psingla1210 | 14 | 9 with comments — 73 (remove Azure-isms), 74 (parse by column name) |
| LiYueqian-James | 8 | 5 with comments — 75 (rename after parsing), 76 (linear execution) |
| v-safilho | 6 | 2 with comments — Bryan: simplify assertions, remove flaky tests |
| nchapagain001 | 6 | 2 with comments — Bryan: infer behavior from single param |
| kayvontadj | 6 | 2 with comments — docs, remove unused code |
| cjhillbrand | 6 | 3 with comments — distinct test cases |
| muskankhedia | 4 | 2 with comments — 78 (workload constants in workload class) |
| dheeparaj | 4 | 2 with comments — version in package install, Async naming |
| bhagyeshpatil | 4 | 2 with comments — don't use regex when split works |

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
