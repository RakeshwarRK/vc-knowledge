# VC PR Training Plan — Full Sweep

## Strategy
- **Deep study**: Framework/architecture PRs (fetch code, blind attempt, compare, extract patterns)
- **Rapid scan**: Workload onboarding PRs (check patterns, flag deviations, batch lessons)
- **Skip**: Docs-only, version bumps, dependabot, typo fixes, <10 line changes

## Phase 1: Bryan (brdeyo) — Framework Authority [32 PRs, 11 done]

### Deep Study (framework-changing, >200 additions)
| PR | +/- | Title | Status |
|----|-----|-------|--------|
| #158 | +5790/-1111 | Metadata contract standardization | TODO |
| #563 | +7222/-876 | Log file telemetry upload (CSV/JSON/YAML schema) | TODO |
| #486 | +1860/-2 | Windows Event Log + Linux System Log monitors | TODO |
| #28 | +1250/-817 | FIO Discovery grouped job results | TODO |
| #33 | +1915/-547 | Custom API ports + proxy scenario | TODO |
| #41 | +1583/-725 | Proxy logger event handlers | TODO |
| #120 | +16198/-1570 | File upload logs (large, co-authored) | TODO |
| #579 | +656/-520 | .NET 8.0 multi-targeting | TODO |
| #236 | +449/-58 | Logical processors per core fix | TODO |
| #525 | +207/-37 | Experiment ID in summary logger | TODO |
| #163 | +132/-23 | --fail-fast option | TODO |
| #503 | +158/-9 | Extension propagation bug fix | TODO |
| #54 | +216/-21 | MockFixture/InMemoryApiClient design flaws | TODO |

### Rapid Scan (bug fixes, small changes)
| PR | +/- | Title | Status |
|----|-----|-------|--------|
| #618 | +223/-191 | Geekbench process kill logic | TODO |
| #481 | +45/-12 | Serilog file rolling fix | TODO |
| #483 | +71/-72 | Telemetry channel dispose | TODO |
| #502 | +59/-31 | Bootstrap no-op params | TODO |
| #564 | +68/-5 | Whitespace hotfix | TODO |
| #631 | +26/-0 | Ensure core directories exist | TODO |
| #29 | +148/-83 | DBNull handling in DataTable | TODO |
| #26 | +312/-91 | --source CLI option | TODO |
| #162 | +87/-528 | File upload descriptor cleanup | TODO |

### Skip (docs, config, trivial)
- #577 (docs move), #583 (test platform), #596 (message text), #614 (cert), #511 (nuget bump), #512 (package size), #198 (6-line log), #31 (typo), #32 (config), #53 (docs)

## Phase 2: Yang (yangpanMS) — Top Contributor [158 PRs]

### Deep Study
| PR | +/- | Title | Status |
|----|-----|-------|--------|
| #36 | +30546 | Commercial workloads (SPEC) | TODO |
| #412 | +477 | Metric verbosity + console print | TODO |
| #429 | +293 | Parallel loop execution | TODO |
| #449 | +892 | --logger CLI customize | TODO |
| #519 | +416 | Summary file logger formatting | TODO |
| #309 | +1140 | Certificate + managed identity for packages/eventhub | TODO |
| #318 | +1400 | ASP.NET server/client workload | TODO |
| #190 | +457 | DiskSpd parser redesign | TODO |
| #214 | +278 | .NET 8 migration | TODO |
| #325 | +433 | Long file paths Windows | TODO |

### Rapid Scan (workload onboarding pattern check)
| PR | +/- | Title | Status |
|----|-----|-------|--------|
| #58 | +5055 | lspci monitor | TODO |
| #64 | +4547 | CoreMark Pro | TODO |
| #68 | +1934 | SPECcpu Windows | TODO |
| #43 | +377 | nvidia-smi monitor | TODO |
| #44 | +247 | CoreMark thread overwrite | TODO |
| #110 | +464 | SuperBench H100 config | TODO |
| #116 | +401 | CoreMark/Pro Windows | TODO |
| #188 | +447 | RPM build | TODO |
| #267 | +937 | UTF-8 BOM conversion | SKIP |
| #269 | +850 | Fix unit test Linux | TODO |
| #283 | +368 | Fix broken unit tests | TODO |
| #298 | +688 | OneBranch pipeline | SKIP |
| #388 | +338 | Version bump v1.16 | SKIP |

## Phase 3: Other Contributors [~245 PRs]

### Prioritized
| Author | PR Count | Focus |
|--------|----------|-------|
| ericavella | 35 | Workload patterns, review style |
| saibulusu | 42 | Workload patterns |
| nmalkapuram | 43 | Workload patterns |
| imadityaa | 28 | Features |
| deep1712 | 26 | Features |
| RakeshwarK | 32 | Self-review for improvement |

### Strategy for Phase 3
- Batch by author, rapid-scan for pattern deviations
- Deep study only if PR introduces new framework patterns
- Focus on review comments from Bryan/Yang on other people's code

## Phase 4: Future Training Opportunities

### After PR Sweep
1. **GitHub Issues analysis** — Study open issues for design intent and feature direction
2. **Code review simulation** — Pick random PRs, write review before seeing actual reviews
3. **Blind workload onboarding** — Pick a new tool (not in VC), design executor/parser/profile from scratch
4. **Cross-repo comparison** — When CRC SDK repo is available, compare architectures
5. **Breaking change analysis** — Track all breaking changes across versions, understand migration patterns
6. **Performance optimization PRs** — Study any PRs focused on perf (caching, parallelism, memory)
7. **Bryan's rejected PRs / closed-without-merge** — Understand what approaches were abandoned and why

## Progress Tracking
- After each PR: update this file + push patterns to vc-knowledge
- After each session: update training-summary.md with new capability assessment
- Track: PRs studied, patterns extracted, session count
