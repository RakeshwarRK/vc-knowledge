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
