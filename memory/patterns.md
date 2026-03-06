# VC Memory — Design Patterns

## 1. Workload Executor Pattern (canonical)
```csharp
public class MyWorkloadExecutor : VirtualClientComponent
{
    public MyWorkloadExecutor(IServiceCollection dependencies, IDictionary<string, IConvertible> parameters = null)
        : base(dependencies, parameters) { }

    // Parameters from profile
    public string PackageName => this.Parameters.GetValue<string>(nameof(this.PackageName));
    public string CommandLine  => this.Parameters.GetValue<string>(nameof(this.CommandLine));

    protected override async Task InitializeAsync(EventContext telemetryContext, CancellationToken cancellationToken)
    {
        // Resolve package path; set up state
        IPackageManager packageManager = this.Dependencies.GetService<ISystemManagement>().PackageManager;
        DependencyPath package = await packageManager.GetPackageAsync(this.PackageName, cancellationToken);
        this.packagePath = package.Path;
    }

    protected override async Task ExecuteAsync(EventContext telemetryContext, CancellationToken cancellationToken)
    {
        ISystemManagement systemManagement = this.Dependencies.GetService<ISystemManagement>();
        ProcessManager processManager = systemManagement.ProcessManager;

        using IProcessProxy process = processManager.CreateProcess(
            "myworkload", this.CommandLine, workingDir);

        await process.StartAndWaitAsync(cancellationToken);
        process.ThrowIfErrored<WorkloadException>(ProcessProxy.DefaultSuccessCodes, ErrorReason.WorkloadFailed);

        await this.CaptureMetricsAsync(process, telemetryContext, cancellationToken);
    }
}
```

## 2. Parameter Access Pattern
```csharp
// Safe read with default
int threadCount = this.Parameters.GetValue<int>(nameof(this.ThreadCount), defaultValue: 4);

// Required parameter (throws if missing)
string packageName = this.Parameters.GetValue<string>(nameof(this.PackageName));

// Optional parameter
string logFolder = this.Parameters.GetValue<string>(nameof(this.LogFolderName), defaultValue: null);
```

## 3. Platform Guard Pattern
```csharp
[SupportedPlatforms("linux-x64,linux-arm64,win-x64,win-arm64")]
public class MyExecutor : VirtualClientComponent { ... }

// In code
if (this.Platform == PlatformID.Unix)
{
    // Linux-specific logic
}
else if (this.Platform == PlatformID.Win32NT)
{
    // Windows-specific logic
}
```

## 4. Metrics Capture Pattern
```csharp
private async Task CaptureMetricsAsync(IProcessProxy process, EventContext telemetryContext, CancellationToken cancellationToken)
{
    MyMetricsParser parser = new MyMetricsParser(process.StandardOutput.ToString());
    IList<Metric> metrics = parser.Parse();

    this.Logger.LogMetrics(
        toolName: "MyWorkload",
        scenarioName: this.Parameters.GetValue<string>("Scenario"),
        scenarioStartTime: startTime,
        scenarioEndTime: endTime,
        metrics: metrics,
        metricCategorization: "All",
        scenarioArguments: this.CommandLine,
        this.Tags,
        telemetryContext);
}
```

## 5. Retry Pattern (Polly)
```csharp
IAsyncPolicy retryPolicy = Policy
    .Handle<Exception>(exc => exc.IsTransient())
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(attempt * 2),
        onRetry: (exc, timeSpan, attempt, ctx) =>
        {
            this.Logger.LogWarning($"Transient failure. Retrying attempt {attempt}...");
        });

await retryPolicy.ExecuteAsync(cancellationToken => this.someOperation(cancellationToken), cancellationToken);
```

## 6. File System Pattern (always use IFileSystem)
```csharp
IFileSystem fileSystem = this.Dependencies.GetService<ISystemManagement>().FileSystem;

// Read file
string content = await fileSystem.File.ReadAllTextAsync(filePath, cancellationToken);

// Check exists
bool exists = fileSystem.File.Exists(filePath);

// Write file
await fileSystem.File.WriteAllTextAsync(filePath, content, cancellationToken);
```

## 7. Package Installation Pattern (in profile)
```json
{
  "Type": "DependencyPackageInstallation",
  "Parameters": {
    "Scenario": "InstallMyPackage",
    "BlobContainer": "packages",
    "BlobName": "mypackage.1.0.0.zip",
    "PackageName": "mypackage",
    "Extract": true
  }
}
```

## 8. State Persistence Pattern
```csharp
IStateManager stateManager = this.Dependencies.GetService<ISystemManagement>().StateManager;

// Save state
await stateManager.SaveStateAsync("my-state-key", new MyState { ... }, cancellationToken);

// Load state
MyState state = await stateManager.GetStateAsync<MyState>("my-state-key", cancellationToken);
```

## 9. Client/Server Synchronization Pattern
```csharp
// Client waits for server to be ready
if (this.IsInRole("Client"))
{
    await this.WaitForServerOnlineAsync(serverLayoutClient, cancellationToken);
    // Then send instructions to server
    await serverLayoutClient.SendInstructionsAsync(instructions, cancellationToken);
}

// Server polls for instructions
if (this.IsInRole("Server"))
{
    await this.WaitForInstructionsAsync(layoutClient, cancellationToken);
}
```

## 10. Cleanup Task Registration
```csharp
this.CleanupTasks.Add(() =>
{
    // Cleanup logic runs after ExecuteAsync completes (success or failure)
    this.KillProcesses(processName);
});
```

## 11. Controller/Agent Template Method Pattern (PR #576)
Base class handles SSH fan-out to N agents in parallel; subclasses implement per-target logic.
```csharp
// Base class (VirtualClientControllerComponent)
protected override async Task ExecuteAsync(EventContext telemetryContext, CancellationToken cancellationToken)
{
    IEnumerable<ISshClientProxy> sshTargets = this.TryGetTargetAgentClients();
    List<Task> agentTasks = new List<Task>();

    foreach (ISshClientProxy target in sshTargets)
    {
        // Fan-out: one Task.Run per SSH target
        agentTasks.Add(Task.Run(() => this.ExecuteAsync(target, telemetryContext, cancellationToken)));
    }

    await Task.WhenAll(agentTasks);
    // Console output serialized via SemaphoreSlim after all complete
}

// Subclass only implements per-target logic
protected abstract Task ExecuteAsync(ISshClientProxy sshTarget, EventContext telemetryContext, CancellationToken cancellationToken);
```

## 12. Parameter Passthrough for Remote Execution (PR #576)
When executing commands on remote agents, prevent local parameter resolution:
```csharp
public RemoteAgentExecutor(IServiceCollection dependencies, IDictionary<string, IConvertible> parameters = null)
    : base(dependencies, parameters)
{
    // CRITICAL: Prevents controller from resolving $.Parameters.* expressions
    // These must be resolved on the remote agent where system context exists
    this.ParametersEvaluated = true;
}
```

## 13. Cross-Process Package Synchronization (PR #576)
When multiple VC instances share a machine, use IsolatedPackageManager with kernel Mutex:
```csharp
// Mutex requires same-thread acquire/release — can't use await
// Wrap in Task.Run with synchronous .GetAwaiter().GetResult()
this.packageMutex.WaitOne();
try
{
    await Task.Run(() => this.innerManager.GetPackageAsync(name, ct).GetAwaiter().GetResult());
}
finally
{
    this.packageMutex.ReleaseMutex();
}
```

## 14. Automatic Behavior Inference (Bryan's principle)
Remove user knobs when the system can infer the right behavior:
```csharp
// --isolated flag REMOVED from CLI
// Instead: automatically set when SSH targets are present
if (this.TargetAgents?.Any() == true)
{
    this.Isolated = true;  // Forced, not optional
}
```

## 15. CLI Argument Rewriting for Remote Forwarding (PR #576)
Strip controller-specific args before forwarding command to remote agent:
```csharp
// "VirtualClient remote --ssh user@host;pw --profile X.json --timeout=120"
// becomes on remote: "VirtualClient --profile X.json --timeout=120"
string targetCommand = originalArgs
    .Where(arg => !arg.Contains("--ssh") && !arg.Contains("--agent-ssh"))
    .Where(arg => arg != "remote");
```

