# VC Memory — Build & Packaging

## Build Commands
### Windows
```cmd
build.cmd                   # Build all projects
build-packages.cmd          # Build NuGet packages from artifacts
build-test.cmd              # Run tests
```

### Linux/Mac
```bash
./build.sh                  # Build all projects
./build-packages.sh         # Build NuGet packages
./build-test.sh             # Run tests
```

## Prerequisites
- .NET SDK 9.0.x

## Build Artifacts
```
{rootdir}/out/
├── bin/
│   ├── Debug|Release/
│   │   ├── AnyCPU/        # Shared assemblies
│   │   ├── x64/           # x64 platform executables
│   │   └── ARM64/         # ARM64 platform executables
├── obj/                   # Intermediate files
└── packages/              # Output NuGet packages
```

## Executable Locations (after build)
```
out/bin/Release/x64/VirtualClient.Main/net9.0/linux-x64/VirtualClient
out/bin/Release/x64/VirtualClient.Main/net9.0/win-x64/VirtualClient.exe
out/bin/Release/ARM64/VirtualClient.Main/net9.0/linux-arm64/VirtualClient
out/bin/Release/ARM64/VirtualClient.Main/net9.0/win-arm64/VirtualClient.exe
```

## NuGet Packages (4 per release)
| Package | Platform |
|---------|---------|
| `VirtualClient.linux-x64` | Linux Intel/AMD |
| `VirtualClient.linux-arm64` | Linux ARM64 |
| `VirtualClient.win-x64` | Windows Intel/AMD |
| `VirtualClient.win-arm64` | Windows ARM64 |

Download URLs: `https://www.nuget.org/api/v2/package/VirtualClient.{platform}/{version}`

## VC Packages (.vcpkg)
A VC package is a structured zip file containing workload binaries/scripts.

### Package Definition
Every package has a `.vcpkg` JSON file in its root:
```json
{
  "name": "fio",
  "description": "FIO flexible I/O benchmark package.",
  "version": "3.36.0",
  "metadata": {}
}
```

### Standard Package Layout
```
/{package_name}/
├── {package_name}.vcpkg      # Package definition
├── linux-x64/                # Platform-specific binaries
│   └── fio
├── linux-arm64/
│   └── fio
├── win-x64/
│   └── fio.exe
└── win-arm64/
    └── fio.exe
```

### Access in Profile/Code
```json
// In profile (resolves to platform-specific subfolder)
"WorkingDirectory": "{PackageDir/Platform:fio}"

// In profile (resolves to package root)
"WorkingDirectory": "{PackageDir:fio}"
```

## CI/CD Pipelines
| Pipeline | File | Purpose |
|---------|------|---------|
| PR Build (Windows) | `.github/workflows/pull-request.yml` | Gate for PRs |
| PR Build (Linux) | `.github/workflows/pull-request-linux.yml` | Gate for PRs |
| Docs Build | `.github/workflows/build-doc.yml` | Build documentation |
| Docs Deploy | `.github/workflows/deploy-doc.yml` | Deploy to GitHub Pages |
| Azure Pipelines (Windows) | `.pipelines/azure-pipelines.yml` | Official release build |
| Azure Pipelines (Linux) | `.pipelines/azure-pipelines-linux.yml` | Official release build |

## Release Process
1. Build artifacts via Azure Pipelines
2. NuGet packages uploaded: `upload-packages.cmd` / `upload-packages-internal.cmd`
3. GitHub Releases page updated with zip artifacts
4. Deb/RPM packages published to Microsoft package store

## Documentation Build
- Built with Docusaurus (`website/` directory)
- Deploy: `publish-document.cmd`
- Live: microsoft.github.io/VirtualClient/

## Key Build Config Files
| File | Purpose |
|------|---------|
| `.buildenv/BuildEnv.props` | Shared build properties |
| `.buildenv/BuildEnv.targets` | Shared build targets |
| `.buildenv/rules/CodeAnalysis.ruleset` | Code analysis rules |
| `.buildenv/rules/StyleCop.Analyzers.ruleset` | StyleCop rules |
| `Directory.Packages.props` | Central NuGet version management |
| `global.json` | .NET SDK version pin |
| `Module.props` | Per-module import (required at bottom of every .csproj) |
