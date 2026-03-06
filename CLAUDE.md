# VirtualClient Knowledge Base

One-stop memory store for [microsoft/VirtualClient](https://github.com/microsoft/VirtualClient) — a .NET 9 cross-platform CLI benchmarking platform for Azure hardware validation.

## Memory Files

| File | Contents |
|------|---------|
| [architecture.md](memory/architecture.md) | Solution layout, projects, dependency direction, build output paths |
| [components.md](memory/components.md) | VirtualClientComponent base class, ISystemManagement, lifecycle |
| [profiles.md](memory/profiles.md) | Profile JSON structure, parameters, metric filters, execution order |
| [profiles-catalog.md](memory/profiles-catalog.md) | All 85 profiles cataloged by category with executors, platforms, parameters |
| [workloads.md](memory/workloads.md) | All workloads with profiles and platform support |
| [executors.md](memory/executors.md) | Executor parameters, process commands, parsers, cross-cutting patterns |
| [cli.md](memory/cli.md) | All command line flags |
| [conventions.md](memory/conventions.md) | Coding standards, naming, PR process, test requirements |
| [patterns.md](memory/patterns.md) | Design patterns: executors, retry, platform guards, metrics capture |
| [telemetry.md](memory/telemetry.md) | Data categories, telemetry sinks, Kusto scripts |
| [client-server.md](memory/client-server.md) | Multi-system workloads, environment layouts, REST API |
| [error-handling.md](memory/error-handling.md) | Exception hierarchy, ErrorReason ranges |
| [extensions.md](memory/extensions.md) | Extension development, NuGet framework, .vcpkg packaging |
| [build-and-packaging.md](memory/build-and-packaging.md) | Build commands, pipelines, vcpkg format |
| [docs-catalog.md](memory/docs-catalog.md) | Full documentation catalog, dependency/monitor components, GitHub issues |
| [pr-history.md](memory/pr-history.md) | Complete PR #1-#651 timeline, breaking changes, deprecations, contributor map |
| [vision.md](memory/vision.md) | VC purpose, design philosophy, strategic direction, open questions |
| [training-summary.md](memory/training-summary.md) | PR training loop results: 7 rounds, 28 patterns, capability growth |
| [recent-activity.md](memory/recent-activity.md) | Recent commits/PRs — auto-updated weekly by GitHub Actions |

## Quick Reference

- **Version:** 2.1.61 (v2, .NET 9)
- **Platforms:** linux-x64, linux-arm64, win-x64, win-arm64
- **Profiles:** 85 total (54 PERF, 5 MONITORS, 6 EXAMPLE, 4 POWER, 16 utility/setup)
- **Workloads:** 40+ (CPU, disk, memory, network, database, GPU/ML, HPC, compression, web, stress, power)
- **Design patterns:** 28 documented (lifecycle, SSH fan-out, process models, CLI design, etc.)
- **Key patterns:** VirtualClientComponent lifecycle, client/server coordination, metric filtering, CPU affinity
- **Auto-update:** GitHub Actions runs weekly (Sunday 06:00 UTC) to refresh recent-activity.md