## Anti-Patterns (Never Do)
- `File.ReadAllText(path)` → use `IFileSystem.File.ReadAllText(path)`
- `Process.Start(...)` → use `ProcessManager.CreateProcess(...)`
- Sharing logic between Actions, Dependencies, Monitors directly
- Swallowing exceptions without logging or re-throwing
- Hard-coding platform paths (use `PlatformSpecifics` helpers)
- Resolving `$.Parameters.*` on controller side when executing remotely
- Using `await` with `Mutex` (thread affinity requirement)

## 16. Flatten Premature Base Classes (PR #549)
When a base class becomes a shared property bag that subclasses use differently, delete it:
```csharp
// BEFORE: DiskWorkloadExecutor base class shared by FIO + DiskSpd
// Properties like DiskFilter, ProcessModel, FileName lived in base
// But FIO needed Engine, DiskSpd had different semantics

// AFTER: Each tool owns its own properties directly
public class FioExecutor : VirtualClientComponent  // NOT : DiskWorkloadExecutor
{
    public string Engine { get; }      // FIO-specific
    public string DiskFilter { get; }  // Owns its own
    public string ProcessModel { get; }
}
```

## 17. Push Platform Logic to Profiles (PR #549)
Use `calculate()` expressions for platform-specific values instead of C# conditionals:
```json
// BEFORE: FioExecutor.GetIOEngine() static method with switch(platform)
// AFTER: Profile parameter with calculate expression
{
    "Engine": "{calculate(\"{Platform}\".StartsWith(\"linux\") ? \"libaio\" : \"windowsaio\")}"
}
```
This makes the IO engine configurable without code changes.

## 18. Separation of Concerns for Disk Lifecycle (PR #549)
Mount point creation, formatting, and workload execution should be independent steps:
```
// WRONG: FioExecutor.CreateMountPointsAsync() inside ExecuteAsync()
// RIGHT: MountDisks dependency runs as a separate profile step before FIO
```
The executor should find disks and use them, not manage their lifecycle.

## 19. Precise Test Fixtures (PR #549)
Test helpers should accept exact device paths instead of generating generic mocks:
```csharp
// BEFORE: fixture.CreateDisks(PlatformID.Unix, withVolume: true)
//   → Returns 4 generic disks with auto-generated paths

// AFTER: fixture.CreateDisk(1, PlatformID.Unix, os: false, "/dev/sdc", "/dev/sdc1", "/dev/sdc2")
//   → Precise control: disk path + partition paths match real topology
```

## 20. FIO Job Files for Multi-Disk Aggregation (PR #549)
When aggregating results across N disks in one process, use FIO's native job file:
```ini
# Dynamically generated job file
[fio_randwrite_496GB_4k_d32_th16_1]
filename=/home/user/mnt_dev_sdc1/fio-test.dat

[fio_randwrite_496GB_4k_d32_th16_2]
filename=/home/user/mnt_dev_sdd1/fio-test.dat
```
Key: strip `--name` from command line args when using job file (it overrides job names).

## 21. Profile as Universal Execution Model (PR #494)
Even ad-hoc commands flow through the profile pipeline for consistent telemetry:
```csharp
// "VirtualClient pwsh script.ps1" becomes:
this.Parameters["Command"] = fullCommand;
this.Parameters["Scenario"] = $"Execute-{commandName}";
this.Profiles = new[] { "EXECUTE-COMMAND.json" };
return base.ExecuteAsync(args, cancellationTokenSource);  // Same profile flow
```

## 22. Inherit and Specialize CLI Commands (PR #494)
New execution modes extend existing command classes:
```csharp
// ExecuteCommand extends ExecuteProfileCommand
// Can handle both bare commands AND profile execution
internal class ExecuteCommand : ExecuteProfileCommand
{
    public override Task<int> ExecuteAsync(...)
    {
        if (!String.IsNullOrWhiteSpace(this.Command))
            return this.ExecuteCommandAsync(...);  // New path
        else if (this.Profiles?.Any() == true)
            return this.ExecuteProfilesAsync(...);  // Existing path
    }
}
```

## 23. Backward-Compatible CLI Promotion (PR #494)
Make required options optional when adding new capabilities:
```csharp
// --profile became optional (was required)
// Root handler changed from RunProfileCommand to ExecuteCommand (superset)
// TreatUnmatchedTokensAsErrors = false on root command
// All existing CLI usage continues to work unchanged
```

## 24. Compose Related Clients Behind One Interface (PR #547)
When two library clients share connection info, wrap them in a single proxy:
```csharp
public class SshClientProxy : ISshClientProxy
{
    internal SshClient SessionClient { get; }   // For command execution
    internal ScpClient SessionScpClient { get; } // For file copy
    
    public async Task ConnectAsync(CancellationToken ct)
    {
        await this.SessionClient.ConnectAsync(ct);
        await this.SessionScpClient.ConnectAsync(ct);
    }
}
```

## 25. Use IFileSystem Abstractions in Infrastructure (PR #547)
File operations in infrastructure code use `IFileInfo`/`IDirectoryInfo` for testability:
```csharp
Task CopyFromAsync(string remoteFilePath, IFileInfo destination, CancellationToken ct);
Task CopyToAsync(IDirectoryInfo source, string remoteDirectoryPath, CancellationToken ct);
// NOT: Task CopyFrom(string remotePath, string localPath) — untestable
```

## 26. Fix Regex at the Foundation (PR #627)
Path detection regex must be precise — anchor, single char, handle both slash types:
```csharp
// BEFORE (fragile): Regex.IsMatch(path, "[A-Z]+:\\\\|^/")
//   Matched "ABC:\\", didn't handle forward slashes on Windows

// AFTER (correct): Regex.IsMatch(path.Trim(), @"^[A-Z]{1}:[\\\\/]|^\/")
//   Anchored, single drive letter, handles / and \ on Windows
```

## 27. Capture Metrics/Logs in Finally (PR #627)
Even if the workload throws, capture available metrics and logs:
```csharp
try
{
    await this.LogProcessDetailsAsync(process, telemetryContext, this.ToolName);
    process.ThrowIfWorkloadFailed();
}
finally
{
    await this.CaptureMetricsAsync(process, telemetryContext, cancellationToken);
    await this.CaptureLogsAsync(cancellationToken);
}
```

## 28. Fully-Qualified Path Check First (PR #627)
When resolving paths, check the most specific/certain case first:
```csharp
if (PlatformSpecifics.IsFullyQualifiedPath(this.ScriptPath))
    this.ExecutablePath = this.ScriptPath;           // Most certain
else if (!string.IsNullOrWhiteSpace(this.PackageName))
    // Resolve relative to package                   // Next most certain
else
    // Resolve relative to VC root                   // Fallback
```

## 29. Root vs User Mount Paths (PR #565)
Distinguish mount point location based on who is running:
```csharp
if (string.Equals(user, "root"))
{
    // /mnt_dev_sdc1
    mountPointPath = "/";
}
else
{
    // /home/user/mnt_dev_sdc1
    mountPointPath = $"/home/{user}";
}
```

## 30. CancellationToken.None for Must-Complete Operations (PR #565)
Once a mount or permission change starts, it must finish — don't pass a cancellable token:
```csharp
await this.diskManager.CreateMountPointAsync(volume, newMountPoint, CancellationToken.None);
await this.systemManager.SetFullPermissionsAsync(newMountPoint, ..., CancellationToken.None, ...);
```

## 31. Extension Methods for Cross-Cutting OS Operations (PR #565)
chmod/chown logic as a reusable extension, not inline in one component:
```csharp
public static async Task SetFullPermissionsAsync(
    this ISystemManagement systemManagement, string directoryPath, PlatformID platform,
    EventContext telemetryContext, CancellationToken cancellationToken, string owner = null, ILogger logger = null)
```

## 32. Keep Packages Read-Only (PR #565)
Write results to temp directory, not the package directory:
```csharp
// BEFORE: this.Combine(this.Prime95Package.Path, "results.txt")
// AFTER:  this.Combine(this.GetTempPath(), FileContext.GetFileName("results.txt", DateTime.UtcNow))
```

## 33. Delete Dead Code Without Ceremony (PR #565)
Remove unused constants, dictionaries, arrays with no backward-compat shims:
```csharp
// 18 unused string constants in UnixDiskManager — deleted
// Static SystemDefinedMountPoints dictionary — deleted
// SysbenchExecutor.SelectWorkloads array — deleted
// No re-exports, no "// removed" comments, no deprecation period
```

