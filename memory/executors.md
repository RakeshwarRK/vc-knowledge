# VirtualClient Executor Reference

## Executor Parameter Quick Reference

### FioExecutor
- **File:** `src/VirtualClient/VirtualClient.Actions/FIO/FioExecutor.cs`
- **Platforms:** linux-arm64, linux-x64, win-x64
- **Parameters:** CommandLine, CoolDownPeriod (00:00:00), DeleteTestFilesOnFinish (true), DiskFill (false), DiskFillSize, DiskFilter ("BiggestSize"), Engine, FileName ("fio-test.dat"), FileSize, ProcessModel ("SingleProcess"), TestFocus, MetricScenario (required)
- **Process Models:** SingleProcess (all disks one process), SingleProcessAggregated (job file), SingleProcessPerDisk (one per disk)
- **Parser:** FioMetricsParser
- **Elevated:** Yes (CreateElevatedProcess)
- **DiskFill state:** Persisted to StateManager under `FioExecutor.DiskFill`

### DiskSpdExecutor
- **File:** `src/VirtualClient/VirtualClient.Actions/DiskSpd/DiskSpdExecutor.cs`
- **Platforms:** win-arm64, win-x64
- **Parameters:** CommandLine (required), DeleteTestFilesOnFinish (true), DiskFill (false), DiskFillSize, DiskFilter ("BiggestSize"), FileName ("diskspd-test.dat"), FileSize, ProcessModel ("SingleProcess"), QueueDepth (16), ThreadCount (ProcessorCount), MetricScenario (required)
- **Parser:** DiskSpdMetricsParser
- **Note:** File sizes sanitized: `496GB` → `496G` (strips trailing B)

### OpenSslExecutor
- **File:** `src/VirtualClient/VirtualClient.Actions/OpenSSL/OpenSslExecutor.cs`
- **Platforms:** linux-arm64, linux-x64, win-x64
- **Parameters:** CommandArguments (required), PackageName
- **Parser:** OpenSslMetricsParser
- **Linux:** Auto-injects `-multi {ProcessorCount}` after `speed` keyword; sets `LD_LIBRARY_PATH={package}/lib64`
- **Windows:** Extends PATH with `{package}/vcruntime`; no `-multi` flag
- **Note:** Cipher name must be last in command string (avoid `-evp`)

### CoreMarkExecutor
- **File:** `src/VirtualClient/VirtualClient.Actions/CoreMark/CoreMarkExecutor.cs`
- **Platforms:** linux-arm64, linux-x64, win-arm64, win-x64
- **Parameters:** CompilerName (""), CompilerVersion (""), ThreadCount (LogicalProcessorCount)
- **Command:** `make XCFLAGS="-DMULTITHREAD={ThreadCount} -DUSE_PTHREAD" REBUILD=1 LFLAGS_END=-pthread`
- **Windows:** Runs via Cygwin bash
- **Parser:** CoreMarkMetricsParser (reads run1.log, run2.log files — not stdout)

### RedisServerExecutor
- **File:** `src/VirtualClient/VirtualClient.Actions/Redis/RedisServerExecutor.cs`
- **Base:** RedisExecutor → VirtualClientComponent
- **Parameters:** BindToCores (true), CommandLine (required), Port (required), IsTLSEnabled (false), ServerInstances (required), RedisResourcesPackageName
- **Affinity:** numactl CPU binding per instance when BindToCores=true
- **Server state:** Published to VC API as ServerState with PortDescription list
- **Success codes:** 0 and 137 (OOM kill accepted)
- **Retry:** 10 retries, linear backoff

### RedisBenchmarkClientExecutor
- **File:** `src/VirtualClient/VirtualClient.Actions/Redis/RedisBenchmarkClientExecutor.cs`
- **Base:** RedisExecutor → VirtualClientComponent
- **Parameters:** ClientInstances (1), CommandLine (required), Duration (60s), WarmUp (false)
- **Parser:** RedisBenchmarkMetricsParser
- **Polling timeout:** 40 minutes for server heartbeat
- **Lock:** Concurrent metric parsing serialized via lock(lockObject)

### NCPSExecutor (Client + Server)
- **Files:** `src/VirtualClient/VirtualClient.Actions/Network/NetworkingWorkload/NCPS/NCPS*.cs`
- **Parameters:** ThreadCount (16), TotalConnectionsToOpen (ThreadCount*100), MaxPendingRequests, ConnectionDuration (0), DataTransferMode ("1"), DisplayInterval (1), Port (9800), PortCount (ThreadCount), TestDuration (60s), WarmupTime (5s), DelayTime (0), ConfidenceLevel (99), AdditionalParams ("")
- **Parser:** NCPSMetricsParser
- **Client cmd:** `ncps -c {serverIP} -r {threads} -bp {port} -np {portCount} -N {conns} ...`
- **Server cmd:** `ncps -s -r {threads} -bp {port} -np {portCount} ...`
- **Retry:** 5 retries on `sockwiz` errors, 3s backoff multiplier

