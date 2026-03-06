# VC Memory — Conventions & Standards

## Code Style
- Style enforced via StyleCop + Roslyn analyzers integrated into build
- **If it builds, most style requirements are met** — the toolset handles enforcement
- Every `.csproj` must import `.buildenv` settings (see bottom of any existing .csproj)

## Naming Conventions
| Entity | Convention | Example |
|--------|-----------|---------|
| Executor class | `{Workload}Executor` | `FioExecutor`, `OpenSslExecutor` |
| Parser class | `{Workload}ResultsParser` | `FioResultsParser` |
| Metrics parser | `{Workload}MetricsParser` | `OpenSslMetricsParser` |
| Test class | `{ClassName}Tests` | `FioExecutorTests` |
| Profile JSON | `PERF-{CATEGORY}-{WORKLOAD}.json` | `PERF-IO-FIO.json` |
| Dependency class | `{Name}Installation` or `{Name}Handler` | `ChocolateyInstallation` |
| Monitor class | `{Name}Monitor` | `PerfCounterMonitor`, `AtopMonitor` |

## Project Namespace Pattern
```
VirtualClient.Actions.{WorkloadName}    // executor + parser classes
VirtualClient.Dependencies.{Name}       // dependency handler classes
VirtualClient.Monitors.{Name}           // monitor classes
```

## Assembly Naming for Extensions
```
{TeamName}.VirtualClient.Extensions.{ComponentType}.dll
// e.g.: CRC.VirtualClient.Extensions.Actions.dll
```

## Test Requirements
- **Unit tests required** for all new/changed code — PRs blocked without them
- Test categories: must use `[Category("Unit")]` or `[Category("Functional")]`
- Use `NUnit` as the test framework
- Test files: `{ClassName}Tests.cs` in `*.UnitTests` or `*.FunctionalTests` project

## MockFixture vs DependencyFixture
| Fixture | Use For | Backing |
|---------|---------|---------|
| `MockFixture` | Unit tests | Moq mocks — pure stubs |
| `DependencyFixture` | Functional tests | In-memory implementations |

```csharp
// Unit test pattern (MockFixture)
[Test]
[Category("Unit")]
public async Task FioExecutorRunsExpectedCommand()
{
    MockFixture fixture = new MockFixture();
    fixture.SetupMocks();
    FioExecutor component = new FioExecutor(fixture.Dependencies, fixture.Parameters);
    await component.ExecuteAsync(CancellationToken.None);
    fixture.ProcessManager.Verify(pm => pm.CreateProcess(...), Times.Once);
}
```

## Test Resource Files
- Stored in `VirtualClient.Actions.UnitTests/Examples/{WorkloadName}/`
- Register in `*.UnitTests.csproj` with `<CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>`

## PR Process
1. Branch from main: `users/{alias}/{ChangeDescription}`
2. No direct pushes to `main`
3. All solutions must build successfully
4. All unit + functional tests must pass (gate requirement)
5. Update relevant markdown docs
6. At least 1 team reviewer approval required
7. CLA agreement required for external contributors (automated bot check)
8. Automated PR build (GitHub Actions) must pass

## PR Build Pipelines
- `.github/workflows/pull-request.yml` — Windows build
- `.github/workflows/pull-request-linux.yml` — Linux build

## Versioning
- Semantic versioning: `{major}.{minor}.{patch}`
- Current line: `2.1.x`
- Version file: `VERSION` at repo root
- Assemblies must have version bumped when changed

## Cross-Platform Rules
- MUST support all 4 platforms unless technically impossible
- Mark exceptions with `[SupportedPlatforms("linux-x64,linux-arm64")]`
- Test on both Windows and Linux before submitting PR
- Never use Windows-only APIs (e.g. `Registry`, `WMI`) without platform guard

## File Organization for New Workloads
```
VirtualClient.Actions/
  {WorkloadName}/
    {Workload}Executor.cs
    {Workload}MetricsParser.cs   (or ResultsParser.cs)

VirtualClient.Actions.UnitTests/
  {WorkloadName}/
    {Workload}ExecutorTests.cs
    {Workload}MetricsParserTests.cs
    Examples/
      {WorkloadName}/
        example_output.txt

VirtualClient.Main/profiles/
  PERF-{CATEGORY}-{WORKLOAD}.json
```