## 34. Dual-Scope Metadata Registry (PR #158)
Metadata at two scopes — persisted (app lifetime) and instance (component lifetime):
```csharp
// Persisted: lives across entire VC execution
MetadataContract.Persist(new Dictionary<string, object> { ["os"] = "linux" }, MetadataContractCategory.Host);

// Instance: per-component, overrides persisted on conflict
this.MetadataContract.Add(new Dictionary<string, object> { ["tool"] = "fio" }, MetadataContractCategory.Scenario);

// Apply: instance wins over persisted
this.MetadataContract.Apply(telemetryContext);
```
Categories: Default, Dependencies, Host, Runtime, Scenario.

## 35. Best-Effort Hardware Discovery (PR #158)
System discovery must never crash the workload:
```csharp
try { memoryChips = await systemManagement.GetMemoryChipsAsync(ct); }
catch { logger.LogWarning("Memory discovery failed"); }
// Continue with empty/partial metadata — never block the benchmark
```

## 36. Platform-Specific Monitor Pattern (PR #486)
Same interface, different acquisition strategies per platform:
```csharp
// Windows: push-model via EventLogWatcher callback + BlockingCollection queue
eventWatcher.EventRecordWritten += this.EventWritten;

// Linux: pull-model via journalctl polling with --since checkpoint
journalctl --since "{lastCheckpoint}" --output json
```
Both cap at MaxEventsPerTelemetryCapture=50 per batch.

## 37. Fail-Fast as Exception Filter Widening (PR #163)
Don't add new error handling — widen the existing catch filter:
```csharp
// The catch clause already existed for terminal errors (>=500)
// --fail-fast simply widens it to include ALL errors
catch (VirtualClientException exc) when ((int)exc.Reason >= 500 || this.FailFast)
{
    throw;  // Propagate instead of logging and continuing
}
```
Applies to Actions only. Dependencies always fail-fast. Monitors never fail-fast.

## 38. Avoid Shared Mutable State in Recursive Component Creation (PR #503)
When building component hierarchies, assign to each component immediately:
```csharp
// WRONG: accumulate in shared variable, gets overwritten by next sibling
extensions.AddRange(componentDescription.Extensions, withReplace: false);

// RIGHT: assign directly to the component being created
component.Extensions.AddRange(componentDescription.Extensions, withReplace: false);

// Pass component's own resolved extensions to children, not the shared variable
CreateComponent(..., component.Extensions, ...);
```

## 39. SafeKill Inside Using Block, Not in CleanupTasks (PR #618)
Process ID becomes 0 after dispose — `kill -9 0` kills everything:
```csharp
// WRONG: CleanupTasks.Add(() => processManager.SafeKill(process));
// After dispose, process.Id == 0 → kill -9 0 kills VC itself!

// RIGHT: SafeKill inside the using block's finally
using (IProcessProxy process = ...)
{
    try { await process.StartAndWaitAsync(ct); }
    finally
    {
        process.Close();
        process.SafeKill(this.Logger, TimeSpan.FromSeconds(30));
    }
}
```

## 40. Create Core Directories at Startup (PR #631)
Race condition prevention — ensure logs/packages/temp exist before any component runs:
```csharp
// In CommandBase.InitializeDependencies(), before any profile execution
this.CreateCoreDirectories(platformSpecifics, fileSystem);
// Prevents: "directory not found" when first workload tries to write logs
```

## 41. ParallelLoopExecution vs ParallelExecution (Yang, PR #429)
Two parallel execution models:
```csharp
// ParallelExecution: run all children once in parallel
// ParallelLoopExecution: run all children in independent loops until timeout
protected override Task ExecuteAsync(...)
{
    foreach (var component in this)
        componentTasks.Add(ExecuteComponentLoopAsync(component, ...));
    return Task.WhenAll(componentTasks);
}
// Each component restarts after completion; first to timeout breaks
```

## 42. Original Verbosity System (Yang, PR #412)
Yang's original 0/1/2 verbosity with switch-case filtering:
```csharp
// This was later replaced by RakeshwarK's #636 (1/2/5 scale with regex)
switch (verbosityFilter.ToLower())
{
    case "verbosity:0": filteredMetrics = m => m.Verbosity <= 0; break;
    case "verbosity:1": filteredMetrics = m => m.Verbosity <= 1; break;
    case "verbosity:2": filteredMetrics = m => m.Verbosity <= 2; break;
    default: return empty;
}
```
Lesson: Yang built the foundation; #636 generalized it with int.TryParse and regex.

## 43. One Executor Per Toolset, Parameterized by Mode (Bryan review, PR #55)
Don't create separate executor classes per diagnostic type — use one executor with a mode parameter:
```csharp
// WRONG: DCGMIDiagDefaultExecutor, DCGMIDiagDiscoveryExecutor, DCGMIDiagFieldGroupExecutor
// RIGHT: One DCGMIDiagExecutor with DiagnosticType parameter
{ "Type": "DCGMIDiagExecutor", "Parameters": { "DiagnosticType": "Default" } }
{ "Type": "DCGMIDiagExecutor", "Parameters": { "DiagnosticType": "Discovery" } }
```
Similarly consolidate parsers unless parsing logic is distinctively different.

## 44. Principle of Least Privilege for Properties (Bryan review, PRs #46, #65)
No public setters on parameters. Expose via "public new" in test subclass:
```csharp
// In production code: read-only parameter
public string ClientScriptPath => this.Parameters.GetValue<string>(...);

// In test helper: expose for testing without public setter
public class TestExecutor : MyExecutor {
    public new string ClientScriptPath => base.ClientScriptPath;
}
```

## 45. Naming Conventions — Bryan's Rules (multiple PRs)
- Profile naming: `PERF-SQL-POSTGRESQL.json` (not PERF-POSTGRESQL), `PERF-CPU-LINPACK.json` (not PERF-CPU-HPL)
- Monitor naming: `MONITORS-GPU-NVIDIA.json` (category-hardware-vendor)
- Parameter names: spell out (`ProblemSize` not `N`, `PartitionBlockingFactor` not `NB`)
- Don't repeat class name in property: `Version` not `HPLVersion` in `HPLinpackExecutor`
- Scenario naming: `InstallMSMPIPackage`, `Diagnostics` (consistent verbs)
- Package naming: `msmpi.10.1.2.zip` (version separate from name)
- Execution components: `SequentialExecution` not `LoopExecution` (match existing `ParallelExecution`)

## 46. Precise Command Validation in Tests (Bryan review, PR #65)
Remove from expected commands list sequentially, assert empty at end:
```csharp
// WRONG: Assert.IsTrue(executedCommands.Contains(expected))
// RIGHT: Remove from expectedCommands[0] in OnCreateProcess, Assert.IsEmpty at end
// This validates both the commands AND their exact order
```

## 47. State Tracking for System Changes (Bryan review, PR #168)
Use state manager to track irreversible system changes (installations):
```csharp
// After installing: save state
await this.StateManager.SaveStateAsync("MsmpiInstallation", new { Installed = true }, ct);
// On next run: check state, skip if already done
// Enables clean reset workflow
```

## 48. IsSupported for Platform No-Ops (Bryan review, PR #168)
Windows-only dependencies should no-op on Linux via IsSupported:
```csharp
protected override bool IsSupported() => this.Platform == PlatformID.Win32NT;
// Not: throw NotSupportedException in ExecuteAsync
```

## 49. Don't Expose Untested Versions as Profile Parameters (Bryan review, PR #55)
Remove version overrides from global profile parameters:
```
// WRONG: Global param "CudaVersion" lets users pick any version
// RIGHT: Pin version in dependency step; users create custom profiles for other versions
// Reason: CRC team can't own SLA for untested version combinations
```

## 50. Pluggable Authentication Pattern (Yang, PR #309)
Layered auth with fallback: SAS → connection string → managed identity → certificate:
```csharp
// OptionFactory parses: "EndpointUrl=...;;;TokenType=ManagedIdentity"
// CertificateManager: thumbprint lookup → issuer/subject → chain validation
// BlobManager accepts TokenCredential abstraction from Azure.Core
```