### LMbenchExecutor (includes LatMemRd)
- **File:** `src/VirtualClient/VirtualClient.Actions/LMbench/LMbenchExecutor.cs`
- **Platforms:** linux-arm64, linux-x64
- **Parameters:** Benchmarks, BinaryName (""), BinaryCommandLine (""), CompilerFlags, MemorySizeMB
- **Default mode:** `make build` → `make results` (with echo pipe for prompts) → `make summary`
- **LatMemRd mode:** When BinaryName="lat_mem_rd", runs `{package}/bin/{arch}/lat_mem_rd {args}`
- **Parser:** LMbenchMetricsParser (ParseResults or ParseLatencyMemReadResults)

### ExampleWorkloadExecutor (template)
- **File:** `src/VirtualClient/VirtualClient.Actions/Examples/ExampleWorkloadExecutor.cs`
- **Platforms:** all 4
- **Parameters:** CommandLine (required), ExampleParameter1 (optional), ExampleParameter2 (1234)
- **Purpose:** Canonical reference for writing new executors

### ExampleWorkloadWithAffinityExecutor (template)
- **File:** `src/VirtualClient/VirtualClient.Actions/Examples/ExampleWorkloadWithAffinityExecutor.cs`
- **Platforms:** all 4
- **Parameters:** BindToCores (false), CommandLine (required), CoreAffinity, TestName ("ExampleAffinityTest")
- **Linux affinity:** Pre-process numactl wrapping
- **Windows affinity:** Post-start Win32 affinity mask via ApplyAffinity()

### CreateResponseFile (dependency)
- **File:** `src/VirtualClient/VirtualClient.Dependencies/CreateResponseFile.cs`
- **Parameters:** FileName ("resource_access.rsp"), Option* (any param starting with "Option")
- **Purpose:** Generates .rsp response files for VC command line

## Cross-Cutting Patterns

### Component Lifecycle
1. `IsSupported()` — platform/arch check
2. `Validate()` — parameter validation
3. `InitializeAsync()` — package resolution, executable setup
4. `ExecuteAsync()` — workload + metrics
5. `CleanupAsync()` — process kill, file cleanup

### Parameter Access
```csharp
// Required: this.Parameters.GetValue<string>(nameof(X))
// Optional with default: this.Parameters.GetValue<bool>(nameof(X), true)
// Optional nullable: this.Parameters.TryGetValue(nameof(X), out IConvertible v)
// TimeSpan: this.Parameters.GetTimeSpanValue(nameof(X), TimeSpan.FromSeconds(60))
```

### Metric Logging
```csharp
this.MetadataContract.AddForScenario("ToolName", commandArgs, toolVersion: version);
this.MetadataContract.Apply(telemetryContext);
this.Logger.LogMetrics(toolName, scenarioName, startTime, endTime, metrics, categorization, args, Tags, ctx);
```

### Process Execution
```csharp
using (BackgroundOperations.BeginProfiling(this, cancellationToken))
{
    using (IProcessProxy process = ProcessManager.CreateProcess(exe, args))
    {
        this.CleanupTasks.Add(() => process.SafeKill(this.Logger));
        await process.StartAndWaitAsync(cancellationToken);
        process.ThrowIfWorkloadFailed();
    }
}
```

### Elevated vs Standard Process
- `ProcessManager.CreateProcess` — standard
- `ProcessManager.CreateElevatedProcess` — sudo/admin
- `ProcessManager.CreateElevatedProcessWithAffinity` — sudo + numactl

### CPU Affinity
- **Linux:** Pre-process wrapping with `numactl --physcpubind={cores}`
- **Windows:** Post-start `process.ApplyAffinity(WindowsProcessAffinityConfiguration)`

### Disk Workload Pattern (FIO, DiskSpd)
- DiskFill state persisted via StateManager key `"{TypeName}.DiskFill"`
- DiskFilter defaults to `"BiggestSize"`
- ProcessModel controls single vs per-disk strategy
- DiskWorkloadProcess wraps IProcessProxy + test files + categorization

### Client/Server Coordination (Redis, NCPS)
- Server saves state to VC API (ports + affinity)
- Client polls: heartbeat → online → state fetch → execute
- `SetServerOnline(bool)` / `PollForServerOnlineAsync`
