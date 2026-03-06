# VC Memory — Command Line Reference

## Basic Usage
```bash
VirtualClient.exe --profile=PERF-CPU-OPENSSL.json [options]
VirtualClient    --profile=PERF-IO-FIO.json [options]   # Linux
```

## Core Options
| Flag | Required | Type | Description |
|------|----------|------|-------------|
| `--p`, `--profile` | Yes | string | Profile name or path. Repeat for multiple profiles. |
| `--ps`, `--packages`, `--package-store` | No* | conn string/SAS | Azure Storage for package downloads. Default: VC public storage. |
| `--t`, `--timeout` | No | timespan | How long to run (e.g. `01:30:00` = 90 min). No timeout = run indefinitely. |
| `--i`, `--iterations` | No | int | Number of full profile iterations. Default = infinite until timeout. |
| `--pm`, `--parameters` | No | key=val pairs | Override profile parameters: `--parameters="ThreadCount=16,FileSize=100G"` |

## Identity & Correlation
| Flag | Description |
|------|-------------|
| `--s`, `--system` | System/experiment name (used in telemetry) |
| `--agentId` | Unique ID for this VC instance (defaults to machine hostname) |
| `--experimentId` | Experiment correlation GUID |
| `--metadata=key=value` | Attach custom metadata to all telemetry events |

## Multi-System / Client-Server
| Flag | Description |
|------|-------------|
| `--layout`, `--layoutPath` | Path to environment layout JSON file |
| `--api-port` | REST API port (default: 4500). Role-specific: `--api-port=4501/Client,4502/Server` |

## Telemetry Sinks
| Flag | Description |
|------|-------------|
| `--event-hub` | Azure Event Hub connection string |
| `--content-store` | Azure Blob connection string for file uploads |
| `--log-level` | Logging verbosity level |
| `--log-to-file` | Write process stdout/stderr to log files in logs directory |

## Azure Integration
| Flag | Description |
|------|-------------|
| `--cs`, `--content-store` | Azure Blob for content uploads |
| `--key-vault` | Azure Key Vault URI for certificate/secret retrieval |

## Packages / Storage Auth
Supports multiple auth methods for `--packages`:
- Storage Account blob service SAS URIs
- Storage Account blob container SAS URIs  
- Microsoft Entra ID/Apps using a certificate
- Azure managed identities

## Profile Execution Examples
```bash
# Simple single workload
VirtualClient.exe --profile=PERF-CPU-OPENSSL.json --timeout=01:00:00

# Workload + monitor profile combo
VirtualClient.exe --profile=PERF-IO-FIO.json --profile=MONITORS-DEFAULT.json --timeout=02:00:00

# Override parameters
VirtualClient.exe --profile=PERF-CPU-COREMARK.json --parameters="ThreadCount=8"

# With Azure package store
VirtualClient.exe --profile=PERF-IO-FIO.json --packages="{SAS_URI}" --timeout=01:00:00

# Client/server scenario
VirtualClient.exe --profile=PERF-REDIS.json --layout=layout.json --agentId=vm-client-0 \
  --packages="{SAS_URI}" --timeout=01:00:00

# Full telemetry
VirtualClient.exe --profile=PERF-IO-FIO.json \
  --event-hub="{EventHubConnString}" \
  --metadata=Team=CRC --metadata=Purpose=PerformanceBaseline \
  --timeout=02:00:00
```

## Installation Methods
| Method | Command |
|--------|---------|
| Debian/Ubuntu | `sudo apt-get install virtualclient` (via Microsoft package repo) |
| RPM | `sudo dnf install virtualclient` (via Microsoft package repo) |
| NuGet zip | Download from NuGet, extract, run directly |
| GitHub Releases | Download zip from Releases page |
| Build from source | `build.cmd` / `build.sh` |

## Executable Locations (after install)
- Linux: `/usr/bin/virtualclient` (symlink to `/opt/virtualclient/VirtualClient`)
- NuGet package: `content/{platform}/VirtualClient` or `content/{platform}/VirtualClient.exe`