## 51. Role-Aware API Port Configuration (Bryan, PR #33)
CLI supports both single port and role-specific mapping:
```bash
# Single port for all roles
--api-port 4500

# Role-specific ports
--api-port 4500/Client,4501/Server
```
Centralized in ApiClientManager with default 4500.

## 52. Group Reporting for Multi-Job FIO (Bryan, PR #28)
Always use `--group_reporting` when FIO runs multiple jobs:
```
# Without: each job reports separately, including zero values for unused metrics
# With: results aggregated across all jobs in a single report
--group_reporting --output-format=json
```

## Bryan's Review Meta-Patterns (synthesized from 50+ review comments)
1. **Naming is architecture** — Names define user mental models; get them right first
2. **Consolidate similar classes** — Cost of abstraction > benefit when logic is similar
3. **Principle of least privilege** — No public setters; expose minimum access surface
4. **Test precision** — Assert order AND content, not just containment
5. **Don't own untested paths** — Pin to validated versions; let users create custom profiles
6. **Dependencies at bottom** — Profile section ordering is a convention, not optional
7. **ConfigureAwait(false) is dead** — .NET removed the need; delete it everywhere
8. **State for idempotency** — Track installations/changes to enable clean resets
9. **Consistency is non-negotiable** — Usings inside namespace, license headers, tab alignment

## 53. Declarative Platform Attributes (Yang, PR #360)
Replace imperative platform checks with `[SupportedPlatforms]` attribute:
```csharp
// OLD (removed 876 lines of boilerplate):
if (this.Platform != PlatformID.Unix) { return; }

// NEW: attribute on class, framework enforces
[SupportedPlatforms("linux-x64,linux-arm64")]
public class MyExecutor : VirtualClientComponent

// Test: Assert.IsFalse(VirtualClientComponent.IsSupported(executor));
// Uses .NET RIDs as canonical platform identity
```

## 54. Centralized Version Management in Module.props (Bryan review, PR #408)
ALL NuGet package versions must live in `Module.props`, never scattered in `.csproj` files:
```xml
<!-- Module.props — single source of truth -->
<PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
<!-- Bryan: "Versions in Module.props to be consistent with our conventions" -->
```
Use `FrameworkReference` for ASP.NET instead of individual NuGet packages.

## 55. Safe CLI Defaults for v2 (Yang, PRs #498/#500)
Simplest invocation is `--profile=X.json`. Defaults:
- `--iterations=1` (run once, not forever)
- `--packages` defaults to public blob store
- No implicit `MONITORS-DEFAULT.json` merge
- `--timeout=never` or `--timeout=-1` for infinite (not `--iterations=-1`)
```
// Principle: safe defaults for new users, opt-in for risky behavior
// Principle: keep options semantically clean (iterations=count, timeout=duration)
```

## 56. Command-Line-Aware Parsing (Yang, PR #410)
Parser inspects the original command line to avoid emitting zero-valued metrics:
```csharp
// DiskSpd: -w100 means write-only → skip read parsing entirely
public enum ReadWriteMode { ReadWrite, ReadOnly, WriteOnly }
// Eliminates polluted telemetry (all-zero read metrics during write-only test)
// Also: enum over two booleans — makes state space explicit
```

## 57. Delay-Before-Process in File Monitors (Yang, PR #406)
File-watching monitors must delay BEFORE processing, not after:
```csharp
// WRONG: process immediately, then delay → race with file writers
// RIGHT: delay first, then process → files have time to finish writing
await Task.Delay(this.ProcessingIntervalWaitTime, cancellationToken);
await this.ProcessFileUploadsAsync(...);
```

## 58. Never Poll HasExited — Use WaitForExitAsync (Yang, PR #50)
.NET `Process.HasExited` becomes true BEFORE output buffers flush:
```csharp
// WRONG: while (!process.HasExited) { await Task.Delay(10); } — loses trailing output
// RIGHT: await process.WaitForExitAsync(cancellationToken);
// ALSO: subscribe OutputDataReceived BEFORE Process.Start(), not after
```

## 59. FileMode.Create Not OpenOrCreate for Downloads (Yang, PR #226)
Interrupted downloads + retry with smaller file leaves stale trailing bytes:
```csharp
// WRONG: new FileStream(path, FileMode.OpenOrCreate) — old bytes remain at end
// RIGHT: new FileStream(path, FileMode.Create) — truncates before writing
// ALSO: isolate downloaded profiles in "downloaded/" subdirectory
```

## 60. Override Destructive Defaults in Raw-Resource Subclasses (Yang+Bryan, PR #48)
Base class cleanup can destroy raw disks/mount points:
```csharp
// FioExecutor base: DeleteTestFilesOnFinish = true (fine for file-based)
// FioDiscoveryExecutor (raw disk): MUST set DeleteTestFilesOnFinish = false in constructor
// Bryan: "Put this definition in the constructor for Discovery and Multi-Throughput as well"
// ALSO: guard ProcessorCount / 2 against zero on single-CPU VMs
```

## 61. Every Catch Block in a Polling Loop Must Include Delay (Yang, PR #66)
Without delay, failures cause tight spin loops:
```csharp
while (!cancellationToken.IsCancellationRequested)
{
    try { /* poll operation */ }
    catch (Exception exc)
    {
        this.Logger.LogError(exc);
        // CRITICAL: delay even on error, or the loop spins at 100% CPU
        await Task.Delay(this.MonitorFrequency, cancellationToken);
    }
}
```

## 62. Never Remove CLI Aliases Without Deprecation (Yang, PR #313)
Even typo'd flags become contracts once shipped:
```csharp
// "--eventbHubConnectionString" has a typo, but users depend on it
// Must keep as alias alongside corrected "--eventHubConnectionString"
// Breaking CLI contracts silently = production incident
```

## 63. No Logic in Property Getters/Setters (Bryan review, PR #271)
Move calculations to `InitializeAsync()`:
```csharp
// WRONG: public int TableCount => this.SystemManager.GetMemoryKB() / 1024;
// RIGHT: calculated in InitializeAsync, stored in field
// For shared logic across components: static helpers on config class
public static int GetTableCount(ISystemManagement sys, string scenario) { ... }
```

## 64. Lock Down Profile Parameters to Intended Use (Bryan review, PR #303)
Purpose-built profiles should not expose parameters that let users deviate:
```csharp
// WRONG: Global param "Benchmark" on a TPCC-only profile — users might override to OLTP
// RIGHT: Remove from globals; users who need different configs create their own profiles
// ALSO: throw on unsupported input, don't silently fall through
else { throw new DependencyException(..., ErrorReason.NotSupported); }
```

## 65. MetricScenario as Override Pattern (Bryan review, PR #246)
Separate metric naming from execution scenario:
```csharp
scenarioName: this.MetricScenario ?? this.Scenario
// Profile can define both Scenario (execution) and MetricScenario (telemetry naming)
// Enables different metric labels without changing execution behavior
```

## 66. Scenario Names Enable --scenarios Filter (Bryan review, PR #562)
Scenario names are functional, not decorative:
```bash
# Users can selectively run: --scenarios=RandomRead_4k_BlockSize,RandomRead_8k_BlockSize
# WRONG: collapsing multiple discovery steps into one breaks this capability
# Bryan: "Fix bugs in code, don't restructure profiles to work around them"
```

## 67. Documentation Describes Intent Not Implementation (Bryan review, PR #300)
```
// WRONG: "This profile exposes the FioExecutor class with template file support"
// RIGHT: "This profile runs an OLTP database I/O workflow"
// ALSO: don't use class names in profile docs (they become stale)
// Supply concrete examples: "weight ratios for random read + write: 70/30"
```

## 68. Parameter Isolation Across Components (Yang review, PR #133)
Never merge global parameters into child components unconditionally:
```csharp
// WRONG: child.Parameters = MergeWith(globalParams) — Workload A's "Command" overrides B
// RIGHT: each component's parameter dictionary is independent
// Caller decides conflict resolution, not the framework
```

## 69. Don't Create New Types When Existing Types Suffice (Yang review, PR #536)
```csharp
// WRONG: new JsonMetric class for parsing script output
// RIGHT: use the canonical Metric class + JsonConvert
// Yang: "Standardize serialization on the same object"
// Parallel types for same concept = maintenance burden
```

