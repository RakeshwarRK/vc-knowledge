# Code Review Simulation — Results & Drill Analysis

## Aggregate Scorecard (8 PRs reviewed)

### Bryan Calibration (5 PRs)

| PR | Type | Hit Rate | Key Miss |
|----|------|----------|----------|
| #641 | New component | 1/1 (100%) | None — caught file name mismatch |
| #645 | 1-line fix | N/A | Clean approval matched |
| #613 | Security/framework | 4/4 (100%) | None — caught all architectural issues |
| #584 | Network workload | 1/2 (50%) | Missed assertion simplification pattern |
| #544 | Framework executor | 1/1 principle, 0/1 direction | **Got naming direction backwards** |
| **Average** | | **~70%** | Naming hierarchy understanding |

### Yang Calibration (2 PRs)

| PR | Type | Hit Rate | Key Miss |
|----|------|----------|----------|
| #520 | Parser update | 1/2 (50%) | Didn't recognize Async suffix as Yang's request |
| #504 | Cooldown feature | 0.5/2 (25%) | **Recommended opposite of Yang** (premature generalization), missed MinimumExecutionInterval overlap |
| **Average** | | **~38%** | Yang's practical/operational knowledge |

## Identified Weaknesses (Drill Angles)

### 1. NAMING HIERARCHY (Critical Gap)
- Failed on PR #544: recommended `SequentialLoopExecution` when Bryan wanted `SequentialExecution`
- Root cause: compared to wrong existing component (ParallelLoopExecution vs ParallelExecution)
- **Drill needed**: Map the complete VC execution model hierarchy and naming logic
- Pattern: name for the execution model, not the optional behavior

### 2. YANG'S OPERATIONAL KNOWLEDGE (Critical Gap)
- Failed on PR #504: didn't know `MinimumExecutionInterval` exists
- Failed on PR #520: didn't recognize convention-enforcement as blocking
- Root cause: insufficient knowledge of existing framework capabilities
- **Drill needed**: Map all existing VirtualClientComponent properties and profile-level parameters

### 3. ASSERTION SIMPLIFICATION (Minor Gap)
- Missed on PR #584: Bryan prefers flat assertions over if/else branching
- **Drill**: Practice rewriting test assertions in Bryan's preferred style

### 4. OVER-REVIEWING (Calibration Issue)
- Average 8-10 items flagged per PR; Bryan/Yang raise 1-2
- Need to prioritize: ONE blocking issue per round
- **Drill**: For each review, force-rank findings and only present top 1-2

### 5. PREMATURE GENERALIZATION (Pattern Mismatch)
- On PR #504: suggested moving cooldown to base class
- Yang explicitly said "add in your component first"
- VC philosophy: start specific, generalize when pattern proves itself
- Contradicts my instinct to DRY/abstract immediately

## Other Drill Types Attempted

### Parser Writing Drill (LatMemRd)
- **Result**: Good structural match (sectionize, metadata, LowerIsBetter)
- **Gap**: Didn't encode array size in metric name (actual implementation does)
- **Gap**: Didn't use TextParsingExtensions.Sectionize (framework utility)
- **Lesson**: VC prefers unique metric names over generic name + metadata

### Architecture Decision Drill (Verbosity: int vs enum)
- **Result**: Correctly predicted VC's integer choice and Bryan's reasoning
- **Lesson**: Good intuition for VC design philosophy

## Recommended Next Drills (Priority Order)

1. **VC Execution Model Map** — Draw the complete hierarchy: ParallelExecution, ParallelLoopExecution, SequentialExecution, VirtualClientComponent lifecycle. Understand naming logic.

2. **Framework Capability Audit** — List all existing VirtualClientComponent base properties, profile-level parameters (like MinimumExecutionInterval), and utility methods. This is what Yang knows cold.

3. **1-Issue Review Drill** — Review 5 more PRs but force yourself to identify ONLY the single most important issue. Practice Bryan's economy.

4. **Blind Parser Drill Round 2** — Try CoreMark, HammerDB, or NTttcp parsers. Focus on using TextParsingExtensions and MetricUnit constants.

5. **Profile Design Drill** — Given a workload spec, design the complete profile JSON with correct Dependencies/Actions/Monitors ordering, parameter references, and platform metadata.
