# VC Memory — Profiles

## Overview
Profiles are JSON (or YAML) documents defining what VC does on a system. They are "recipes".

## Profile Sections
| Section | Required | Description |
|---------|----------|-------------|
| `Metadata` | No | Informational + a few runtime-affecting properties |
| `Parameters` | No | Global overridable parameters (defaults) |
| `Actions` | No | Workloads to run sequentially |
| `Dependencies` | No | Prerequisites to install/configure (run first, sequentially) |
| `Monitors` | No | Background monitors (run in parallel with Actions) |

## Execution Order (Always)
```
1. Dependencies  (all, sequential)
2. Monitors      (all, in-parallel — start almost immediately)
3. Actions       (all, sequential — start as soon as monitors are running)
```

## Profile Naming Conventions
| Prefix | Purpose | Example |
|--------|---------|---------|
| `PERF-` | Performance workload | `PERF-CPU-OPENSSL.json` |
| `PERF-CPU-` | CPU benchmarks | `PERF-CPU-COREMARK.json` |
| `PERF-IO-` | I/O benchmarks | `PERF-IO-FIO.json` |
| `PERF-GPU-` | GPU benchmarks | `PERF-GPU-MLPERF-NVIDIA.json` |
| `MONITORS-` | Monitor-only profiles | `MONITORS-DEFAULT.json` |
| `BOOTSTRAP-` | Bootstrap/setup | `BOOTSTRAP-DEPENDENCIES.json` |
| `EXAMPLE-` | Reference examples | `EXAMPLE-WORKLOAD.json` |
| `GET-STARTED-` | Hello-world examples | `GET-STARTED-OPENSSL.json` |

## Metadata Properties
| Property | Type | Effect |
|----------|------|--------|
| `SupportsIterations` | bool | If false, `--iterations` CLI flag is ignored |
| `RecommendedMinimumExecutionTime` | timespan | Guidance for `--timeout` |
| `SupportedPlatforms` | string | e.g. `"win-x64,linux-x64"` |
| `SupportedOperatingSystems` | string | e.g. `"Ubuntu,Windows"` |

## Parameters System
### Global Parameters
Defined at profile top level; passed down to all components. Overridable via `--parameters` on CLI:
```bash
VirtualClient.exe --profile=PROFILE.json --parameters="ThreadCount=16,FileSize=100G"
```

### Component-Level Parameter Reference
```json
"ThreadCount": "$.Parameters.ThreadCount"   // reference global param
"Port": "$.Parameters.ServerPort"            // reference another global param
```

### Inline Self-References (within same component)
```json
"CommandLine": "--name={Scenario}_{FileSize} --size={FileSize}"
// {Scenario} and {FileSize} reference other params in same component
```

## Well-Known Parameters (system-resolved at runtime)
| Parameter | Resolves To |
|-----------|-------------|
| `{Architecture}` | `x64` or `arm64` |
| `{Platform}` | `linux-x64`, `win-x64`, `linux-arm64`, `win-arm64` |
| `{OS}` | `linux` or `windows` |
| `{LogicalProcessorCount}` | vCPU count |
| `{PhysicalProcessorCount}` | Physical core count |
| `{SystemMemoryBytes/KB/MB/GB}` | Total RAM in various units |
| `{ExperimentId}` | Experiment correlation ID |
| `{LogDir}` | Application logs directory path |
| `{TempDir}` | Application temp directory path |
| `{PackageDir:name}` | Path to installed package (e.g. `{PackageDir:openssl}`) |
| `{PackageDir/Platform:name}` | Platform-specific path within package |
| `{ScriptDir:folder}` | Path to script in build output scripts dir |

## Time Span Parameter References
```
{ParameterName.TotalDays}
{ParameterName.TotalHours}
{ParameterName.TotalMinutes}
{ParameterName.TotalSeconds}
{ParameterName.TotalMilliseconds}
```
Example: `"-d{Duration.TotalSeconds}"` where `Duration: "00:05:00"` → `-d300`

## Inline Calculations
Uses Roslyn C# scripting. Format: `{calculate(<expression>)}`
```json
"ThreadCount": "{calculate({LogicalProcessorCount}/2)}",
"QueueDepth": "{calculate(512/{ThreadCount})}",
"IOEngine": "{calculate(\"{Platform}\".StartsWith(\"linux\") ? \"libaio\" : \"windowsaio\")}"
```

## MetricFilters Parameter
Controls which metrics are emitted. Applied in this order:
1. Verbosity filter (if present)
2. Exclusion filters (prefix `-`)
3. Inclusion filters (regex)

### Verbosity Levels
| Level | Meaning |
|-------|---------|
| 1 | Standard/Critical — most important for decisions |
| 2 | Detailed |
| 3–4 | Reserved |
| 5 | Verbose — all diagnostic/internal |

```json
"MetricFilters": "verbosity:1"                     // only critical metrics
"MetricFilters": "bandwidth,iops,_p99"             // include matching patterns
"MetricFilters": "-histogram*,-internal*"          // exclude patterns
"MetricFilters": "verbosity:2,-histogram*,iops"   // combine all types
```

## Multiple Profile Merging
```bash
VirtualClient.exe --profile=PERF-CPU-OPENSSL.json --profile=MONITORS-DEFAULT.json
```
Profiles are merged in order: Actions, Dependencies, Monitors from each are concatenated. First profile's name is used for telemetry.

## Complete Profile Example
```json
{
  "Description": "FIO I/O Stress Performance Workload",
  "Metadata": {
    "SupportedPlatforms": "linux-x64,linux-arm64",
    "RecommendedMinimumExecutionTime": "01:00:00"
  },
  "Parameters": {
    "FileSize": "496G",
    "Duration": "00:05:00"
  },
  "Dependencies": [
    {
      "Type": "DependencyPackageInstallation",
      "Parameters": {
        "Scenario": "InstallFioPackage",
        "BlobContainer": "packages",
        "BlobName": "fio.2.0.0.zip",
        "PackageName": "fio",
        "Extract": true
      }
    }
  ],
  "Actions": [
    {
      "Type": "FioExecutor",
      "Parameters": {
        "Scenario": "RandomWrite_4k_BlockSize",
        "PackageName": "fio",
        "CommandLine": "--name=fio_randwrite_{FileSize}_4k --size={FileSize} --rw=randwrite --bs=4k --iodepth=32 -d{Duration.TotalSeconds}",
        "FileSize": "$.Parameters.FileSize",
        "Duration": "$.Parameters.Duration",
        "MetricFilters": "verbosity:1"
      }
    }
  ],
  "Monitors": [
    {
      "Type": "PerfCounterMonitor",
      "Parameters": {
        "Scenario": "CaptureCounters",
        "MonitorFrequency": "00:01:00"
      }
    }
  ]
}
```