## 70. Parameter Cohesion in Profiles (Bryan review, PR #491)
Logically coupled parameters must share required/optional status:
```json
// WRONG: "PerformanceLibrary" optional + "PerformanceLibraryVersion" required
// RIGHT: both required or both optional — asymmetric optionality confuses users
```

## 71. API Design Through Method Overloads (Bryan review, PR #647)
Prefer clean overloads over different method names:
```csharp
// RIGHT:
AssertCommandsExecuted(params string[] expectedCommands);
AssertCommandsExecuted(bool exactOrder, params string[] expectedCommands);
// ALSO: parameter name hides implementation ("expectedCommands" not "expectedCommandPatterns")
```

## 72. Canonical Workload File Structure (Yang, PRs #36/#64)
Every new workload follows this structure:
```
src/.../VirtualClient.Actions/<Workload>/
    <Workload>Executor.cs
    <Workload>MetricsParser.cs
VirtualClient.Actions.UnitTests/<Workload>/
    <Workload>ExecutorTests.cs
    <Workload>MetricsParserTests.cs
VirtualClient.Actions.UnitTests/Examples/<Workload>/
    <ExampleOutput>.txt          # Real sample output
VirtualClient.Actions.FunctionalTests/
    <Workload>ProfileTests.cs
VirtualClient.Main/profiles/
    PERF-<CATEGORY>-<WORKLOAD>.json
website/docs/workloads/<workload>/
    <workload>.md, <workload>-profiles.md, <workload>-metrics.md
```

## 73. Remove Azure-isms from Generic Components (Yang review, PR #135)
Hardware-targeting components must be cloud-agnostic:
```csharp
// WRONG: VmSeries: "nvv4" — ties component to Azure
// RIGHT: GpuModel: "mi25" — works on any machine with that GPU
// Principle: same driver/workload should work on Azure, AWS, or bare metal
```

## 74. Parse by Column Name, Not Position (Yang review, PR #280)
Tool output columns may reorder between versions:
```csharp
// WRONG: value = row[3] — positional, breaks on column reorder
// RIGHT: value = row["utilization.gpu"] — header-based, survives schema changes
// Especially critical for nvidia-smi, lshw, lspci output
```

## 75. Rename/Delete Output Files After Parsing (Yang review, PR #175)
Prevent double-parsing stale results on re-runs:
```csharp
// After parsing results.xml → rename to results.xml.parsed
// Without cleanup: re-run with exitcode=0 but no new output silently reports old results
```

## 76. Keep Executor Execution Linear (Yang review, PRs #192/#197)
Don't parallelize side operations inside executors:
```csharp
// WRONG: Task.Run(() => UploadFile(...)) inside ExecuteAsync
// RIGHT: collect artifacts into list, process sequentially after workload completes
// Parallelism inside executors creates race conditions and complicates error handling
```

## 77. Put Queryable Labels in Native Telemetry Columns (Yang review, PR #197)
Sub-scenario labels must be in Kusto-queryable columns:
```csharp
// WRONG: metricName = "Blender_monster_render_time" — label buried in name
// RIGHT: scenario = "monster", metricName = "render_time" — filterable in Kusto
```

## 78. Workload-Specific Constants Stay in Workload Class (Yang review, PR #218)
Don't pollute shared contracts with workload-specific values:
```csharp
// WRONG: adding JavaVersion = "17" to VirtualClient.Contracts
// RIGHT: private const string DefaultJavaVersion = "17" in HadoopExecutor
// Shared projects contain only truly cross-cutting constants
```

## 79. Package Manager Belongs in Profile, Not Code (Yang review, PR #461)
Use declarative profile dependencies for OS packages:
```json
// WRONG: hardcoded "dnf install iptables" in C# code
// RIGHT: LinuxPackageInstallation dependency in profile JSON with apt/dnf sections
// Keeps executor OS-agnostic; profile declares requirements
```



## Patterns 81-88: Session 4 Drill Discoveries

### 81. DataTable Parser Pattern
For key:value or tabular output, use `DataTableExtensions.ConvertToDataTable()` + `DataTable.GetMetrics()` rather than individual regex extractions. Filter lines in `Preprocess()` to only delimiter-matching lines, then parse as table. Post-hoc patch units and relativity on individual metrics.

### 82. Preserve Tool Metric Names
Don't rename metrics — use the exact names from the tool's output (e.g., "Iterations/Sec" not "CoreMark Score"). VC convention preserves the tool's original naming for traceability.

### 83. Metric Verbosity Assignment
Primary performance metrics get `Verbosity = 1`, secondary/config metrics get `Verbosity = 5`. This controls which metrics appear at different logging levels. Never omit verbosity.

### 84. Conservative Metric Relativity
Only set `HigherIsBetter`/`LowerIsBetter` for true performance metrics. Configuration values (iterations count, thread count) get `MetricRelativity.Undefined`. Don't over-specify.

### 85. Scenario Names Are Metric-Focused
Scenario names describe WHAT is measured, never the action verb. Use "Gflops" not "ExecuteHpcg", "SHA256" not "RunOpenSSL". For IO: "RandomWrite_4k_BlockSize". For compression: "7zLZMAUltraMode".

### 86. Installation Archetypes
Six patterns for workload installation:
1. Download & run: DependencyPackageInstallation (Geekbench, OpenSSL)
2. Build from source: CompilerInstallation + GitRepoClone (CoreMark, HPCG)
3. Windows cross-platform: Chocolatey + Compiler (Lapack)
4. Client-server: DependencyPackage + ApiServer (Redis, Memcached)
5. IO workload: FormatDisks + MountDisks + DependencyPackage (FIO)
6. GPU/Container: FormatDisks + Docker + NvidiaCuda (MLPerf)

### 87. MinimumExecutionInterval Is a Gap Timer
MinimumExecutionInterval (profile-level) is the minimum gap between re-executions, NOT the benchmark duration. Usually small (00:01:00). Don't confuse with Duration (component-level, controls how long to run).

### 88. Unit Test Coverage Pattern
VC tests always cover 4 areas: happy path execution, CancellationToken respect, error propagation (wraps in WorkloadException), and unsupported component skipping (IsSupported=false). Uses delegate-based TestComponent inner classes, not mocking frameworks.

## Patterns 89-98: Session 4b Deep Drills

### 89. Parser Strategy Matches Output Format
- Tabular text (key:value) → DataTable + GetMetrics (CoreMark, LMbench)
- XML output → XmlSerializer + POCO classes (NTttcp)
- JSON output → JsonConvert.DeserializeObject (FIO)
- Don't force one parsing pattern on all tools.

### 90. Client-Server Profile: CommandLine vs Typed Parameters
Client-server executors (Redis, Memtier, NTttcp) use raw CommandLine strings, not decomposed typed parameters. The executor appends server IP/port but benchmark flags are a single string. Simpler workloads (CoreMark, Geekbench) may use typed parameters.

### 91. Redis Benchmark Key Space Design
Warmup scenarios pre-populate distinct key pools (--key-prefix sm/med/lg) with --ratio 1:0 (write-only) and --requests=allkeys. Benchmark scenarios then use --ratio 1:1 (50/50 R/W) with --key-pattern R:R (random access). Each data size needs its own warmup.

### 92. IO Profile Triple Dependency
IO workloads always need: (1) LinuxPackageInstallation for the tool, (2) FormatDisks, (3) MountDisks with DiskFilter. Windows uses DependencyPackageInstallation scoped with SupportedPlatforms: "win-x64". Don't include FormatDisks/MountDisks for in-memory stores (Redis).

### 93. FIO Data Integrity Scenarios
IO profiles include DataIntegrity scenarios alongside performance scenarios: --verify=sha256 --do_verify=1, smaller file size (4G vs 496G), --numjobs=1 --iodepth=1, subset of block sizes, no --runtime (run to completion).

### 94. Source Compilation Pattern
Some workloads compile from source: LinuxPackageInstallation (build deps) → GitRepoClone/WgetPackageInstallation (source) → ExecuteCommand (make/configure). Redis, CoreMark, HPCG use this. Blob download is for pre-built binaries only.

### 95. ServerInstances Scaling
Redis/Memcached use ServerInstances={LogicalCoreCount} because they're single-threaded per process. One instance per core saturates the server. This is domain knowledge that affects profile design.

