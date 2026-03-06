# VC Memory — Error Handling

## Exception Hierarchy
All VC exceptions derive from `VirtualClientException` → `Exception`.

```csharp
VirtualClientException           // base
├── WorkloadException            // errors during workload execution
├── WorkloadResultsException     // errors parsing workload results  
├── DependencyException          // errors during dependency installation
├── MonitorException             // errors during monitoring
└── ApiException                 // errors from/to REST API
```

## ErrorReason Ranges (critical to understand)
| Range | Severity | Behavior |
|-------|---------|---------|
| 100–399 | **Transient** | Handled — VC continues to next component |
| 400–499 | **Uncertain** | Generally handled — may or may not be transient |
| ≥ 500 | **Terminal/Permanent** | **VC crashes** — unrecoverable |

### Example ErrorReasons
| Reason | Value | Category |
|--------|-------|---------|
| `WorkloadResultsNotFound` | 314 | Transient |
| `ProfileNotFound` | 500 | Terminal |
| `DependencyNotFound` | ~300s | Transient |

## Throwing Exceptions (standard patterns)

```csharp
// Transient: workload result file not found (will retry)
throw new WorkloadException(
    $"The expected results file '{resultFile}' was not found for workload '{this.PackageName}'.",
    exc,
    ErrorReason.WorkloadResultsNotFound);  // 314 — transient

// Terminal: profile file does not exist (cannot recover)
throw new DependencyException(
    $"A profile '{profileName}' does not exist at path '{profilePath}'.",
    exc,
    ErrorReason.ProfileNotFound);  // 500 — crashes VC

// Dependency not installed (transient — retry)
throw new DependencyException(
    $"The required package '{this.PackageName}' was not found on the system.",
    exc,
    ErrorReason.DependencyNotFound);
```

## Always Include Inner Exception
```csharp
// WRONG
throw new WorkloadException("File not found.");

// CORRECT — preserves full exception chain
throw new WorkloadException(
    $"The script '{scriptPath}' was not found in package '{this.PackageName}'.",
    exc,                           // always include original exception
    ErrorReason.DependencyNotFound);
```

## Retry Pattern (Polly)
Used for transient failures. Polly is already a dependency — never implement manual retry loops.

```csharp
// Standard retry policy pattern used in VC
IAsyncPolicy retryPolicy = Policy
    .Handle<Exception>(exc => exc is TransientException)
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(attempt * 2));

await retryPolicy.ExecuteAsync(async () =>
{
    await this.someOperation(cancellationToken);
});
```

## FailFast Mode
- Parameter: `FailFast` (bool, default: false)
- When true: VC exits on ANY error regardless of ErrorReason severity
- Useful for CI validation scenarios where any failure is a hard stop

## Profile Execution Error Logic
- Located in: `VirtualClient.Core/ProfileExecutor.cs`
- Unhandled exception types → crash
- `VirtualClientException` with transient ErrorReason → log + continue
- `VirtualClientException` with terminal ErrorReason → crash

## Error Message Quality Rules
1. **Name the file/resource** — never say "file not found" without naming it
2. **Include package/component context** — which workload was running?
3. **Suggest remediation** if possible — what should the user check?
4. **Think "user will see this"** — write for the operator, not just the developer

```csharp
// BAD
throw new WorkloadException("The file was not found.", exc);

// GOOD
throw new WorkloadException(
    $"The expected results file '{resultsFile}' was not found in the '{this.PackageName}' package " +
    $"after workload execution completed. Verify the workload ran successfully.",
    exc,
    ErrorReason.WorkloadResultsNotFound);
```
