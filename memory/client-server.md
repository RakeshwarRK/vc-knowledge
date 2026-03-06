# VC Memory — Client/Server Workloads

## Overview
Some workloads require 2+ systems (e.g., network benchmarks). VC coordinates them via:
1. An **Environment Layout** file describing all systems and their roles
2. A **built-in REST API** for inter-instance communication

## Environment Layout
Passed via `--layout` or `--layoutPath` CLI flag. JSON format:

```json
{
  "clients": [
    {
      "name": "{agent_id_or_machine_name}",
      "role": "Client",
      "privateIPAddress": "10.1.0.1"
    },
    {
      "name": "{agent_id_or_machine_name}",
      "role": "Server",
      "privateIPAddress": "10.1.0.2"
    }
  ]
}
```

The `name` must match the `--agentId` CLI flag (or machine hostname if not specified).
Roles are profile-specific — check the workload's documentation for valid role names.

## Example: 3-Node Layout (NGINX)
```json
{
  "clients": [
    { "name": "vm-0", "role": "Client",       "privateIPAddress": "10.1.0.1" },
    { "name": "vm-1", "role": "Server",       "privateIPAddress": "10.1.0.2" },
    { "name": "vm-2", "role": "ReverseProxy", "privateIPAddress": "10.1.0.3" }
  ]
}
```

## CLI Usage for Multi-System
```bash
# Client system
VirtualClient.exe --profile=PERF-NETWORK.json --system=Demo --timeout=1440 \
  --layoutPath=layout.json --agentId=vm-0 --packages="{SAS_URI}"

# Server system  
VirtualClient.exe --profile=PERF-NETWORK.json --system=Demo --timeout=1440 \
  --layoutPath=layout.json --agentId=vm-1 --packages="{SAS_URI}"
```

## REST API (inter-instance communication)
- Runs within each VC process on port **4500** (default)
- Used for: heartbeats, instructions/eventing, shared state
- **Requires elevated/privileged mode** (firewall rules must be modified)
- Custom port: `--api-port=4501`
- Role-specific ports: `--api-port=4501/Client,4502/Server`

### API Endpoints
| Endpoint | Purpose |
|----------|---------|
| Heartbeat | Check if other instance is online and running |
| Instructions/Eventing | Send/receive instructions between instances |
| State | Persist data for cross-instance coordination |

## Component Role Support
In the component class, declare supported roles:
```csharp
protected override void SupportedRoles { get; } = new List<string> { "Client", "Server" };
```

The component checks its role via:
```csharp
string role = this.Parameters.GetValue<string>("Role");
bool isClient = role.Equals("Client", StringComparison.OrdinalIgnoreCase);
```

## Client/Server Profiles (examples)
| Profile | Roles |
|---------|-------|
| `PERF-NETWORK-*.json` | Client, Server |
| `PERF-REDIS-*.json` | Client, Server |
| `PERF-MEMCACHED-*.json` | Client, Server |
| `PERF-SYSBENCH-*.json` | Client, Server |
| `EXAMPLE-CLIENT-SERVER-WORKLOAD-*.json` | Client, Server |

## Reference Implementation
- `VirtualClient.Actions/Examples/ClientServer/` — canonical client/server example
- `VirtualClient.Actions/Redis/` — production client/server workload
- `VirtualClient.Actions/Memcached/` — production client/server workload