### 96. Yang's Operational Review Checklist
Yang reviews new workloads for: MIT headers on all .cs files, .vcpkg file in source, blob store upload, sensitive data redaction in test files, platform-specific line ending parsing, complete metric extraction, example arguments in comments.

### 97. Abstraction Level Decision Rule
"Where does this concern naturally live?" determines abstraction level:
- ProcessorAffinity → process creation layer (cross-cutting, goes in Core)
- CoolDownPeriod → individual executor logic (workload-specific, stays per-component)
- Don't ask "how many consumers?" — ask "where does this naturally belong?"

### 98. Nullable vs Default Parameters
Use GetValue<T>(name, default) for parameters where absence means "use this default."
Use TryGetValue + nullable for parameters where absence has DIFFERENT meaning than any value (e.g., RecordCount null = "auto-calculate based on memory" vs RecordCount=1000 = "exactly 1000").

## Patterns 99-100: Execution Flow & Profile Contract

### 99. Profile Action Error Handling Flow
ProfileExecutor runs actions in a loop until timeout. Error handling per iteration:
- ErrorReason ≥ 500 → terminal, throws immediately (ProfileNotFound, PlatformNotSupported, DependencyNotFound)
- ErrorReason 400-499 → serious but retryable, swallowed and retried next iteration (InvalidResults, ApiStatePollingTimeout)
- ErrorReason < 400 → transient, swallowed (WorkloadFailed, MonitorErrors)
- FailFast=true overrides all — any error throws immediately
Dependencies run sequential/one-time. Actions loop forever. Monitors run parallel background.

### 100. JSONPath Parameter Flow Is Non-Negotiable
Every tunable value in a profile MUST be a top-level `$.Parameters.*` entry with JSONPath references (`$.Parameters.Duration`) in component Parameters. This is the profile contract — users customize behavior only through top-level Parameters. Hardcoding values in component Parameters that users might want to change violates this contract. Bryan blocks PRs that bypass this.

## Patterns 101-105: Memcached Deep Drill (Session 5)

### 101. Multi-threaded vs Single-threaded Server Scaling
- Redis: single-threaded → ServerInstances={LogicalCoreCount} (many processes)
- Memcached: multi-threaded (-t N) → ONE process, no ServerInstances needed
- Don't blindly copy scaling patterns between workloads — understand the server's threading model first.

### 102. CommandLine Placeholder Syntax (Two Systems)
Two distinct substitution systems in profiles:
- `$.Parameters.X` — JSONPath reference for parameter VALUES (in component Parameters block)
- `{X}` — ApplyParameters string substitution (WITHIN CommandLine strings)
- CommandLine uses `{Duration}` and `{Port}`, NOT `$.Parameters.Duration`

### 103. Scenario Naming Encodes Full Configuration
For benchmark matrix scenarios: `tool_threads_clients_datasize_ratio`
Example: `memtier_8t_16c_32b_r1:1` — encodes threads, clients, data size, and ratio.
NOT just the data size ("Memtier_1kb_String" loses thread/client dimensions).

### 104. WarmUp Flag Controls Metric Emission
Warmup scenarios set `WarmUp: true` to suppress performance metrics for pre-population runs. Without it, warmup data pollutes real benchmark results. Warmup scenarios also use ClientInstances=1 (only need one writer to populate).

### 105. ApiServer Dependency for Client-Server
Client-server workloads ALWAYS need `ApiServer` in Dependencies for cross-machine coordination. Enables heartbeat polling, state synchronization, and IP/port discovery between client and server VMs.

## Patterns 106-111: Redis/Memcached Comparison & Review Calibration (Session 5): Redis/Memcached Comparison & Review Calibration (Session 5)

### 106. TLS Cross-Cutting Propagation
When TLS is a dimension, IsTLSEnabled must appear on EVERY component: server executor, ALL client executors (including warmups), the compile step, and TLS resources dependency. Missing on one component = silent plaintext fallback.

### 107. Key Space Sizing Reflects Server Memory Model
Redis key-maximum values are smaller than Memcached for same data sizes because Redis has higher per-key overhead (encoding metadata, expiry pointers). Key space must fit in memory — exceeding causes eviction/OOM.

### 108. WgetPackageInstallation vs GitRepoClone
- WgetPackageInstallation: release tarballs (pinned version, specific URL) — Redis server uses this
- GitRepoClone: HEAD/branch builds (version pinned via git checkout in ExecuteCommand) — memtier uses this

