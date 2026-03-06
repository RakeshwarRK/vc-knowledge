# VC Memory — Telemetry & Data

## Data Categories
| Category | Description |
|----------|-------------|
| **Logs/Traces** | Structured runtime logging; errors with full callstacks |
| **Workload Metrics** | Measurements from benchmark output (IOPS, throughput, latency, etc.) |
| **System Events** | Important system changes (registry, .exe executions, config changes) |

## Privacy / Data Collection
- VC does **NOT** collect user benchmark data by default — no data sent to Microsoft
- Users must **explicitly** provide Azure connection strings to enable telemetry egress
- Microsoft can only infer usage from Azure Storage download traces (package downloads)
- Benchmark example outputs in source are for unit testing — scrubbed, not real results

## Local Output (always produced)
```
{app_dir}/logs/
├── metrics.csv         # structured metric output
├── traces.log          # runtime trace/debug log
└── {workload}/         # raw workload output per executor
    └── workload.log
```

## Scaled Output (user-configured via CLI flags)
| Sink | Purpose | Connection String Flag |
|------|---------|----------------------|
| Azure Event Hub | Streaming telemetry, big-data integration | `--event-hub` |
| Azure Data Explorer (Kusto) | Analysis and reporting | (via Event Hub) |
| Azure Blob Storage | File uploads (logs, profiler .bin files) | `--content-store` |

## Kusto Queries (in repo)
Location: `src/VirtualClient/Scripts/Kusto/`
- `WorkloadPerformance/Metrics.csl` — standard metrics query
- `WorkloadPerformance/Events.csl` — events query
- `WorkloadPerformance/Functions/GetVirtualClientMetricsSummary.csl`
- `WorkloadPerformance/Functions/GetVirtualClientMonitorsData.csl`
- `WorkloadPerformance/Functions/GetMetricsDescriptors.csl`
- `WorkloadDiagnostics/Traces.csl`
- `WorkloadDiagnostics/Functions/GetVirtualClientErrors.csl`

## Metadata Contract
Attached to every telemetry event automatically. Includes:
- System info: OS, architecture, platform, CPU/memory specs
- Experiment context: ExperimentId, AgentId, ClientRequestId
- Component context: component type, scenario name, profile name

## MetricFilters (profile parameter)
Controls which metrics are captured per-component. Applied in order:
1. `verbosity:N` — include metrics with verbosity ≤ N (1=critical, 5=all)
2. `-pattern` — exclusion regex (prefix with `-`)
3. `pattern` — inclusion regex

```json
"MetricFilters": "verbosity:2,-histogram*,bandwidth|iops|latency"
```

## Metric Verbosity Levels
| Level | Meaning |
|-------|---------|
| 1 | Standard/Critical — key decision metrics |
| 2 | Detailed |
| 3–4 | Reserved |
| 5 | Verbose — all internal/diagnostic |

## Metric Naming Conventions
- Typically: `{category}_{measurement}_{unit}` or `{test_name}/{metric_name}`
- Workload-specific; defined in each parser class
- P99 latency often: `read_latency_p99`, `write_latency_p99`

## Logging Extensions
- Location: `VirtualClient.Contracts` project
- Ensure consistent structured telemetry patterns across all components
- Use `EventContext` for structured log calls (not raw string formatting)

## Event Hub Table Mappings
- Location: `src/VirtualClient/Scripts/Kusto/EventHub/`
- `Tables.txt` — Kusto table definitions
- `TableMappings.txt` / `TableMappingsNew.txt` — JSON→Kusto column mappings

## Telemetry CLI Flags
| Flag | Purpose |
|------|---------|
| `--event-hub=<conn>` | Azure Event Hub connection string |
| `--content-store=<conn>` | Azure Blob for file uploads |
| `--metadata=<key>=<value>` | Attach custom metadata to all events |
| `--log-level=<level>` | Trace verbosity (default: Information) |
| `--log-to-file` | Write process stdout/stderr to log files |
