# MCP Server

Wendy ships a built-in [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server that lets AI assistants — Claude Code, Claude Desktop, Codex, and others — interact with your devices directly through tool calls.

## Starting the server

```bash
wendy mcp serve
```

The server speaks JSON-RPC over stdio. Pass `--device` to connect to a specific device on startup:

```bash
wendy mcp serve --device my-pi.local:50051
```

Without `--device`, the server starts unconnected and waits for a `device_connect` tool call.

## Setup

Run the setup command to automatically configure Claude Code and Claude Desktop:

```bash
wendy mcp setup
```

This writes the MCP server entry to `~/.claude.json` (Claude Code) and the Claude Desktop config file on your platform. Restart your AI tool after running it.

### Manual config (Claude Code)

Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "wendy": {
      "type": "stdio",
      "command": "wendy",
      "args": ["mcp", "serve"]
    }
  }
}
```

### Manual config (Claude Desktop)

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `~/.config/Claude/claude_desktop_config.json` (Linux):

```json
{
  "mcpServers": {
    "wendy": {
      "command": "wendy",
      "args": ["mcp", "serve"]
    }
  }
}
```

## Available tools

### Device

| Tool | Description |
|------|-------------|
| `device_list` | List devices from config |
| `device_connect` | Connect to a device by address |
| `device_disconnect` | Disconnect from the current device |
| `device_info` | Get agent version, OS, CPU, GPU, and feature set |
| `device_set_default` | Save an address as the default device |

### Containers

| Tool | Description |
|------|-------------|
| `container_list` | List all containers and their running state |
| `container_start` | Start a container and capture output (bounded) |
| `container_stop` | Stop a running container |
| `container_delete` | Delete a container, optionally with its image and volumes |
| `container_stats` | Get memory and storage usage for all containers |
| `container_attach` | Attach to a running container and capture a snapshot of output |

### Telemetry

All telemetry tools collect a bounded snapshot (default 10 OTLP batches) and return standard OTLP JSON. See [OpenTelemetry docs](../otel.md).

| Tool | Description |
|------|-------------|
| `telemetry_logs` | Stream logs — filter by app, service, severity |
| `telemetry_metrics` | Stream metrics — filter by app, service, metric name prefix |
| `telemetry_traces` | Stream traces — filter by app, service, span name prefix |

### WiFi

| Tool | Description |
|------|-------------|
| `wifi_list` | List nearby networks and signal strength |
| `wifi_connect` | Connect to a network by SSID and password |
| `wifi_status` | Get current connection status and SSID |
| `wifi_disconnect` | Disconnect from WiFi |
| `wifi_known_networks` | List saved/known networks |

### Bluetooth

| Tool | Description |
|------|-------------|
| `bluetooth_scan` | Scan for nearby peripherals (configurable timeout) |
| `bluetooth_connect` | Connect to a peripheral by address |
| `bluetooth_disconnect` | Disconnect from a peripheral |

### Hardware

| Tool | Description |
|------|-------------|
| `hardware_capabilities` | List GPUs, cameras, I2C buses, USB devices, etc. — filter by category |

### Provisioning

| Tool | Description |
|------|-------------|
| `provisioning_status` | Check if the device is provisioned with Wendy Cloud |
| `provisioning_start` | Enroll the device with a Wendy Cloud token and wait for completion |

### OS

| Tool | Description |
|------|-------------|
| `os_update` | Trigger an OS update and stream download/install progress |

### File sync

`filesync_sync` requires binary file transfer which is not supported over MCP. Use the CLI directly:

```bash
wendy run
```

## Connection flow

1. Claude calls `device_list` to see configured devices
2. Claude calls `device_connect` with an address
3. All other tools operate on the connected device
4. Call `device_disconnect` or start a new `device_connect` to switch devices

The server holds one active connection at a time. Connecting to a new device closes the previous connection.