### 109. Bryan Reviews Mechanical Before Semantic
For naming issues, Bryan checks file-name-matches-class-name BEFORE semantic naming concerns. Start with the most basic checks first. (From PR #641: file was CreateResourceFile.cs but class was CreateResponseFile.)

### 110. Cross-Cutting Changes Require Completeness Evidence
Bryan demands explicit evidence that ALL affected files were found. PR description must state: "Searched all profiles for X; only these N profiles match." Missing even one profile = PR blocked.

### 111. Fault Tolerance Over Precision (Yang's Principle)
When a dependency lookup can fail (env var missing, tool not in PATH), provide a sensible default rather than throwing. Yang on PR #458: "assume compiler version is 10 (use gcc>10 workaround)" — pragmatic resilience over precise detection.

## Patterns 112-120: Network Client-Server Deep Dive (Session 5)

### 112. Two Client-Server Executor Architectures
1. **Direct reference** (Redis/Memcached): Profile lists separate Types for server and client, framework routes by Role.
2. **Orchestrator** (SockPerf, NTttcp, CPS): Single Type, executor checks own role, creates sub-executors via `CreateWorkloadClient()`/`CreateWorkloadServer()` virtual methods.

### 113. 6-Step Client-Server Sync Protocol
Canonical client flow: (1) PollForHeartbeatAsync, (2) PollForServerOnlineAsync, (3) ResetServerAsync, (4) SendInstructionsAsync(StartExecution), (5) PollForExpectedStateAsync(ExecutionStarted), (6) ExecuteClientAsync, (finally) ResetServerAsync + PollForStateDeletedAsync.

### 114. Instructions-Based Server Communication
Network2 executors use `Instructions` objects: `ClientServerReset` (stop workloads) and `ClientServerStartExecution` (start specified workload). Instructions carry the full Parameters dictionary.

### 115. Server State Machine via Local API
Server publishes state: Ready → ExecutionStarted (when process detected running) → Deleted (on reset). Client polls for these transitions. State is a typed object stored through the self-hosted API.

### 116. Background Server with Explicit Kill
Network servers run indefinitely via `WaitAsync(cancellationToken)`. Must `SafeKill` in finally block. Exit code 137 (SIGKILL) treated as success for servers.

### 117. ProcessStartRetryPolicy for Port Binding
Transient port conflicts (TIME_WAIT from previous runs) handled via retry with exponential backoff. Policy matches specific error message strings.

### 118. Workload Timeout = WarmupTime + 2×TestDuration
Absolute timeout prevents stuck workloads. 2× multiplier gives headroom for slow finalization.

### 119. Results File Polling
When tools write to file (not stdout), poll for file existence + non-empty content with timeout. Catch IOException (file still being written). Throw WorkloadResultsException on timeout.

### 120. SetIfNotDefined vs GetValue Default
`Parameters.SetIfNotDefined(name, value)` writes the default INTO the dictionary (visible to other readers). `GetValue<T>(name, default)` returns default without writing. Use SetIfNotDefined when the default must be forwarded to sub-executors or serialized.

## Patterns 121-132: Domain Knowledge & Edge Cases (Session 5)

### 121. The 512 Queue Depth Invariant
FIO and DiskSpd both use `QueueDepth = 512 / ThreadCount` with `ThreadCount = LogicalCoreCount / 2`. Total outstanding I/O is ALWAYS 512 regardless of VM size. This makes IOPS results directly comparable across different VMs. Changing this formula breaks cross-VM comparability.

### 122. FileSize Must Exceed RAM
FIO FileSize=496G and DiskSpd -c496G must exceed the VM's maximum possible RAM. If the file fits in memory, the OS caches it and you benchmark RAM, not disk. The 496G leaves 4G headroom from the 500G DiskFill for filesystem metadata.

### 123. DiskFill Forces SSD Steady-State
Pre-filling the disk with DiskFillSize=500G forces the SSD controller into steady-state where garbage collection and wear-leveling are active. Without fill, SSDs report artificially high write speeds on empty NAND pages.

### 124. Block Size Set Covers Real Workloads
4k=filesystem pages, 8k=SQL Server pages, 12k=alignment stress test (non-power-of-2), 16k=InnoDB pages, 1024k=sequential streaming. FIO and DiskSpd use identical block sizes for cross-platform comparability.

### 125. Redis Key Count Inversely Proportional to Key Size
`10M×32B≈320MB`, `500K×1KB≈500MB`, `50K×10KB≈500MB`. Total memory footprint per pool is kept similar (300-500MB). Ensures working set exceeds L3 cache. Memcached uses larger key counts because its slab allocator has lower per-key overhead.

### 126. Pipeline Depth Is Redis Queue Depth
Warmup pipeline=32 (stable bulk loading), benchmark pipeline=100 (saturate command queue for peak throughput measurement). Pipeline depth determines how many commands are in-flight before waiting for responses.

### 127. OpenSSL Algorithm Coverage
The 13-algorithm set covers: hashes (MD5, SHA1/256/512), symmetric ciphers (DES-EDE3, AES-128/192/256-CBC, Camellia-128/192/256-CBC), and asymmetric (RSA2048/4096). AES tests hardware acceleration (AES-NI), Camellia serves as software-only control. RSA measures TLS handshake capacity.

### 128. Process Exit Code Semantics
137 = SIGKILL (128+9) — expected for killed servers. 130 = SIGINT (128+2) — expected for interrupted clients. -1 = SafeKill result. Always add workload-specific exit codes to success list with comments explaining WHY.

### 129. Task.WhenAny vs WhenAll Decision
Servers → WhenAny (detect first failure fast, restart all). Independent workloads (FIO per-disk) → WhenAll (each process is independent, partial success not meaningful).

### 130. DiskFill State Persistence
Save state AFTER disk fill completes; check state BEFORE starting fill. State key includes executor type name to avoid collisions. This prevents re-filling disks (30+ minutes) on every execution round.

### 131. Two-Layer Retry for Client-Server
Flow-level retry (3 retries) wraps the entire client workflow (connect, get state, run). Instance-level retry (3 retries) wraps individual process startups. ALWAYS exclude OperationCanceledException from retry — cancellation means "stop", not "try again".

### 132. Delete Results Before Run, Not After
Geekbench deletes results file BEFORE executing, not after. If the new run crashes, you get a clean "no results" error instead of silently parsing stale previous-run results.

### 133. Calculate Expression Uses Roslyn CSharpScript
`{calculate(...)}` is evaluated by `Microsoft.CodeAnalysis.CSharp.Scripting.CSharpScript.EvaluateAsync<object>()` — it compiles and executes real C# code at runtime. This means full C# expression syntax works: method calls (`"linux-x64".StartsWith("linux")`), casts (`(double)322.5/5`), ternary (`true ? "yes" : "no"`). Bool results are auto-lowercased to `true`/`false`.

### 134. Three-Phase Expression Evaluation Order
Profile expressions resolve in strict order: (1) well-known variables (`{LogicalCoreCount}`, `{Platform}`, `{PackagePath:x}`), (2) parameter cross-references (`{ThreadCount}`, `{Port}`), (3) calculate expressions. Each phase loops up to 5 iterations for nested resolution. This is WHY `{calculate(512/{ThreadCount})}` works — `{ThreadCount}` resolves in phase 2 before calculate runs in phase 3.

### 135. Platform Ternary Pattern
`{calculate("{Platform}".StartsWith("linux") ? "libaio" : "windowsaio")}` — strings inside calculate MUST be quoted because after phase 1 substitution, CSharpScript sees `"linux-x64".StartsWith("linux")`. Used in FIO (IO engine), SPECCPU (benchmark selection), and SPECCPU (compiler flags by architecture). Single profile → cross-platform.

### 136. Integer Division Gotcha in Calculate
C# integer division truncates: `512/3` → `170`, not `170.67`. When computing percentages of memory, ALWAYS multiply first: `{SystemMemoryBytes} * 80 / 100` (correct) vs `{SystemMemoryBytes} / 100 * 80` (loses precision). This is real C# math — `long * int / int` stays in integer domain throughout.

### 137. NetworkingWorkloadExecutor — Meta-Orchestrator Pattern
A third client-server architecture beyond Direct Reference and Orchestrator: `NetworkingWorkloadExecutor` uses a `ToolName` parameter to dispatch to the right tool-specific sub-executor. This allows a single profile to mix NTttcp + Latte + CPS + SockPerf scenarios under one executor type. Profile pattern: "one executor type, tool selection via parameter."

### 138. Per-Tool Prefixed Parameters in Combined Profiles
When multiple tools share a profile, each gets prefixed top-level params: `SockPerfPort`/`SockPerfDuration`, `NTttcpPort`/`NTttcpDuration`, `CpsPort`/`CpsDuration`. This allows independent override of each tool's config via `--parameters` CLI. Generic `TestDuration` can serve as fallback.

### 139. Network Configuration as Prerequisite Dependency
Network workloads require system-level tuning before benchmarks: firewall disable, busy poll enable. Done via `system_config` package with platform-conditional `ExecuteCommand` steps (`config_network.sh` for Linux, `config_network.cmd` for Windows). The `EnableBusyPoll` param flows from top-level into the script command.

### 140. Pre-Built Binary Packages vs Source Compilation
Not all tools compile from source. Network tools ship as pre-built binaries in `networking.4.0.0.zip` via `DependencyPackageInstallation`. Source compilation (`WgetPackageInstallation` + build deps) is only for tools that MUST compile on-target (Redis for CPU-specific optimizations, SockPerf source is available but pre-built is preferred in the combined profile).

### 141. Profiling as Cross-Cutting Scenario Concern
Every network scenario carries `ProfilingScenario`, `ProfilingEnabled`, `ProfilingMode` params. These enable optional perf/dotnet-trace profiling. `ProfilingScenario` value typically matches Scenario name exactly. Top-level defaults: `ProfilingEnabled: false`, `ProfilingMode: "None"`.

### 142. Protocol Variation Over Message Size for Latency Tools
SockPerf varies TCP/UDP (not message size) because latency is protocol-sensitive: TCP has connection overhead, UDP is connectionless. NTttcp varies buffer sizes AND thread counts because throughput scales with both. Each tool varies the dimension most relevant to its measurement domain.

### 143. ApiServer Role Restriction
`"Roles": "Server"` on ApiServer means it only starts on Server VM. Client-server sync requires API on server side only in some profiles. This is distinct from profiles that start ApiServer on both roles.

### 144. MetricScenario for Metric Output Normalization
Profiles use `MetricScenario` (lowercase) alongside `Scenario` (PascalCase/display) to normalize metric names. Example: Scenario="AES-128-CBC", MetricScenario="aes-128-cbc". Ensures consistent metric keys regardless of scenario display name casing.

### 145. Tags for Metric Categorization
Scenarios can have `"Tags": "CPU,OpenSSL,Cryptography"` — comma-separated categories for filtering/grouping metrics in telemetry. Tags are typically workload-level consistent (same across all scenarios in a profile).

### 146. TimeSpan Decomposition in Commands
`{Duration.TotalSeconds}` decomposes a TimeSpan parameter into numeric seconds for CLI arguments. Available: `.TotalDays`, `.TotalHours`, `.TotalMinutes`, `.TotalSeconds`, `.TotalMilliseconds`. Keeps source of truth as TimeSpan while giving CLI the number it needs.

### 147. Executor-Managed Threading
OpenSSL's `-multi` flag is NOT in the profile — executor adds it internally. Profile specifies WHAT to benchmark; executor handles HOW (including parallelism). Keeps profiles portable across VM sizes. Compare to FIO where ThreadCount IS in the profile because it's a tuning dimension.

### 148. Minimal Scenario Naming for CPU Workloads
CPU workload scenarios use just the variant name: `MD5`, `SHA256`, `RSA2048` — not `OpenSSL_Speed_MD5`. Tool name is implicit from executor type; benchmark mode (speed) is the only mode. Only add prefixes when disambiguation is needed (multiple tools in one profile, or multiple modes).

### 149. GitRepoClone Dependency Type
`GitRepoClone` clones a git repo as a dependency. Params: `RepoUri` (GitHub URL), `PackageName` (output name). Use for live repos (CoreMark). Use `WgetPackageInstallation` for tagged release tarballs. Use `DependencyPackageInstallation` for pre-built packages from blob storage.

### 150. CompilerInstallation Dependency Type
Dedicated dependency for build compiler. Params: `CompilerVersion` (empty = system default), `CygwinPackages` (Windows build via Cygwin: "gcc-g++,gcc,perl"). Handles both Linux (apt/dnf) and Windows (Cygwin) transparently.

### 151. ThreadCount=null for Auto-Detection
`"ThreadCount": null` means "executor decides." Different from `{calculate({LogicalCoreCount})}` which forces a value. null = trust executor's default algorithm. The executor typically uses logical core count internally.

### 152. MinimumExecutionInterval vs RecommendedMinimumExecutionTime
`MinimumExecutionInterval` (top-level, enforced): prevents re-execution faster than interval. `RecommendedMinimumExecutionTime` (Metadata, advisory): tells users how long to run. Independent settings — one enforces, one advises.

### 153. Don't Over-Apply Cross-Cutting Patterns
MetricScenario and Tags exist in OpenSSL but NOT CoreMark. Not all profiles use the same cross-cutting params. Always check executor source or similar profiles before assuming a pattern is universal. False positives (adding unnecessary params) are as wrong as false negatives (missing required params).

### 154. Scenario Naming — 6 Rules
1. Prefix with tool name ONLY in combined multi-tool profiles (`NTttcp_TCP_...`, `SockPerf_TCP_...`)
2. Use native benchmark test names when available (`oltp_read_write`, `memtier_8t_16c_32b_r1:1`)
3. Encode ALL varying dimensions in name (`RandomWrite_4k_BlockSize`, `NTttcp_TCP_4K_Buffer_T32`)
4. Case: PascalCase for IO patterns, lowercase for sizes, UPPERCASE for protocols, original case for native names
5. Special prefixes: `DiskFill` (prep), `warmup_` (pre-populate), `DataIntegrity_` (verification), `Execute...Benchmark` (single formal benchmark)
6. For single-scenario workloads: `Execute{Tool}Benchmark` or `{WhatItMeasures}`

### 155. Dynamic MetricScenario with Variable Substitution
`MetricScenario` can be a runtime template: `cpustress_t{ThreadCount}_fft{MaxTortureFFT}_{Duration.TotalMinutes}mins`. Variables resolve at runtime creating metric names like `cpustress_t7_fft512_15mins`. Enables metric differentiation when users override params via CLI.

### 156. Min/Max Range Parameters for Tool-Native Configs
Always use the tool's native parameter names. Prime95 has `MinTortureFFT`/`MaxTortureFFT` (range, set equal for single FFT). Don't invent `FFTSize` abstraction — it hides the tool's range capability and doesn't match the CLI.

### 157. Two Command Line Models: Profile-Specified vs Executor-Built
**Profile-specified**: FIO, DiskSpd, Redis, Memcached, OpenSSL have `CommandLine`/`CommandArguments` in the profile with `{Param}` placeholders resolved via ApplyParameters. The profile controls the exact CLI invocation.
**Executor-built**: Prime95, CoreMark have NO CommandLine — the executor builds the command internally from individual parameters (MinTortureFFT, ThreadCount, etc.). Use executor-built when the command structure is complex or version-dependent.
Parameter names: `CommandLine` (most executors) vs `CommandArguments` (OpenSSL, possibly newer convention).

### 158. Two Parameter Substitution Syntaxes
`$.Parameters.X` = JSONPath reference from one Parameters block to another (e.g., `"Duration": "$.Parameters.Duration"` copies top-level Duration into action). Resolved at profile load time.
`{X}` = ApplyParameters substitution inside string values (e.g., `"--size={FileSize}"` in CommandLine). Resolved at expression evaluation time from the merged parameter set. These are DIFFERENT systems — don't mix them up.

### 159. Dependency Type Decision Tree (21 types, 7 patterns)
**Package installation**: DependencyPackageInstallation (blob store, most common), LinuxPackageInstallation (apt/yum), WgetPackageInstallation (URL download), GitRepoClone (git repo), ChocolateyInstallation/ChocolateyPackageInstallation (Windows), JDKPackageDependencyInstallation (Java).
**Build**: CompilerInstallation (GCC), ExecuteCommand (make/configure escape hatch).
**Disk**: FormatDisks → MountDisks (always paired, IO/DB workloads).
**Runtime**: ApiServer (client-server, always last), DockerInstallation, NvidiaCudaInstallation + NvidiaContainerToolkitInstallation (GPU stack).
**DB-specific**: MySQLServerInstallation/Configuration, PostgreSQLServerInstallation/Configuration, SysbenchConfiguration, HammerDBExecutor (as dependency).

### 160. Seven Dependency Combination Patterns
1. **Simple binary**: DependencyPackageInstallation only (Prime95, Geekbench)
2. **Cross-platform**: LinuxPackageInstallation + FormatDisks/MountDisks + DependencyPackageInstallation with SupportedPlatforms filter (FIO)
3. **Build-from-source**: LinuxPackageInstallation (build deps) → WgetPackageInstallation/GitRepoClone (source) → ExecuteCommand (compile) → ApiServer (Redis, Memcached)
4. **Database**: FormatDisks/MountDisks → LinuxPackageInstallation → DependencyPackageInstallation × 2 → DBServerInstallation → DBServerConfiguration × N → BenchmarkConfiguration → ApiServer
5. **GPU**: FormatDisks → GitRepoClone → NvidiaCudaInstallation → DockerInstallation → NvidiaContainerToolkitInstallation
6. **Compilation workload**: ChocolateyInstallation → ChocolateyPackageInstallation (Cygwin) → CompilerInstallation → DependencyPackageInstallation (SPEC CPU)
7. **Network**: DependencyPackageInstallation × 2 (config + tools) → LinuxPackageInstallation → ExecuteCommand (platform-conditional) × 2 → ApiServer

### 161. Distro-Specific Package Names in LinuxPackageInstallation
Use `Packages-Apt`/`Packages-Yum`/`Packages-Dnf` when package names differ across distros: `libtirpc-dev` (Debian/Ubuntu) vs `libtirpc-devel` (RHEL/Fedora). Only use generic `Packages` when the name is identical everywhere (e.g., `make`, `gcc`, `automake`).

### 162. Hardware-Adaptive Parameters with Calculate
LMBench: `"MemorySizeMB": "{calculate({SystemMemoryMegabytes} / 4)}"` — test 25% of RAM. This is the same pattern as Redis/Memcached memory sizing and MySQL/PostgreSQL buffer pool sizing. Always use calculate for memory-proportional configs so profiles adapt to any VM size.

### 163. RecommendedMinimumExecutionTime Can Be Per-Configuration
LMBench uses `"(4-cores)=04:00:00,(16-cores)=10:00:00,(64-cores)=16:00:00"` — a human-readable per-configuration guideline, not a single value. This format communicates that execution time scales with hardware.

### 164. Executor Type Name Casing Must Match Exactly
`LMbenchExecutor` (lowercase 'b'), not `LMBenchExecutor`. Type names in profiles must match the C# class name exactly. When in doubt, check the source code for the exact casing.

### 165. Network Profile v1→v2 Evolution
PERF-NETWORK.json (v1) uses `NetworkingWorkloadExecutor` (meta-orchestrator) with `ToolName` dispatch. PERF-NETWORK-2.json (v2) uses typed executors (`SockPerfClientExecutor2`, `NTttcpClientExecutor2`, `CPSClientExecutor2`) with explicit `Role: "Client"` and a `NetworkingWorkloadProxy` for the server. This shows the framework migrating from meta-orchestrators to direct role-typed executors — the v2 pattern is preferred for new profiles.

### 166. Executor Runtime Environment Setup
OpenSSL executor sets `LD_LIBRARY_PATH` on Linux and appends to `PATH` on Windows. This is done in the executor, not the profile. Executors handle platform-specific environment variable setup so profiles stay clean. Similarly, `-multi {ProcessorCount}` is injected by the executor, not specified in profile CommandArguments.
