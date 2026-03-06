# VC Memory — Architecture

## Solution Structure
```
src/VirtualClient/
├── VirtualClient.Main/              # CLI entry point; contains all profile JSON files
├── VirtualClient.Contracts/         # Data contracts, POCOs, VirtualClientComponent base class
├── VirtualClient.Core/              # Shared interfaces + implementations (ISystemManagement, etc.)
├── VirtualClient.Actions/           # Workload executor components
├── VirtualClient.Actions.FunctionalTests/
├── VirtualClient.Actions.UnitTests/
├── VirtualClient.Dependencies/      # Dependency installer/handler components
├── VirtualClient.Dependencies.FunctionalTests/
├── VirtualClient.Dependencies.UnitTests/
├── VirtualClient.Monitors/          # Background monitoring components
├── VirtualClient.Monitors.UnitTests/
├── VirtualClient.TestFramework/     # MockFixture, DependencyFixture, test helpers
└── VirtualClient.TestExtensions/    # MockSetupExtensions, shared test utilities
```

## Dependency Direction (strict — never violate)
```
Actions / Dependencies / Monitors
        ↓
    Core + Contracts
        ↓
   Common (VirtualClient.Common)
```
- Components (Actions, Dependencies, Monitors) MUST NOT depend on each other
- Core owns all shared interfaces; components access them via DI container

## Platform Targets
- `linux-x64`, `linux-arm64`, `win-x64`, `win-arm64`
- Build produces all 4 via `dotnet publish` of `VirtualClient.Main.csproj`

## Build Output Locations
```
{rootdir}/out/bin/Debug|Release/AnyCPU/        # AnyCPU assemblies
{rootdir}/out/bin/Release/x64/VirtualClient.Main/net9.0/linux-x64/VirtualClient
{rootdir}/out/bin/Release/x64/VirtualClient.Main/net9.0/win-x64/VirtualClient.exe
{rootdir}/out/bin/Release/ARM64/VirtualClient.Main/net9.0/linux-arm64/VirtualClient
{rootdir}/out/bin/Release/ARM64/VirtualClient.Main/net9.0/win-arm64/VirtualClient.exe
{rootdir}/out/packages/                        # NuGet packages
```

## Key Configuration Files
- `.buildenv/` — Build environment props, code analysis rules, StyleCop rules
- `Directory.Packages.props` — Central NuGet package version management
- `global.json` — .NET SDK version pin
- `Repo.props` — Repo-wide MSBuild properties
- `Module.props` — Per-module MSBuild import (must appear at bottom of every .csproj)
- `src/VirtualClient/VERSIONING.md` — Versioning policy

## Runtime Internals
- Framework: .NET 9
- DI: `Microsoft.Extensions.DependencyInjection` (`IServiceCollection`)
- Logging: `Microsoft.Extensions.Logging` + custom telemetry extensions
- JSON: Newtonsoft.Json
- File system abstraction: `System.IO.Abstractions` (IFileSystem)
- Retry: Polly
- Unit test mocking: Moq + AutoFixture

## External Resources
- Docs: microsoft.github.io/VirtualClient/
- Public packages storage: Azure Blob (default, no credentials needed for public packages)
- NuGet (4 packages, one per platform): VirtualClient.linux-x64, VirtualClient.linux-arm64, etc.
- Extensions framework NuGet: `VirtualClient.Framework` (bundles Contracts + Core)
- Examples: github.com/microsoft/VirtualClient-Examples (or internal CRC-VirtualClient-Examples)
