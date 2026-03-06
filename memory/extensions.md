# VC Memory — Extensions Development

## What Are Extensions?
Extensions are Actions, Monitors, and Dependencies developed in a **separate repo** from the main VC platform.
They are loaded at runtime via dependency packages — no changes to the core VC repo needed.

## NuGet Framework Package
```xml
<PackageReference Include="VirtualClient.Framework" Version="2.1.*" />
<PackageReference Include="VirtualClient.TestFramework" Version="2.1.*" />
```
- `VirtualClient.Framework` bundles `Contracts + Core`
- Version must match the target VC runtime version (major.minor must match)

## Version Compatibility
| Framework Lib Version | Supported VC.exe Versions |
|----------------------|--------------------------|
| `1.0.*` | `1.0.*` |
| `1.1.*` | `1.1.*` |
| `2.0.*` | `2.0.*` |
| `2.1.*` | `2.1.*` ← current |

## Assembly Requirements
1. Must target `.NET 9.0` (same as VC runtime)
2. Must have `VirtualClient` in the assembly name
3. Must include this attribute in an `AssemblyInfo.cs`:
```csharp
[assembly: VirtualClient.Contracts.VirtualClientComponentAssembly]
```

## Assembly Naming Convention
```
{TeamName}.VirtualClient.Extensions.{ComponentType}.dll
// Examples:
CRC.VirtualClient.Extensions.Actions.dll
CRC.VirtualClient.Extensions.Monitors.dll
CRC.VirtualClient.Extensions.Dependencies.dll
```

## Extension Package Structure (.vcpkg)
```
/my-extensions/
├── my-extensions.vcpkg          # Package definition JSON (root)
├── linux-x64/
│   ├── MyTeam.VirtualClient.Extensions.Actions.dll
│   └── profiles/
│       └── MY-CUSTOM-PROFILE.json
├── linux-arm64/
│   └── ...
├── win-x64/
│   └── ...
└── win-arm64/
    └── ...
```

## Package Definition File (.vcpkg)
```json
{
  "name": "myteamvcextensions",
  "description": "My Team Virtual Client extensions.",
  "version": "1.0.1",
  "metadata": {
    "extensions": true      // ← REQUIRED for extension packages
  }
}
```

## Loading Extensions at Runtime
Extensions are loaded as a dependency in the profile:
```json
{
  "Dependencies": [
    {
      "Type": "DependencyPackageInstallation",
      "Parameters": {
        "Scenario": "InstallMyExtensions",
        "BlobContainer": "packages",
        "BlobName": "myteamvcextensions.1.0.1.zip",
        "PackageName": "myteamvcextensions",
        "Extract": true
      }
    }
  ]
}
```

## Profile Extensions
Profile JSON files placed in `{platform}/profiles/` folder within the package are automatically available to the VC runtime after the package is loaded.

## Key Reference
- Dev guide: `website/docs/developing/0020-develop-extensions.md`
- Usage guide: `website/docs/guides/0221-usage-extensions.md`
- Internal reference repo: CRC-VirtualClient-Examples (Azure DevOps)
