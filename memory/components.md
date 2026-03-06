# VC Memory — Components

## Component Types
| Type | Purpose | Base Project |
|------|---------|-------------|
| **Actions** | Execute workloads/benchmarks | VirtualClient.Actions |
| **Dependencies** | Install prerequisites, download packages, configure system | VirtualClient.Dependencies |
| **Monitors** | Run in background capturing metrics | VirtualClient.Monitors |

All three derive from `VirtualClientComponent` (in `VirtualClient.Contracts`).

## VirtualClientComponent Base Class
- Location: `src/VirtualClient/VirtualClient.Contracts/VirtualClientComponent.cs`
- Constructor receives `IServiceCollection dependencies` (DI container) + optional `IDictionary<string, IConvertible> parameters`

### Key Properties
| Property | Type | Description |
|----------|------|-------------|
| `Dependencies` | `IServiceCollection` | The full DI container |
| `Parameters` | `IDictionary<string, IConvertible>` | Profile + CLI parameters (case-insensitive) |
| `Platform` | `PlatformID` | Current OS platform |
| `CpuArchitecture` | `Architecture` | x64 or ARM64 |
| `AgentId` | `string` | VM/machine ID in experiment |
| `ExperimentId` | `string` | Experiment correlation ID |
| `Logger` | `ILogger` | Structured logger |
| `MetadataContract` | `MetadataContract` | Telemetry metadata |
| `PlatformSpecifics` | | Platform-specific path helpers |
| `CleanupTasks` | `IList<Action>` | Tasks run on component completion |
| `Extensions` | `IDictionary<string, JToken>` | Profile-defined extension data |
| `SupportedRoles` | `IList<string>` | Roles this component handles (client/server) |
| `FailFast` | `bool` | Exit on any error (default: false) |
| `LogToFile` | `bool` | Log process output to file |
| `ClientRequestId` | `Guid?` | Correlate client↔server operations |
| `Roles` | | The role(s) this instance is playing |
| `Layout` | `EnvironmentLayout` | Multi-system topology description |

### Lifecycle Methods (implement in subclass)
```csharp
protected override Task InitializeAsync(EventContext telemetryContext, CancellationToken cancellationToken)
protected override Task ExecuteAsync(EventContext telemetryContext, CancellationToken cancellationToken)
protected override Task CleanupAsync(EventContext telemetryContext, CancellationToken cancellationToken)
```

### Getting Dependencies from DI
```csharp
ISystemManagement systemManagement = this.Dependencies.GetService<ISystemManagement>();
IFileSystem fileSystem = systemManagement.FileSystem;
IPackageManager packageManager = systemManagement.PackageManager;
ProcessManager processManager = systemManagement.ProcessManager;
```

## ISystemManagement Interface
- Location: `src/VirtualClient/VirtualClient.Core/ISystemManagement.cs`
- The **master gateway** to all system services. Never bypass it with direct System.IO calls.

```csharp
public interface ISystemManagement : ISystemInfo
{
    IDiskManager       DiskManager      { get; }  // Disk info, mount points, formatting
    IFileSystem        FileSystem       { get; }  // File system operations (abstracted)
    IFirewallManager   FirewallManager  { get; }  // Firewall rule management
    IPackageManager    PackageManager   { get; }  // Package discovery, download, extraction
    ProcessManager     ProcessManager   { get; }  // Create/manage OS processes
    ISshClientFactory  SshClientFactory { get; }  // SSH client creation
    IStateManager      StateManager     { get; }  // Persist/retrieve state between operations
    void EnableLongPathInWindows();                // Override 260-char Windows path limit
}
```

## ExampleWorkloadExecutor Pattern
- Location: `src/VirtualClient/VirtualClient.Actions/Examples/`
- Canonical reference for new workload implementation
- File: `ExampleWorkloadExecutor.cs`
- Client/server examples: `VirtualClient.Actions/Examples/ClientServer/`

## Actions Examples
- `OpenSslExecutor` — CPU crypto benchmark
- `DiskSpdExecutor` — Windows disk I/O
- `FioExecutor` — Linux disk I/O
- `CoreMarkExecutor`, `CoreMarkProExecutor` — CPU benchmarks
- `RedisServerExecutor` / `RedisClientExecutor` — client/server pair
- `MemcachedServerExecutor` / `MemcachedClientExecutor` — client/server pair

## Dependency Examples
- `DependencyPackageInstallation` — download .vcpkg from Azure Blob
- `GitRepoClone` — clone a Git repo as a package
- `CompilerInstallation` — install gcc/g++ at specified version
- `WgetPackageInstallation` — download via wget
- `ExecuteCommand` — run arbitrary shell command as dependency step
- `ChocolateyInstallation` — Windows package manager
- `DockerInstallation` — install Docker

## Monitor Examples
- `PerfCounterMonitor` — system perf counters
- `AtopMonitor` — Linux atop process/resource monitor
- `NvidiaSmiMonitor` — NVIDIA GPU metrics
- `LspciMonitor` — PCI device information capture

## Profile Component Registration
Components are referenced by **type name string** in profile JSON:
```json
{ "Type": "OpenSslExecutor", "Parameters": { ... } }
```
The assembly must be decorated with `[assembly: VirtualClientComponentAssembly]` to be scanned.

## CPU Core Affinity
- Enabled via `BindToCores` parameter + `ProcessorAffinity` property
- Linux: uses `numactl` (affinity set before process starts — zero timing window)
- Windows: uses `Process.ProcessorAffinity` (~1ms window after start)
- Core formats: `"0,2,4,6"` (individual), `"0-7"` (range), `"0-3,8-11"` (combined)
- Windows supports up to 64 cores (single processor group)
- Reference: `ExampleWorkloadWithAffinityExecutor.cs`
