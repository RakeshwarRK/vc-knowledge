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
