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
